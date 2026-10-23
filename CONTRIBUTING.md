# Quieting a Samsung Tizen TV on a model QE50Q60CAUXXH — a practical how-to

What we did on a Samsung QE50Q60CAUXXH (firmware T-NKLCDEUC-0090-2220.4) with
the quiet-tizen project, and what you need to repeat it on your own TV. This
is for someone who found the original case study
(https://github.com/jss8649/quiet-tizen — written for a *different* model,
UN75U8000FFXZA) and wants to try it on theirs. Everything here was executed
against a live device and verified by reading state back from it.

Result on our unit: advertising pipelines, ambient content recognition and
always-on voice wake disabled — 28 registry entries, 5 systemd stop-markers,
1 preference — while the TV stays a fully working HDMI display with
push-to-talk voice and the apps we actually use. Every change has an exact
inverse.

## 0. What you need

- The repo: `github.com/jss8649/quiet-tizen`. Read its AGENTS.md first — it
  tells you the cleanup profile is per-model and must not be reused blindly.
  Believe it: the code enforces it (a gate in `tools/tv.py` refuses any
  model/firmware other than the case-study device).
- Your exact TV data, from Settings → Support → About: the model code and the
  full S/W version string (ours: `QE50Q60CAUXXH`,
  `T-NKLCDEUC-0090-2220.4`). Type them literally — the gate compares exact
  strings. Reading from a photo? Zoom and re-read; OCR on firmware strings
  misreads characters.
- Developer Mode ON, SDB reachable on port 26101. No prior root and no
  retail hacks are needed: the chain uses the package-installer path plus two
  pre-existing file capabilities on the TV (`/usr/bin/dotnet` with
  cap_setgid+cap_sys_admin, `/usr/sbin/sdbd-tarlauncher` with
  cap_setuid+setgid) to reach root-context command execution. On retail
  firmware `sdb shell` prints `closed` — expected, not a failure.
- A workstation on the same LAN that the TV can dial back to (the payloads
  use a reverse callback). Fedora/openSUSE firewall:

      sudo firewall-cmd --zone=public \
        --add-rich-rule='rule family="ipv4" source address="TV_IP" accept' \
        --timeout=3600

  Use `--timeout` (self-cleaning) and re-check the rule before every long
  batch — an expired rule is the #1 cause of "TV did not connect to callback
  within 15s" and looks like a transport bug.
- .NET SDK 6 to build the payloads (`SamsungTvArchiveRoot`,
  `SamsungTvRootAgent`); the TV runtime is Microsoft.NETCore.App 6.0.9, so
  match it.

## 1. First contact and diagnostics

    QT_PROFILE=profiles/<your-model>.json python3 tools/tv.py --host TV_IP \
      --sdb <path-to-sdb> --firmware 'T-NKLCDEUC-0090-2220.4' doctor

The model/firmware gate will fire on a new model — that is the design, not a
bug. Handle it by creating a profile for your model (step 2), not by
bypassing the check. `doctor` verifies the reverse callback and the payload
runtime (capabilities dump, `dotnet --list-runtimes` on the TV).

## 2. Create a profile for YOUR model

Use the U8000F profile as a *template only*; re-derive everything from your
TV with read-only queries (docs/01-inventory.md flow, driven through
`tv.py exec --script`):

- Registry export: `/opt/dbspace/.pkgmgr_parser.db`, table
  `package_app_info` (`app_id,app_disable,app_onboot,app_autorestart`).
- Unit files: for every candidate service run
  `systemctl cat <unit> | grep ConditionPath` — the name inside
  `ConditionPathExists=!/opt/usr/share/systemd/system/<marker>` is the ONLY
  valid marker for that unit.

Differences we found on the Q60C; expect yours to differ too:

- Marker basenames are per-firmware. Case study: `org.tizen.pisa-service`;
  ours: `pisa-service`. One marker gates every unit sharing it
  (pisa-service + adagent-service).
- `/opt/usr/share/systemd/system/` did not exist yet — `mkdir -p` before the
  first marker.
