# Public Fork and Windows Release Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish the reviewed local chat-session fix in an attributed public fork and attach a verified, personally self-signed Windows x64 installer to GitHub Release `v2.1.5-personal.1`.

**Architecture:** Preserve the upstream repository and history, keep user data outside Git, and isolate personal signing from the upstream protected signing identity. Validate the source change first, rebase onto current upstream, build with the existing Nuxt/PyInstaller/Electron pipeline plus a generated local signing override, then publish only the reviewed commits and verified installer.

**Tech Stack:** Git, GitHub CLI, Python 3.11 with uv, pytest, Node.js/npm, Nuxt, Electron 40, electron-builder 26, NSIS, Windows certificate store, PowerShell.

---

### Task 1: Audit the publication boundary

**Files:**
- Inspect: `.gitignore`
- Inspect: `src/wechat_decrypt_tool/routers/chat.py`
- Inspect: `tests/test_chat_realtime_change_scan.py`
- Inspect: `tests/test_chat_source_auto.py`
- Protect: `D:\weixindaochu\.wda-data`
- Protect: `D:\weixindaochu\导出目录`

- [ ] **Step 1: Record the exact working tree and remotes**

Run:

```powershell
Set-Location D:\weixindaochu\WeChatDataAnalysis
git status --short
git remote -v
git log --oneline --decorate -5
```

Expected: only the three known chat fix files are modified; the approved design and plan commits are present; `origin` still identifies the upstream repository at this stage.

- [ ] **Step 2: Prove private data directories are outside the repository**

Run:

```powershell
$repo = (Resolve-Path D:\weixindaochu\WeChatDataAnalysis).Path
$private = @(
  (Resolve-Path D:\weixindaochu\.wda-data).Path,
  (Resolve-Path D:\weixindaochu\导出目录).Path
)
$private | ForEach-Object {
  if ($_.StartsWith($repo + '\', [StringComparison]::OrdinalIgnoreCase)) {
    throw "Private path is inside repository: $_"
  }
  "OUTSIDE_REPO $_"
}
```

Expected: two `OUTSIDE_REPO` lines and no exception.

- [ ] **Step 3: Review every pending source line**

Run:

```powershell
git diff --check
git diff -- src/wechat_decrypt_tool/routers/chat.py tests/test_chat_realtime_change_scan.py tests/test_chat_source_auto.py
```

Expected: no whitespace error; the diff only adds contact-database change detection and live display-name refresh with regression coverage.

- [ ] **Step 4: Scan tracked and untracked publication candidates for sensitive file types**

Run:

```powershell
git ls-files | rg -i '(^|/)(wechat_db_config\.json|\.env($|\.)|.*\.db(-wal|-shm)?$|.*\.sqlite3?$|.*\.log$|.*\.pfx$|.*\.p12$|.*\.key$)'
git status --short --untracked-files=all
```

Expected: no tracked database, environment, log, private-key, or credential file; untracked output contains only the plan directory if it has not yet been committed.

### Task 2: Verify and commit the chat-session fix

**Files:**
- Modify: `src/wechat_decrypt_tool/routers/chat.py`
- Modify: `tests/test_chat_realtime_change_scan.py`
- Modify: `tests/test_chat_source_auto.py`

- [ ] **Step 1: Run the focused regression tests**

Run:

```powershell
$env:PATH = 'D:\weixindaochu\.tools\uv-bootstrap\Scripts;' + $env:PATH
uv run --no-sync python -m pytest -q tests/test_chat_realtime_change_scan.py tests/test_chat_source_auto.py
```

Expected: all tests in both files pass with exit code `0`.

- [ ] **Step 2: Run adjacent chat-session tests**

Run:

```powershell
uv run --no-sync python -m pytest -q tests/test_chat_list_messages_re_scope.py tests/test_chat_realtime_name2id_sync.py tests/test_chat_realtime_message_content_text.py
```

Expected: all adjacent tests pass with exit code `0`.

- [ ] **Step 3: Commit only the fix and tests**

Run:

