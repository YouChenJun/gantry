# YouChenJun accelerated fork

This repository is a fork of [getgantry/gantry](https://github.com/getgantry/gantry).
The maintenance branch is **`maint/accelerated`**.

## What this branch contains

Selected upstream PRs that were still open, plus Simplified Chinese (`zh-Hans`):

| Upstream | What |
|----------|------|
| #17 | Only prompt apple/container setup when an Apple host is configured |
| #21 | Prevent empty container setup sheet on launch |
| #22 | Keep one main Gantry window |
| #24 | Badge foreign-architecture images |
| #26 | Track apple/container 1.6.1 (skips older #16/#18/#20/#23) |
| #13 | Dependabot CI action bumps |
| #27 / local | zh-Hans String Catalogs + AppCore alerts |

Not merged (no PR yet): upstream issues #15 and #25.

Skipped as superseded: #16, #18, #20, #23.

## Auto-updates (Sparkle)

`Info.plist` `SUFeedURL` points at:

`https://raw.githubusercontent.com/YouChenJun/gantry/maint/accelerated/appcast.xml`

Signed Sparkle releases rewrite that `appcast.xml` on `maint/accelerated`.

**Important:** `SUPublicEDKey` still ships upstream’s public key until you generate a
**fork-owned** Sparkle keypair. For Sparkle to verify fork builds you must:

1. Run Sparkle’s `generate_keys` / `generate_keys -x`
2. Put the private key in the repo secret `SPARKLE_PRIVATE_KEY`
3. Replace `SUPublicEDKey` in `Info.plist` with the matching public key
4. Cut a signed release

Do **not** copy upstream’s private key.

## Cutting a release

### Prerequisites (GitHub UI)

1. Enable **Actions** on `YouChenJun/gantry` if they are disabled for forks.
2. Optional secrets (Settings → Secrets and variables → Actions):
   - `SPARKLE_PRIVATE_KEY` — required for signed Sparkle / appcast updates
   - `RELEASE_PLEASE_TOKEN` — optional PAT so release PRs get CI (GITHUB_TOKEN works)
3. This fork does **not** bump `getgantry/homebrew-tap`. No `HOMEBREW_TAP_DEPLOY_KEY` needed.
4. Builds use **ad-hoc** code signing in CI (same as upstream’s workflow). There is no
   Apple Developer certificate requirement in the current workflow. Gatekeeper may still
   warn on download; users can open via right-click or `xattr -dr com.apple.quarantine`.

### Option A — unsigned GitHub Release (works without Sparkle secrets)

1. Ensure `MARKETING_VERSION` / `version.txt` match the version you will ship
   (`scripts/sync-version.sh 0.24.1`).
2. Actions → **Release** → Run workflow  
   - Branch: `maint/accelerated`  
   - `version`: e.g. `0.24.1`  
   - `unsigned`: **true**
3. When the job finishes, open the GitHub Release `v0.24.1` and download `Gantry-0.24.1.zip`.

### Option B — signed Sparkle release

1. Add `SPARKLE_PRIVATE_KEY` and update `SUPublicEDKey` (see above).
2. Prefer Conventional Commits on `maint/accelerated`, then let **Release Please**
   open a release PR; merge it to tag and dispatch the signed build.
3. Or Actions → **Release** → Run workflow with `unsigned: false`.

### Option C — tag push

```bash
git checkout maint/accelerated
git tag v0.24.1
git push origin v0.24.1
```

Tag pushes still check out `maint/accelerated` for the build. Without
`SPARKLE_PRIVATE_KEY` the signed path fails — use Option A instead.

## Syncing from upstream

```bash
git fetch upstream
git checkout maint/accelerated
git merge upstream/main   # resolve conflicts; keep zh-Hans catalogs + UX fixes
git push origin maint/accelerated
```

## Default branch

`main` remains a mirror of upstream releases. Day-to-day fork work happens on
`maint/accelerated`. Changing the GitHub default branch is optional.

## CI runner note

The Release workflow uses GitHub-hosted **`macos-26`** runners (Gantry’s
deployment target is macOS 26 / Xcode 26+). Free-plan queues for those runners
can sit in `queued` for a long time. If a run never starts, check
[Actions](https://github.com/YouChenJun/gantry/actions) and your plan’s macOS
minutes; do not switch the workflow to `macos-15` — that image lacks the macOS
26 SDK the project needs.