- Voice wiring differs: `voice-app`/`voice-client` are `Type=oneshot` app
  preloaders (already disabled on our firmware), the wake-word engines have
  no systemd units at all — gated purely via the registry — and
  `ondevice-voice` is `.path`-activated. A copied profile would have been
  wrong in both directions.

One-line patch so the profile path comes from an env var (gate stays intact):

    PROFILE = os.environ.get("QT_PROFILE", "profiles/un75u8000ffxza-1300.9.json")

## 3. Payload adaptation — two things to fix locally

**a) The injection size cliff — measure it on YOUR TV.** The payload rides
the installer path as a shell argument; the repo caps it at
`SDB_APPINSTALL_MAX_SAFE_BYTES = 510`, which the U8000F tolerated. Our Q60C
breaks far earlier — found by padding a probe command: **359 bytes OK,
363 FAIL**; we set 340 and kept the measurement in a comment. Symptom of
hitting your cliff: short commands work, but batched `doctor`/`_check_target`
captures die with `closed` — easily misdiagnosed as a firewall problem.

**b) An upstream bug in `exec --script`.** The script is base64-encoded but
piped straight into `/bin/sh -s` without decoding, so every run fails with
rc 127. Fix in `tools/tv.py`:

    printf '%s' <b64> | base64 -d | /bin/sh -s --

After both fixes `doctor` is green, and `root-check` ends every privileged
run with: `UID/EUID 0 verified; root staging cleaned; passwd and SDB key
hashes unchanged.` Keep that line in view — it is your assurance the staging
left no trace in authentication files.

## 4. Snapshot before touching anything

    python3 tools/tv.py ... exec --script recipes/snapshot.sh --arg qe50-01

(No dots in the snapshot name — a character whitelist rejects them with a
silent exit 1.) This creates `/opt/usr/share/quiet-tizen-backups/qe50-01/`
on the TV: registry DB + a dump of vconftool preferences. Pull both to the
workstation — plans are generated from the local copy:

    python3 tools/tv.py ... pull /opt/usr/share/quiet-tizen-backups/qe50-01/.pkgmgr_parser.db .local/qe50-01-registry.db
    python3 tools/tv.py ... pull /opt/usr/share/quiet-tizen-backups/qe50-01/preferences-before.txt .local/

## 5. Generate plans, review the lists, choose what to disable

    for g in advertising voice context smart-extras; do
      python3 tools/plan_cleanup.py --snapshot .local/qe50-01-registry.db \
        --profile profiles/qe50q60cauxxh-2220.4.json --groups $g \
        --output .local/plans/qe50-$g-01
    done

Each plan contains before.json + apply.sql/apply.sh + restore.sql/restore.sh:
guarded `UPDATE`s that set `app_disable=true, app_onboot=false,
app_autorestart=false` for exact `app_id`s, plus the exact inverse.

Review the app_id lists per group before applying — the groups are a menu,
not an order. What we chose:

- **Advertising (8, applied)**: com.samsung.tv.acr, canalysis-service,
  targeted-ad-service, adagent-service, pisa-service,
  org.tizen.pisa-control-service, contents-recognition-dbmgr,
  com.samsung.tizen.samsung-analytics.
- **Context recognition (7, applied)**: com.samsung.tv.vif,
  content-grab-service, and the five context-aware* entries (agent, service,
  toast-view, grab-control, server-requester).
- **Voice (13, applied as a custom "voice-b" group)**: disabled both wake
  engines (bixby + alexa), bixby-prov, bixby-context-client,
  bixbycapsuleviewer, mybixby, alexa-client, alexa-fullscreen-app,
  utterance-recommender core+app, voice-instant-app, voiceassistant-music,
  voice-onboarding. Kept: voice-app, voice-client, voice-app-service,
  ondevice-voice, multi-assistant bus. Why this split is safe for
  push-to-talk: voice-app/voice-client are oneshot preloaders, nothing
  `Requires=`/`BindsTo=` them (only a soft `Wants=` from aov-service), and
  the STT/TTS servers are D-Bus activated by name. Voice *preferences*
  (`voice_activated` etc.) left alone on purpose — suppression happens at
  registry/unit level, so the TV menu stays your user-facing switch.