```powershell
git add -- src/wechat_decrypt_tool/routers/chat.py tests/test_chat_realtime_change_scan.py tests/test_chat_source_auto.py
git diff --cached --check
git diff --cached --stat
git commit -m "fix: refresh live group chat remarks"
```

Expected: one focused commit containing exactly three files.

### Task 3: Rebase the personal commits onto current upstream

**Files:**
- Inspect: Git history only

- [ ] **Step 1: Preserve the upstream remote name**

Run:

```powershell
if ((git remote) -contains 'origin' -and (git remote get-url origin) -eq 'https://github.com/LifeArchiveProject/WeChatDataAnalysis.git') {
  git remote rename origin upstream
}
git remote -v
```

Expected: `upstream` fetch and push URLs point to `LifeArchiveProject/WeChatDataAnalysis`.

- [ ] **Step 2: Fetch the current upstream history and tags**

Run:

```powershell
git fetch --unshallow upstream
git fetch upstream --tags --prune
git log --oneline --decorate --graph --max-count=12 HEAD upstream/main
```

Expected: the current `upstream/main` commit is visible with full ancestry.

- [ ] **Step 3: Rebase the personal commits**

Run:

```powershell
git rebase upstream/main
```

Expected: rebase succeeds and retains the design, plan, and chat-fix commits. If upstream already contains an equivalent fix, inspect the conflict and drop only the redundant fix commit; never resolve by overwriting new upstream behavior wholesale.

- [ ] **Step 4: Repeat the focused tests after rebase**

Run:

```powershell
uv run --no-sync python -m pytest -q tests/test_chat_realtime_change_scan.py tests/test_chat_source_auto.py
git status --short
```

Expected: tests pass and the working tree is clean.

### Task 4: Set the personal release version and notes

**Files:**
- Modify: `pyproject.toml`
- Modify: `uv.lock`
- Modify: `desktop/package.json`
- Modify: `desktop/package-lock.json`
- Create: `docs/releases/v2.1.5-personal.1.md`

- [ ] **Step 1: Update Python project metadata**

Change the project version in `pyproject.toml` from:

```toml
version = "2.1.5"
```

to:

```toml
version = "2.1.5-personal.1"
```

Then run:

```powershell
uv lock
```

Expected: `pyproject.toml` and the root package entry in `uv.lock` report `2.1.5-personal.1` without unrelated dependency upgrades.

- [ ] **Step 2: Update Electron package metadata without creating a tag**

Run:

```powershell
npm --prefix desktop version 2.1.5-personal.1 --no-git-tag-version
```

Expected: only the root versions in `desktop/package.json` and `desktop/package-lock.json` change to `2.1.5-personal.1`.

- [ ] **Step 3: Add public Release notes**

Create `docs/releases/v2.1.5-personal.1.md` with this content:

```markdown
# WeChatDataAnalysis v2.1.5-personal.1

Personal Windows x64 build forked from [LifeArchiveProject/WeChatDataAnalysis](https://github.com/LifeArchiveProject/WeChatDataAnalysis).

## Change

- Refresh group-chat display names when local WeChat contact data changes, so newly edited group remarks appear in realtime chat sessions.

## Windows package

- Platform: Windows x64
- Package type: NSIS Setup.exe
- Signing: personal self-signed certificate; this is not an upstream official signature
- Windows SmartScreen or an unknown-publisher warning may appear on first installation

The installer contains the application runtime only. It does not contain the maintainer's chat records, WeChat databases, account data, exports, or decryption keys.
```

- [ ] **Step 4: Validate and commit the version change**

Run:

```powershell
node -e "const p=require('./desktop/package.json'); const l=require('./desktop/package-lock.json'); if(p.version!=='2.1.5-personal.1'||l.version!=='2.1.5-personal.1'||l.packages[''].version!=='2.1.5-personal.1') process.exit(1)"
rg -n '^version = "2\.1\.5-personal\.1"$' pyproject.toml uv.lock
git diff --check
git add -- pyproject.toml uv.lock desktop/package.json desktop/package-lock.json docs/releases/v2.1.5-personal.1.md
git commit -m "chore: prepare v2.1.5-personal.1"
```

Expected: metadata and notes are committed without dependency-version churn.

