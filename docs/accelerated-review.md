# Code review: `maint/accelerated` vs upstream `main`

Reviewed tip: local `maint/accelerated` after review fixes (see git log).
Baseline: `upstream/main` @ `82d7250` (Release 0.24.0).
Release verified: https://github.com/YouChenJun/gantry/releases/tag/v0.24.1

Scope: merged upstream PRs #17, #21, #22, #24, #26, #13 + zh-Hans + fork Actions.

## Summary

No conflict markers. Localizable catalogs parse as JSON; `%@` placeholder counts
match between keys and zh-Hans values. Force-unwrap / `try!` / `as!` in the
delta appear only in unit tests (`ModelTests.swift`), not production paths.

Several logic/i18n issues were found; clearly safe ones were fixed on this
branch (see “Fixed in this review”).

## Findings

| Sev | File:line | Issue | Fixed |
|-----|-----------|-------|-------|
| High | `Info.plist` `SUPublicEDKey` / Sparkle | `SUFeedURL` points at the fork appcast, but `SUPublicEDKey` is still upstream’s public key. A *signed* fork appcast will fail Sparkle verification until a fork keypair is generated (`SPARKLE_PRIVATE_KEY` + plist public key). Unsigned `v0.24.1` zip is unaffected. | N (needs real keypair; do not invent) |
| Med | `Packages/DockerKit/.../Endpoints/Images.swift` `listImages` | `#24` always queried `manifests=true`. Daemons that reject the query would fail the entire image refresh. | **Y** — fall back to plain `/images/json` |
| Med | `App/Views/ContentView.swift` compose/dockerfile `onReceive` | Notification handlers called `Pending*Opens.drain()`, discarding *all* buffered URLs when one live notification arrived (multi-file open race). | **Y** — `remove(url)` only |
| Med | `Packages/AppCore/.../ContainerAlert.swift` ~55 | `exited(code:)` interpolated an `Int` into `String(localized:)`. Catalog key uses `%@`; runtime key may be `%lld`, so zh-Hans body may not apply. | **Y** — `String(code)` |
| Med | `Packages/DockerKit/.../Endpoints/Images.swift` `fillMissingPlatforms` | One inspect per image missing platform — slow/noisy on hosts with hundreds of images; failures are silent (no badge). | N (by design of #24; monitor) |
| Med | `App/ComposeOpen.swift` `MainWindowActivation` ~10–19 | Hard-codes window id prefix `main-AppWindow-` then `NSWorkspace.openApplication`. Fragile if SwiftUI renames windows; heavier than `openWindow`. | N |
| Med | `.github/workflows/release.yml` | Always checks out `maint/accelerated` even for tags from other branches — intentional for the fork, surprising otherwise. | N (documented in FORK.md) |
| Low | `App/Views/Containers/ContainerListView.swift` ~202 | Arch badge looks up `archByImageID[container.imageID]`. If daemon ImageID/Id spellings diverge, badge is a false negative (no crash). | N |
| Low | `SystemInfo.architecturesEquivalent` | Only aliases amd64↔x86_64 and arm64↔aarch64; other spellings rely on exact match. Case-folding already applied. | N |
| Low | `App/Views/Shared/ArchBadge.swift` help / catalogs | Help string localized; catalog may carry both `%@` and source-style keys until Xcode re-extracts. | N (harmless) |
| Low | `ContentView.containerToolingStatus` DEBUG | Any `--gantry-test-*` arg forces `.notInstalled` — intentional for UI tests. | N |
| Low | Confirmation / host-removal dialogs | Some dialog titles still English or only partially localized (e.g. “Remove this host?”). | N |
| Low | `Packages/AppCore/Tests/.../ModelTests.swift` 75,176–177 | `try!` / force unwrap — tests only. | N |
| Info | `#17` + `#21` gating | Apple-host guard runs after DEBUG test hooks; production path requires an Apple Container host before launch prompt. Manual `gantryShowContainerSetup` still works without that guard. | OK |
| Info | `#22` pending jump | `PendingContainerJump` + notification + `onAppear` take/clear is idempotent. | OK |

## Fixed in this review

1. `listImageSummaries()` manifests fallback  
2. `PendingComposeOpens.remove` / `PendingDockerfileOpens.remove` + ContentView handlers  
3. `ContainerAlert` exit-code interpolation uses `String(code)`  
4. Removed untracked junk `/workspace/gantry/release.yml` (not `.github/workflows/release.yml`)

## Remaining risks

- Sparkle auto-update for the fork is **not** trustworthy until a fork EdDSA keypair replaces upstream’s `SUPublicEDKey` and `SPARKLE_PRIVATE_KEY` is set.
- Large image libraries pay N inspects on refresh (#24).
- macOS 26 GitHub runners can queue for a long time on free plans (FORK.md).

## Method notes

- `git log upstream/main..HEAD` / diff name-only  
- `rg` for `try!`, `as!`, `fatalError`, conflict markers in changed Swift  
- JSON parse + placeholder count check on all `*.xcstrings`  
- Manual read of ContentView setup sheet, GantryApp window id, ArchBadge, Images endpoint, ContainerAlert, ComposeOpen activation  