- **Smart-extras (11, deliberately NOT applied)**: health, SmartThings,
  The House, adplayer, AOV dashboard. Planned, reviewed, kept — fully
  reversible to apply later if ever wanted.

## 6. Apply, marker, verify — one group at a time

For each group: push its restore script to the TV backup dir *before*
applying, then apply, then place stop-markers, then verify.

    python3 tools/tv.py ... push .local/plans/qe50-advertising-01/restore.sh adv-restore.sh
    python3 tools/tv.py ... exec --command 'cp /home/owner/share/tmp/sdk_tools/quiet-tizen/adv-restore.sh /opt/usr/share/quiet-tizen-backups/qe50-01/ && chmod 700 /opt/usr/share/quiet-tizen-backups/qe50-01/adv-restore.sh'
    python3 tools/tv.py ... exec --script .local/plans/qe50-advertising-01/apply.sh

Stop-markers — signature `apply|restore UNIT MARKER SNAPSHOT_NAME`, ONE
unit/marker pair per call (no `unit:marker` syntax, snapshot name last):

    python3 tools/tv.py ... exec --script recipes/stop-marker.sh \
      --arg apply --arg pisa-service.service --arg pisa-service --arg qe50-01

Verify by reading state back, every time:

- Registry: `select app_id,app_disable,app_onboot,app_autorestart from
  package_app_info where app_id in (...);`
- Units: `systemctl is-active <units>`. After a marker stop, services may sit
  at `ActiveState=failed` — run `systemctl reset-failed`; the healthy end
  state is inactive with `ConditionResult=yes` (the marker is doing its job).
- Processes: grep `/proc/*/cmdline` for the disabled binaries.

One optional preference we applied (old value is in the preferences dump):

    vconftool set -f -t string db/was/preload_by_country false

Final verified state on our TV: 28 registry IDs disabled (8 + 7 + 13), 5
markers in `/opt/usr/share/systemd/system/` (pisa-service, canalysis-daemon,
context-aware-agent, context-aware-service, com.samsung.tv.bixby-prov), no
wake-engine or bixby processes in `ps`, smart-extras byte-identical to the
snapshot. Reboot the TV afterwards so all registry flags reload cleanly.

## 7. Rollback

Everything has an exact inverse: one `restore-registry-<group>.sh` per phase
(on the TV and locally), markers are single files (`stop-marker.sh restore`,
or plain `rm` + `reset-failed`), preferences from the dump, and the full
registry DB snapshot under `/opt/usr/share/quiet-tizen-backups/<name>/`.

## Pitfalls checklist (all of these bit us once)

1. Firmware string misread from a photo — zoom and re-read.
2. Firewall rich rule expired mid-batch — re-check before every batch.
3. Injection size cliff is per device — measure it, don't trust 510.
4. `exec --script` rc 127 — missing `base64 -d` in tv.py.
5. `stop-marker.sh` silent exit 1 — wrong arg order, dots in the snapshot
   name, or a marker basename that doesn't exist on YOUR firmware (derive it
   from `ConditionPathExists`, never copy it across profiles).
6. `mkdir -p /opt/usr/share/systemd/system` before the first marker.
7. Quoting through `exec --command` mangles SQL — prefer `exec --script`.
8. `pull` takes both positionals: `pull <remote> <local>`.
9. Services at `failed` after a condition-stop — `systemctl reset-failed`.
10. Distinguish systemd truth (units, conditions, activation type) from
    registry truth (disable/onboot/autorestart) — some components, like the
    wake engines here, exist only in the package registry.

---

Device work executed 2026-10-22/23 on QE50Q60CAUXXH, firmware
T-NKLCDEUC-0090-2220.4. Performed with Hermes Agent (Nous Research), model
qwen3.8-flash-next on Strata by Niko1221, IQ3_S quantization.
Thank you jss8649 and all people and agents involved. 