### Task 5: Validate dependencies and package configuration

**Files:**
- Verify: `desktop/package.json`
- Verify: `desktop/package-lock.json`
- Verify: `frontend/package-lock.json`
- Verify: `desktop/tests/package-config.test.cjs`
- Verify: `desktop/tests/installer-safety.test.cjs`

- [ ] **Step 1: Synchronize locked Python build dependencies**

Run:

```powershell
uv sync --no-editable --extra build --extra voice-transcription
uv run --no-sync python -c "import fastapi, PyInstaller; print('python build imports ok')"
```

Expected: synchronization succeeds and prints `python build imports ok`.

- [ ] **Step 2: Verify local npm dependencies against lock files**

Run:

```powershell
npm --prefix frontend ci
npm --prefix desktop ci
```

Expected: both commands complete with exit code `0`; dependencies remain local to their projects.

- [ ] **Step 3: Run desktop package-safety tests before building**

Run:

```powershell
node --test desktop/tests/package-config.test.cjs desktop/tests/installer-safety.test.cjs desktop/tests/native-core-before-pack.test.cjs desktop/tests/windows-source-native-core-bootstrap.test.cjs
```

Expected: all selected Node tests pass.

### Task 6: Build a separately self-signed Windows installer

**Files:**
- Generate locally, ignored by Git: `desktop/build/personal-release/electron-builder.json`
- Generate locally in Windows certificate store: `CN=WeChatDataAnalysis Personal Build`
- Generate: `desktop/dist/WeChatDataAnalysis-2.1.5-personal.1-Setup.exe`

- [ ] **Step 1: Refresh the public native runtime cache**

Run the source-public bootstrap through the repository's normal build preparation:

```powershell
$env:WCE_NATIVE_CORE_SOURCE_PROFILE = 'source-public'
$env:WCE_NATIVE_CORE_SOURCE_RUNTIME_ROOT = 'D:\weixindaochu\.tools\source-native-core'
npm --prefix desktop run build:ui
npm --prefix desktop run build:backend
npm --prefix desktop run build:icon
```

Expected: the build selects or downloads a non-expired verified public native-core runtime, builds the backend, and stages the UI and icons without traceback or integrity failure.

- [ ] **Step 2: Create or reuse the personal signing identity**

Run:

```powershell
$subject = 'CN=WeChatDataAnalysis Personal Build'
$cert = Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.Subject -eq $subject -and $_.HasPrivateKey -and $_.NotAfter -gt (Get-Date).AddDays(90) } |
  Sort-Object NotAfter -Descending |
  Select-Object -First 1
if (-not $cert) {
  $cert = New-SelfSignedCertificate -Type CodeSigningCert -Subject $subject -CertStoreLocation Cert:\CurrentUser\My -KeyExportPolicy NonExportable -KeyLength 3072 -HashAlgorithm SHA256 -NotAfter (Get-Date).AddYears(2)
}
$env:WDA_PERSONAL_CERT_THUMBPRINT = $cert.Thumbprint
$cert | Select-Object Subject,Thumbprint,NotAfter,HasPrivateKey
```

Expected: one non-expired code-signing certificate with a private key is shown. No PFX or private key file is created.

- [ ] **Step 3: Generate an ignored personal electron-builder config**

Run from `desktop`:

```powershell
Set-Location D:\weixindaochu\WeChatDataAnalysis\desktop
node -e "const fs=require('fs'),path=require('path'),p=require('./package.json'); const c=structuredClone(p.build); delete c.win.signtoolOptions; c.win.forceCodeSigning=true; c.win.certificateSha1=process.env.WDA_PERSONAL_CERT_THUMBPRINT; const d=path.join('build','personal-release'); fs.mkdirSync(d,{recursive:true}); fs.writeFileSync(path.join(d,'electron-builder.json'),JSON.stringify(c,null,2)+'\n')"
```

Expected: generated config preserves packaging resources but removes the unavailable upstream signing hook and selects only the personal certificate thumbprint. `git status --short` remains clean because `desktop/build` is ignored.

- [ ] **Step 4: Build the NSIS installer**

Run:

