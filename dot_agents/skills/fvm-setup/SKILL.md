---
name: fvm-setup
description: >
  Pin a Flutter project to one SDK version with FVM and wire VS Code to it: write
  .fvmrc, create the .fvm/flutter_sdk symlink, gitignore .fvm/, point the Dart
  extension at the pin, align bare `flutter` calls in the repo's own files and CI
  to the pin, and verify the terminal, the IDE, and fvm all resolve the same
  version. Invoke on "set up fvm", "pin the flutter version", "fvm this project",
  "why does pubspec.lock keep changing between machines/editors". Takes the target
  version as argument; reads an existing .fvmrc or falls back to the active
  `flutter --version` when given none. Skip for installing FVM itself and for
  clearing old cached SDKs (`fvm remove` / `fvm cleanup`).
argument-hint: "Flutter version to pin, e.g. 3.44.6"
---

# FVM Setup

Pin one Flutter SDK per project and make every entry point agree. Without a pin, the
terminal (`fvm default`), the IDE's configured SDK, and a hardcoded path can each resolve a
different Flutter, and because `flutter_test` pins `meta` and `test_api` to exact versions
per SDK, `pubspec.lock` rewrites those lines on every resolve.

**Wrong skill if:** FVM is not installed (`fvm --version` fails): say so and stop.
Installing FVM is the user's call.

## 1. Decide the version

Use the argument if given. Otherwise read `.fvmrc`'s `flutter` key if the file exists, else
take the active `flutter --version` and confirm that version with the user before
continuing.

The pin must be an exact release (`3.44.6`): extract the version number, never a channel
name, a ref, or a whole `flutter --version` line. If the user insists on a channel, say the
CI half of step 7 will be skipped.

## 2. Install the version

Run `fvm list` and read its `Cache directory:` line: call it `<cache-dir>` and use it for
every SDK path below. If `<cache-dir>` ends in `/versions`, its parent is `<fvm-root>`, used
only for the `cache.git` paths in the recovery steps; if it does not, treat `<fvm-root>` as
unknown and never delete a `cache.git` path by guess.

If the target is absent from `fvm list`, run `fvm install <version>`.

`fvm install` and `fvm use` on FVM 4.3.x rebuild `<fvm-root>/cache.git` into a
heads/tags-only mirror one branch at a time, and start over from zero if interrupted.
Re-check on any FVM upgrade. If a command sits on `Creating local git cache...` or a
per-branch `git fetch` for more than a couple of minutes:

1. Kill that process by PID (`pgrep -f 'fvm install'` / `pgrep -f 'fvm use'`). Never
   `pkill -f fvm`: it reaches processes this skill did not start.
2. Delete `<fvm-root>/cache.git.lock` and `<fvm-root>/cache.git.heads-tags-tmp.*`. If
   `<fvm-root>` is unknown or `cache.git` is not under it, stop and report rather than
   deleting by guess.
3. Re-run `fvm list` and confirm the target version is listed and
   `<cache-dir>/<version>/bin/flutter` exists. If not, re-run `fvm install <version>` once
   and let it run to completion: with the lock and tmp dirs gone the rebuild is slow, not
   stuck, so the stall rule above does not apply to this re-run. If it stalls a second
   time, stop and report; do not kill and retry again.
4. Skip `fvm use` for the rest of this skill and create its outputs by hand (steps 3 and
   4). Those are all `fvm use` produces once the version is installed.

## 3. Write `.fvmrc`

Project root, exactly:

```json
{
  "flutter": "<version>",
  "flavors": {}
}
```

## 4. Create the `.fvm/flutter_sdk` symlink

Point it at the installed SDK:

```
mkdir -p .fvm
ln -sfn <cache-dir>/<version> .fvm/flutter_sdk
```

`fvm use <version>` does this too when its cache step is not stuck; either way the result
must be `.fvm/flutter_sdk` resolving to the target SDK directory.

## 5. Gitignore `.fvm/`

Add `.fvm/` to the project `.gitignore` if absent. `.fvmrc` is the shared pin and stays
tracked; `.fvm/` holds the machine-local symlink.

## 6. Wire VS Code

Merge into `.vscode/settings.json`, removing any existing `dart.flutterSdkPath` that points
at an absolute or version-specific path:

```json
{
  "dart.flutterSdkPath": ".fvm/flutter_sdk",
  "files.exclude": { "**/.fvm": true },
  "files.watcherExclude": { "**/.fvm": true }
}
```

`dart.flutterSdkPath` follows the symlink; `files.exclude` hides `.fvm/` from the Explorer
and search; `files.watcherExclude` keeps the watcher off the SDK copy.

## 7. Align the repo and CI to the pin

Grep for bare `flutter ` invocations in `CLAUDE.md`, `AGENTS.md`, `README*`, and `*.sh` in
the directory holding the `pubspec.yaml` you are pinning (the same directory as step 3's
`.fvmrc`): not its subdirectories, not any directory above it. Skip any already preceded by
`fvm` and prefix the rest with `fvm` (`fvm flutter test`). Before editing a `CLAUDE.md` or
`AGENTS.md` there, run `readlink -f` on it; if the result lands outside that directory,
report the file and leave it alone: those files have other consumers. Add only the `fvm`
prefix: no Flutter, Dart, or package version numbers, which churn and would go stale in a
durable instruction file. An entry point left on `PATH`'s Flutter rewrites `pubspec.lock` on
its next resolve, undoing the pin.

CI is different: the runner has no FVM. Check `.github/workflows/*.{yml,yaml}` for a job
that installs Flutter. Do not prefix its commands. Instead make the installer read the
version from `.fvmrc` at build time so it never needs manual sync: for
`subosito/flutter-action` v2, replace `flutter-version:` with `flutter-version-file: .fvmrc`
and keep `channel:` (`instruction_external_facts.md` §subosito/flutter-action). Apply this
only when `.fvmrc` holds an exact release and the job checks out the repo before installing
Flutter; otherwise report the workflow and leave it alone. Report, rather than edit, any workflow
that installs Flutter another way (manual download, container image, asdf, mise).

## 8. Verify

- `fvm flutter --version` reports `<version>`; `.fvm/flutter_sdk/bin/dart --version` reports
  the matching Dart version.
- Drift check, only when `git status --porcelain pubspec.lock` prints nothing and the file
  is tracked; otherwise skip this bullet and say the drift check did not run. Run
  `fvm flutter pub get`, then `git diff --stat pubspec.lock`. A non-empty diff on this first
  resolve after the pin changed is expected: review the `meta` / `test_api` lines and keep
  it. Run `fvm flutter pub get` again: a non-empty diff on the second, back-to-back resolve
  means another entry point is resolving a different SDK: find it.

## 9. Hand off

Tell the user to reload the VS Code window so the Dart extension picks up the new SDK, and
that the integrated terminal still uses `PATH`'s `flutter`: prefix commands with `fvm` or
prepend `.fvm/flutter_sdk/bin` to the terminal's `PATH`.

Commit `.fvmrc`, `.gitignore`, `.vscode/settings.json`, `pubspec.lock` when the resolve
rewrote it, and any repo file you changed in step 7, together, when the user asks.

Never `fvm remove` the pinned version or the version FVM's `default` link points at.
