# `dsh plugin add` installs the release before the one you just published

English | [中文](plugin-add-installs-the-previous-release.zh.md)

Verified on `@deepseek-ai/dsh` 0.2.2-rc.1 with pnpm 12.18.0 on 2026-11-16. We installed two plugins from npm into scratch `DSH_HOME`s and read the plugin manager's source at `dsh-v0.2.2-rc.1`.

## Symptom

You publish a fix. An hour later someone runs `dsh plugin add <your-package>` and gets the previous version, while npm shows the new one as `latest`. Nothing in the output says a newer release exists.

Measured on 2026-11-16 around 05:40 UTC:

| package | newest on npm (published) | `dsh plugin add <name>` installed |
|---|---|---|
| `dsh-context` | 0.85.1 (11-15 18:48 UTC, about 11 h earlier) | 0.84.1 |
| `dsh-media-desk` | 2.7.2 (11-15 17:12 UTC, about 12.5 h earlier) | 2.7.1 |

## Cause

dsh does not choose the version. `dsh plugin add` and the installer in the web UI both run `pnpm add` inside the profile, using the `pnpm` on your `PATH` (`packages/boot/plugin-manager/src/operations.ts:34`, `:369`). From pnpm 11, pnpm's `minimumReleaseAge` defaults to 24 hours, so an unpinned name resolves to the newest release older than a day.

dsh knows this happens. The plugin manager reports the version pnpm actually installed, and its type comment says `minimumReleaseAge` "can make it older than the newest release" (`plugin-manager/src/types.ts:176`). The desktop packaging README names "pnpm 11's default 24-hour `minimumReleaseAge`".

This is a supply-chain defence, not a dsh bug. A compromised release has a day to be noticed before it reaches anyone's profile.

## Fix

- **To install a release younger than a day, pin it.** `dsh plugin add dsh-context@0.85.1` installed 0.85.1, and pnpm recorded the exemption in the profile:

  ```
  Added 1 entry to minimumReleaseAgeExclude in pnpm-workspace.yaml (set minimumReleaseAgeStrict to true to gate these updates with a prompt):
    dsh-context@0.85.1
  ```

  The exemption covers that exact version only. The next unpinned update waits its day again.
- **Plugin authors:** put the pinned command in a hotfix's release note (`dsh plugin add <name>@<version>`), or expect users to get the fix a day late. Pin the version in your own install test too, or it tests the previous release.
- **Read what you got:** pnpm prints `+ <name> <version>` at the end of the install. That line is the installed version, not the one npm calls `latest`.

Not tested: a `pnpm` older than 11 on `PATH`. We also did not test switching the window off; doing so would remove the defence for every package in that profile.