```powershell
npx electron-builder --config build/personal-release/electron-builder.json --win nsis --x64 --publish never
```

Expected: exit code `0` and `desktop/dist/WeChatDataAnalysis-2.1.5-personal.1-Setup.exe` exists with a size greater than zero.

- [ ] **Step 5: Verify the signature identity**

Run:

```powershell
$installer = Resolve-Path .\dist\WeChatDataAnalysis-2.1.5-personal.1-Setup.exe
$sig = Get-AuthenticodeSignature -LiteralPath $installer
$sig | Select-Object Status,StatusMessage,@{n='Subject';e={$_.SignerCertificate.Subject}},@{n='Thumbprint';e={$_.SignerCertificate.Thumbprint}}
if ($sig.SignerCertificate.Subject -ne 'CN=WeChatDataAnalysis Personal Build') { throw 'Unexpected installer signer' }
```

Expected: signer subject is exactly `CN=WeChatDataAnalysis Personal Build`. Trust status may be `UnknownError` or `NotTrusted` on a machine that does not trust the personal certificate; `NotSigned` is not acceptable.

### Task 7: Smoke-test and audit the installer

**Files:**
- Verify: `desktop/dist/WeChatDataAnalysis-2.1.5-personal.1-Setup.exe`
- Verify: `desktop/dist/win-unpacked/**`

- [ ] **Step 1: Run the repository's Windows package smoke test**

Run from `desktop`:

```powershell
$env:WCE_WINDOWS_INSTALLER_SMOKE_ALLOWED = '1'
npm run smoke:win
```

Expected: packaged backend health check and Electron launch smoke succeed. If the script explicitly requires the upstream private-PKI signer, run its non-signature runtime checks against `dist/win-unpacked` and record the skipped upstream-signature assertion in Release notes; do not weaken native-core integrity checks.

- [ ] **Step 2: Scan packaged output for local data and paths**

Run:

```powershell
$targets = @('.\dist\win-unpacked', '.\dist\WeChatDataAnalysis-2.1.5-personal.1-Setup.exe')
$patterns = 'D:\\weixindaochu|C:\\Users\\vrea|wxid_[0-9A-Za-z]|wechat_db_config\.json|\.wda-data|导出目录'
rg -a -n -i --hidden --glob '!*.map' $patterns $targets
```

Expected: no match. Any match must be inspected and removed at its source before publication.

- [ ] **Step 3: Compute immutable release evidence**

Run:

```powershell
$installer = Get-Item .\dist\WeChatDataAnalysis-2.1.5-personal.1-Setup.exe
$hash = Get-FileHash -LiteralPath $installer.FullName -Algorithm SHA256
[pscustomobject]@{Path=$installer.FullName;Bytes=$installer.Length;SHA256=$hash.Hash} | Format-List
```

Expected: a non-zero file size and a 64-character SHA-256 hash.

### Task 8: Authenticate safely and create the public fork

**Files:**
- Modify locally: Git remotes and GitHub credential storage

- [ ] **Step 1: Install GitHub CLI from the official winget package if absent**

Run:

```powershell
if (-not (Get-Command gh -ErrorAction SilentlyContinue)) {
  winget install --id GitHub.cli --exact --source winget --accept-source-agreements --accept-package-agreements
}
gh --version
```

Expected: GitHub CLI prints its version. No GitHub password is used.

- [ ] **Step 2: Authorize through GitHub's browser/device flow**

Run:

```powershell
gh auth login --hostname github.com --git-protocol https --web
gh auth status
$githubLogin = gh api user --jq .login
"Authenticated GitHub login: $githubLogin"
```

Expected: the user completes GitHub authorization in the browser and `gh auth status` reports an authenticated account.

- [ ] **Step 3: Create or reuse the public fork and configure remotes**

Run:

```powershell
gh repo fork LifeArchiveProject/WeChatDataAnalysis --clone=false
$githubLogin = gh api user --jq .login
$forkUrl = "https://github.com/$githubLogin/WeChatDataAnalysis.git"
if (-not ((git remote) -contains 'origin')) { git remote add origin $forkUrl }
git remote set-url origin $forkUrl
gh repo view "$githubLogin/WeChatDataAnalysis" --json nameWithOwner,isFork,isPrivate,parent
git remote -v
```

