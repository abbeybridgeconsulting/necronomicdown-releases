# How to publish a Necronomicdown beta

Version 0.1.0 is stored in `downloads/v0.1.0` in Git. Future DMGs should be uploaded as GitHub Release assets. The website can read `latest.json` from the default branch for current download details.

This is the checklist for the **ad hoc signed, non-notarised Apple Silicon build**. It does not require an Apple Developer membership. The app requires macOS 15 or later; Intel Macs are not supported by this build. The process is manual: there is no release automation yet.

- App source: `~/Documents/abbeybridgeconsulting/repos/markdown-reader`
- Release repository: `~/Documents/abbeybridgeconsulting/repos/necronomicdown-releases`
- GitHub releases: https://github.com/abbeybridgeconsulting/necronomicdown-releases/releases

## 1. Choose and update the version

From the app source repository, choose a new version (for example `0.1.1`; use the same number throughout). Never reuse a published version for different bytes.

```sh
cd ~/Documents/abbeybridgeconsulting/repos/markdown-reader
git status --short
npm version 0.1.1 --no-git-tag-version
```

Change the `version` fields in `src-tauri/tauri.conf.json` and `src-tauri/Cargo.toml` to the same version. `npm version` updates `package.json` and `package-lock.json`; the next Cargo build updates `Cargo.lock` if needed. Review the source and version changes now, then commit them including any Cargo.lock update after the build and before publishing, so the release can be traced to a source commit. Review any existing uncommitted changes: they will be included in the build.

## 2. Build and verify

On an Apple Silicon Mac with Xcode command-line tools, Rust, Node and the project's dependencies installed:

```sh
cd ~/Documents/abbeybridgeconsulting/repos/markdown-reader
npm ci
unset RELEASE_SIGNING_IDENTITY
OPEN_DMG=0 zsh scripts/build-local.sh
```

The build signs the app and Quick Look extension ad hoc. It creates `target/aarch64-apple-darwin/release/bundle/dmg/Necronomicdown_VERSION_aarch64.dmg`. If the build fails, stop; do not use an older DMG that may still be present at that path. The disk-image step needs macOS disk-image services and may fail inside a restricted sandbox; run the build in a normal Terminal session if needed.

Replace `0.1.1` below with the version you built:

```sh
version=0.1.1
app="target/aarch64-apple-darwin/release/bundle/macos/Necronomicdown.app"
cd target/aarch64-apple-darwin/release/bundle/dmg
# Run codesign from the source root, or use the absolute app path below.
codesign --verify --deep --strict --verbose=2 "$HOME/Documents/abbeybridgeconsulting/repos/markdown-reader/$app"
hdiutil verify "Necronomicdown_${version}_aarch64.dmg"
shasum -a 256 "Necronomicdown_${version}_aarch64.dmg" > "Necronomicdown_${version}_aarch64.dmg.sha256"
cat "Necronomicdown_${version}_aarch64.dmg.sha256"
```

Test the **final DMG** on another Mac if possible: mount it, drag the app to Applications, open it, and exercise the reader and Quick Look extension. As the app is not notarised, macOS may block the first launch. After trying to open it, use System Settings → Privacy & Security → Open Anyway if you trust the file. Check the downloaded copy again after publication.

## 3. Scan and publish

1. Submit the final DMG to VirusTotal and wait for its analysis. Uploading shares the file with VirusTotal and its partners. Record the actual report URL and result; a hash-based URL alone does not prove a scan happened.
2. In the GitHub release repository, create a new release with a version tag such as `v0.1.1`, targeting its `main` branch. Mark beta releases as pre-releases. Attach the exact tested DMG and its `.sha256` file. GitHub Release assets are uploaded separately from ordinary Git commits.
3. Include the version, Apple Silicon and macOS 15 requirements, source commit, SHA-256, VirusTotal report link and first-launch instructions in the release notes. State that the app is ad hoc signed and not Apple-notarised.
4. Publish only after testing and review. Download the published asset, verify its SHA-256 against the checksum, and test it. Do not overwrite an existing release asset: a changed file needs a new version and checksum.
5. After publishing and verifying the download, update `latest.json` on `main` with the new version, actual asset URL, filename, SHA-256 and completed VirusTotal report URL. Keep the existing field names and accurate platform, architecture, signing and notarisation values. Validate it with `python3 -m json.tool latest.json` and commit and push it. The website can fetch `https://raw.githubusercontent.com/abbeybridgeconsulting/necronomicdown-releases/main/latest.json`; it should handle fetch errors and validate the data. Do not point it to an unverified or unpublished asset.
6. Update the website if it does not read `latest.json`. Update the Homebrew tap's version, URL and SHA-256 once a tap exists. Until then, keep Homebrew marked coming soon.

## GitHub account and SSH

The release checkout's `origin` uses `git@github.com-abbeybridgeconsulting:abbeybridgeconsulting/necronomicdown-releases.git` and is configured to offer only the Abbey Bridge key. Verify the account with:

```sh
ssh -T -o IdentitiesOnly=yes git@github.com-abbeybridgeconsulting
```

A successful authentication says `Hi abbeybridgedev!` (SSH exits non-zero because GitHub does not offer a shell). If the key needs unlocking, use your normal SSH agent/Keychain setup. Do not use the `mcfr4g` account to push this repository. Git access and permission to create GitHub Releases are separate from generating a DMG.
