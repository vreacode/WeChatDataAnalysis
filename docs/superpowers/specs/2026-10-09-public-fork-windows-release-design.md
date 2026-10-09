# Public Fork and Windows Release Design

## Goal

Publish the current local WeChatDataAnalysis source changes as an attributed public GitHub fork and provide a verified Windows x64 installer in GitHub Release `v2.1.5-personal.1`.

## Publication model

- Create a public GitHub fork of `LifeArchiveProject/WeChatDataAnalysis` under the GitHub account authorized by the user through GitHub's browser/device authentication flow.
- Preserve the upstream Git history and attribution. Do not present the fork as an independently authored project.
- Keep `upstream` pointing to `LifeArchiveProject/WeChatDataAnalysis` and use the user's fork as `origin` after the fork has been created.
- Publish the current group-chat remark refresh fix and its regression tests as a focused commit.
- Tag the verified release commit as `v2.1.5-personal.1`.

## Installer design

- Build the existing Electron NSIS Windows x64 target from `desktop/package.json` using the repository's documented `npm run dist:win` pipeline.
- Retain the upstream application identity and product name so existing runtime assumptions remain intact.
- Accept the project's current self-signed/private-PKI signing flow. The Release notes must warn that Windows SmartScreen or publisher prompts may appear.
- Do not invent a second packaging system or reconfigure GitHub Actions for this first personal release.

## Privacy and secret boundary

The publication must exclude all user data and machine-specific state, including:

- `D:\weixindaochu\.wda-data`
- `D:\weixindaochu\导出目录`
- local WeChat databases, decrypted databases, media, exports, and account identifiers
- encryption keys, key caches, native-core license caches, credentials, tokens, cookies, and `.env` files
- logs, temporary build files, virtual environments, `node_modules`, and local absolute-path configuration
- the GitHub password disclosed in chat; it must never be written to disk, Git configuration, a command line, a commit, or a Release

Before pushing, inspect tracked files, untracked files, the staged diff, and the commits being published. Before uploading the installer, inspect its packaged file list and run the repository's Windows smoke checks.

## Verification flow

1. Run the focused Python regression tests for the current chat-session change.
2. Run the broader relevant test suite required by the repository's package scripts.
3. Install or reuse only project-local dependencies according to lock files.
4. Build the Windows x64 NSIS installer with `npm run dist:win`.
5. Run `npm run smoke:win` and any package-integrity tests applicable to the produced artifact.
6. Confirm the `.exe` exists, is non-empty, has the expected version, and contains no obvious user paths, account data, databases, logs, or credentials.
7. Record the SHA-256 digest and file size in the Release notes.
8. Create the public fork, push the reviewed commits and tag, create the GitHub Release, upload the verified `.exe`, and verify the public Release URL and downloadable asset.

## Authentication and failure handling

- GitHub authentication must use GitHub CLI browser/device authorization or an already valid credential. Account-password Git authentication is forbidden.
- If GitHub CLI is unavailable, install it from its official distribution or use GitHub's browser UI; do not embed credentials in remote URLs.
- If the build fails, preserve successful setup work, capture the complete error, and apply the smallest repository-consistent fix.
- If native artifacts or signing inputs required by the upstream build are unavailable for redistribution, stop before publishing a partial or misleading installer and report the exact blocker.

## Release contents

The Release will contain:

- tag and title `v2.1.5-personal.1`
- the verified Windows x64 `Setup.exe`
- concise attribution to the upstream project
- a summary of the group-chat remark refresh fix
- supported-platform and self-signed installer warnings
- SHA-256 and file size

No chat records, test exports, source archives manually assembled from the working tree, or unrelated local launcher files will be attached.

## Success criteria

- The fork is public and visibly linked to the upstream repository.
- The current local fix and tests are committed without unrelated or sensitive files.
- The Windows x64 installer builds and passes the available smoke/integrity checks.
- The installer is downloadable from GitHub Release `v2.1.5-personal.1`.
- The published asset hash matches the locally verified installer.