Expected: `isFork` is `true`, `isPrivate` is `false`, parent is `LifeArchiveProject/WeChatDataAnalysis`, `origin` is the user's fork, and `upstream` is the original project.

### Task 9: Push and publish the verified Release

**Files:**
- Publish: reviewed Git commits
- Publish: tag `v2.1.5-personal.1`
- Upload: `desktop/dist/WeChatDataAnalysis-2.1.5-personal.1-Setup.exe`

- [ ] **Step 1: Perform the final pre-push review**

Run:

```powershell
git status --short
git log --oneline upstream/main..HEAD
git diff --check upstream/main..HEAD
git diff --stat upstream/main..HEAD
git ls-files | rg -i '(^|/)(.*\.db(-wal|-shm)?$|.*\.sqlite3?$|.*\.pfx$|.*\.p12$|.*\.key$|\.env($|\.))'
```

Expected: clean working tree; only reviewed personal commits; no sensitive tracked files.

- [ ] **Step 2: Push the main branch**

Run:

```powershell
git push --set-upstream origin main
```

Expected: push succeeds without force and GitHub shows the fork ahead of upstream by the reviewed commits.

- [ ] **Step 3: Create and push the annotated tag**

Run:

```powershell
git tag -a v2.1.5-personal.1 -m "WeChatDataAnalysis v2.1.5-personal.1"
git push origin v2.1.5-personal.1
```

Expected: the tag resolves to the exact locally built and tested commit.

- [ ] **Step 4: Create the GitHub Release and upload the installer**

Run from the repository root:

```powershell
$installer = Get-Item desktop\dist\WeChatDataAnalysis-2.1.5-personal.1-Setup.exe
$sha256 = (Get-FileHash -LiteralPath $installer.FullName -Algorithm SHA256).Hash
$releaseNotesPath = Join-Path $env:TEMP 'wda-v2.1.5-personal.1-release-notes.md'
$baseNotes = Get-Content docs\releases\v2.1.5-personal.1.md -Raw
$verification = "`n## Verification`n`n- File: ``$($installer.Name)```n- Size: $($installer.Length) bytes`n- SHA-256: ``$sha256```n"
Set-Content -LiteralPath $releaseNotesPath -Value ($baseNotes.TrimEnd() + "`n" + $verification) -Encoding utf8
gh release create v2.1.5-personal.1 $installer.FullName --repo "$githubLogin/WeChatDataAnalysis" --title "v2.1.5-personal.1" --notes-file $releaseNotesPath --verify-tag
```

Expected: GitHub returns the public Release URL, and its notes contain the exact locally measured size and SHA-256 digest.

- [ ] **Step 5: Verify the remote asset and hash**

Run:

```powershell
$verifyDir = Join-Path $env:TEMP 'wda-release-verify-v2.1.5-personal.1'
New-Item -ItemType Directory -Force -Path $verifyDir | Out-Null
gh release view v2.1.5-personal.1 --repo "$githubLogin/WeChatDataAnalysis" --json url,isDraft,isPrerelease,assets
gh release download v2.1.5-personal.1 --repo "$githubLogin/WeChatDataAnalysis" --pattern 'WeChatDataAnalysis-2.1.5-personal.1-Setup.exe' --dir $verifyDir --clobber
$localHash = (Get-FileHash desktop\dist\WeChatDataAnalysis-2.1.5-personal.1-Setup.exe -Algorithm SHA256).Hash
$remoteHash = (Get-FileHash (Join-Path $verifyDir 'WeChatDataAnalysis-2.1.5-personal.1-Setup.exe') -Algorithm SHA256).Hash
if ($localHash -cne $remoteHash) { throw 'Downloaded Release asset hash mismatch' }
"RELEASE_HASH_VERIFIED $remoteHash"
```

Expected: Release is neither draft nor prerelease unless explicitly requested, exactly one installer asset is listed, and `RELEASE_HASH_VERIFIED` prints the committed digest.
