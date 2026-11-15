---
name: idea-to-open-source-github-app-skill
description: Turn an idea, a workaround, or a small tool an agent just built into a finished open-source app published on GitHub, and own the two jobs around it — finding existing open-source solutions before building anything, and making a repository discoverable (About, topics, README, visual assets). Use when the user says "make this an app", "post it to GitHub", "open source this", "is there an existing repo or library for this", "optimize this repo for stars or SEO", or when a session produced a working local script, daemon, menubar app, MCP server, CLI or plugin that others could use. Covers prior-art search and reuse, choosing the artifact shape, surviving reboots, a clean public tree, licence and upstream attribution, a one-command installer, real release assets, registry and plugin manifests, repo metadata, and a stranger-eye install test before calling it done.
---

# Idea to open-source GitHub app

A session often ends with something that works on one machine: a watcher script, a
patched fork, a small server. This skill carries it the rest of the way to a public
repository that a stranger can install with one command and that keeps working after
a reboot. Run every step yourself; do not hand steps back to the user.

This skill also owns two jobs that used to be separate: finding existing open-source
work before building (section 1), and making a repository discoverable and convincing
(sections 7 and 8). Use those sections on their own when that is all the task needs.

Related skills, when installed: `optimal-mcp-designer-skill` (MCP tool surface and
remote/stdio publishing), `plugin-creator` (plugin manifests),
`universal-mobile-deploy-skill` (store apps). Use them for their part instead of
repeating it here.

## Standing rules

- Minimum that works. Most of any idea already exists somewhere; search extensively,
  reuse, and add only the missing piece. Do not over-engineer.
- One product, one repository. Its app, website, plugins and MCP tools live together;
  split copies drift. When the source must stay in a private monorepo, publish a
  public mirror and run the mirror workflow from the public repo.
- Never ship a second variant beside the first. Change the existing app and put the
  choice in its UI or a flag (for example an endpoint picker: original service or
  self-hosted), then delete the superseded repo, folder, install and listing in the
  same task and prove nothing stale is left.
- Name it after the general job a stranger would search for, not a narrow feature or
  an internal codename.
- Anything with a window must look finished: little text, clear hierarchy, sensible
  size, restrained colour. An ugly app is not done.
- Credentials come from the owner's password manager or environment at the tool
  layer. Never ask for them and never commit them.
- Prove it with a real account and a real task, including the sign-in or OAuth path.
- Publishing includes the listings: every marketplace, registry and store that fits,
  submitted for review and followed until approved, with the approved links added to
  the README and website.

## 1. Look around first

Search first, write second. Do this before packaging an idea, and equally before
writing any non-trivial helper, wrapper or algorithm, before integrating an unfamiliar
API or SDK, and whenever an error message or stack trace appears.

```bash
gh search repos "<what it does> <platform>" --sort stars --limit 15 --json fullName,description,stargazersCount,pushedAt,license
gh search repos "<known tool> alternative" --sort stars --limit 10 --json fullName,description,stargazersCount
gh search code "<distinctive API, symbol, config key or exact error string>" --limit 15 --json repository,path
gh repo view <owner>/<repo> --json stargazersCount,description,pushedAt,licenseInfo,isArchived
```

Use a code-search tool over public repositories (a Grep MCP, when the runtime has one)
for literal call sites and error strings; start literal, narrow by language, path or
repo, and move to a regex only when the literal search is too weak. Read several
results, not the first hit.

Judge a candidate by evidence, not by its pitch:

| Signal | Good | Warning |
|---|---|---|
| Stars | 1k+ solid, 10k+ excellent; under 1k is fine for a young, active project | under 100 on a project several years old |
| Last commit | within 6 months | more than 2 years ago, or archived |
| Docs | clear README with working examples | sparse or outdated |
| Issues | maintainers answer | hundreds unanswered |
| Licence | present and compatible | none |

When the need is universal (format conversion, media download, file manipulation,
scraping, automation), the answer is usually an established tool: install it, run it
on the user's input, deliver the result, and name the project and link that solved it.
Adapt found code to the local codebase rather than pasting it, and cite the repository
and path that informed the implementation.

For the packaging decision:

- A maintained project that fits: use or extend it, and say so. Do not publish a clone.
- A project that proves the technique but lacks the product (a status tool where you
  need an automatic one, a library with no installer): build on it, credit it by name
  and link in the README, and state in one line what yours adds.
- Nothing: proceed. The README still needs a "how this differs from X" line for the
  nearest neighbours, because the first reader will ask.

Also recover the owner's own history: earlier repos of the same kind, how they were
named, licensed, installed and released. Match the best one; fix its known gaps
rather than copying them.

## 2. Choose the smallest shape that installs in one command

| The thing is | Ship it as | Install route |
|---|---|---|
| Background behaviour with no UI (watch, sync, reconnect) | Single binary or script + LaunchAgent / systemd user unit | `curl -fsSL <raw>/install.sh \| sh` |
| Needs a visible on/off or status | Menubar or tray app wrapping the same daemon | Release asset (DMG/ZIP), plus the script route |
| Tools for AI agents | MCP server (stdio via `npx`/`uvx`, optional remote `/mcp`) | `npx -y <pkg>` and a registry entry |
| Instructions and workflows for agents | Skill or plugin | `npx skills add <owner>/<repo>` or plugin marketplace |
| A command people type | CLI on npm / PyPI / Homebrew tap | `npx <pkg>`, `pipx install`, `brew install <owner>/tap/<name>` |

Pick one primary route and make it flawless. Add a second only when it serves a
different user (for example a DMG for non-terminal users).

## 3. Make it real before making it public

- Prove it on the real task it was built for, end to end, not a mock.
- Give it the three flags every background tool needs: `--status` (what it sees now),
  `--test` (exercise the hard step without side effects) and a force/once mode.
- Log one line per decision to a file under the user's log directory, with a timestamp.
- It must never fight the user. A tool that acts on a condition ("no network") acts
  only on that condition as the system reports it; anything more aggressive is opt-in
  behind a flag and off by default.
- Persistence on macOS: a LaunchAgent with `RunAtLoad` and `KeepAlive`, a
  `ThrottleInterval`, and stdout/stderr to the log. Do not set `ProcessType
  Background` on anything that relies on timers; the system defers them by a minute
  or more. Start/stop with `launchctl bootstrap|bootout gui/$(id -u)` and restart with
  `launchctl kickstart -k`. Verify by killing the process and seeing a new pid, and
  with `launchctl print gui/$(id -u)/<label>`.
- After install, read the restart count (`launchctl print` shows `runs`). A count that
  keeps climbing means something is killing it: commonly a local watchdog or reaper
  that matches a keyword in the command line. Add the service to that keep-list.
- A release build signed with a distribution identity must not share a bundle id with
  the development build; give the dev build its own id so the two do not corrupt each
  other's system permissions.
- If it depends on a private or undocumented API, an entitlement, or a permission
  prompt, say so in the README with the OS versions it was tested on.

## 4. Build the public tree from a clean copy

Publish a fresh repository made from the files that belong in it, not the working
folder with its history.

Leave out: agent and editor state (`.claude/settings.local.json`, session notes,
`AGENTS*.md` written for one machine), `.env` files, absolute home paths, personal
emails, device names, account identifiers, debug notes, test scratch files, build
output, and every binary or large media file. Binaries and videos go to Releases,
never into git.

Scan before the first push, then turn on the host's own protection:

```bash
command -v gitleaks >/dev/null || brew install gitleaks
gitleaks dir . --redact
grep -rInE '/Users/[a-z]+|/home/[a-z]+|[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[a-z]{2,}' --exclude-dir=.git . | grep -v noreply
```

A repository that ever held a secret is not flipped public; copy the current tree
into a new one. Revoked keys and rewritten history still leak.

## 5. Licence and upstream

- Original work: add `LICENSE` (MIT unless the owner uses another), name it in the
  README, and set the same SPDX id in the package metadata. No licence means nobody
  may use it.
- Built on someone else's code: keep their `LICENSE` and copyright lines intact, keep
  `NOTICE` for Apache-2.0, and add a visible "Based on <project> by <author>" line
  near the top of the README stating what was added.
- Keep the upstream as a second remote and merge it, so updates stay cheap:
  `git remote add upstream <url> && git fetch upstream && git merge upstream/main`.
- Give your own releases a tag prefix the upstream workflows do not match (for
  example `menubar-v1.2.0` when upstream triggers on `v*`), or their CI will try to
  publish their package from your repo.
- Anything you publish to a registry carries your own package name, repository URL
  and author. Remove or rewrite inherited manifests, funding files and policies that
  still describe the upstream project. One manifest per registry; no near-duplicates.

## 6. One-command installer

`install.sh` at the repo root, safe to re-run, with an `uninstall` argument. Shape:

```sh
#!/bin/sh
set -eu
NAME=<name>; LABEL=<reverse.dns.label>; REPO=<owner>/<repo>
BIN="$HOME/.local/bin/$NAME"; PLIST="$HOME/Library/LaunchAgents/$LABEL.plist"
if [ "${1:-}" = uninstall ]; then
  launchctl bootout "gui/$(id -u)/$LABEL" 2>/dev/null || true
  rm -f "$BIN" "$PLIST"; echo "removed"; exit 0
fi
mkdir -p "$HOME/.local/bin" "$HOME/Library/LaunchAgents" "$HOME/Library/Logs"
T=$(mktemp -d); curl -fsSL "https://github.com/$REPO/archive/refs/heads/main.tar.gz" | tar -xz -C "$T" --strip-components=1
( cd "$T" && make )            # or: download the release asset for this arch
install -m 755 "$T/$NAME" "$BIN"
cat > "$PLIST" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
<key>Label</key><string>$LABEL</string>
<key>ProgramArguments</key><array><string>$BIN</string></array>
<key>RunAtLoad</key><true/><key>KeepAlive</key><true/><key>ThrottleInterval</key><integer>5</integer>
<key>StandardOutPath</key><string>$HOME/Library/Logs/$NAME.log</string>
<key>StandardErrorPath</key><string>$HOME/Library/Logs/$NAME.log</string>
</dict></plist>
EOF
launchctl bootout "gui/$(id -u)/$LABEL" 2>/dev/null || true
launchctl bootstrap "gui/$(id -u)" "$PLIST"
echo "installed: $BIN (log: ~/Library/Logs/$NAME.log)"
```

The installer creates everything the app needs. If the README says "runs at login",
a file in the repo must write that login item; an app that only reads a plist someone
else created is broken for every new user.

Native macOS app:

```bash
for a in arm64 x86_64; do swiftc -O -target $a-apple-macos13.0 Sources/*.swift -o build/$NAME-$a; done
mkdir -p build/$NAME.app/Contents/MacOS && lipo -create build/$NAME-arm64 build/$NAME-x86_64 -output build/$NAME.app/Contents/MacOS/$NAME
cp Info.plist build/$NAME.app/Contents/ && codesign --force -s - build/$NAME.app
S=$(mktemp -d); cp -R build/$NAME.app "$S/"; ln -s /Applications "$S/Applications"
hdiutil create -volname "$NAME" -srcfolder "$S" -fs HFS+ -format UDZO -ov dist/$NAME-$VER.dmg
codesign -dv build/$NAME.app 2>&1 | grep -E 'Signature|TeamIdentifier'; spctl -a -vv build/$NAME.app
```

Build the DMG from a private staging folder as above. Tools that lay the window out through
Finder (`create-dmg`) open the volume on the owner's screen, where a stray paste or drop ends up
inside the published image. Mount every DMG and list its contents (`ls -A`) before uploading.

A menu bar app can be running and invisible: a full menu bar or a menu bar manager tucks new
icons behind an overflow arrow. Confirm the item through accessibility
(`osascript -e 'tell application "System Events" to tell process "<App>" to name of every menu item of menu 1 of menu bar item 1 of menu bar 1'`)
before calling the UI broken, and say so in the README.

Before shipping ad-hoc signed, look for an existing Developer ID identity; a missing one in
`security find-identity` is not proof there is none:

- List the team's certificates through the App Store Connect API (`GET /v1/certificates`). An
  active `DEVELOPER_ID_APPLICATION` entry means a private key exists on some Mac.
- Check every keychain file, not only the default search list:
  `for k in ~/Library/Keychains/*.keychain-db; do security find-identity -v -p codesigning "$k"; done`.
  Older login keychains (`login_renamed_N`) often still hold the key; unlock with
  `security unlock-keychain -p` (never `show-keychain-info`, which opens a dialog and hangs), run
  `security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k`, add the keychain to the
  search list for the build and restore the list afterwards.
- Only creating a new Developer ID certificate needs the account holder (the API answers 403).

Then sign with `codesign --options runtime --timestamp --sign <identity>`, notarize the zip and
the DMG with `xcrun notarytool submit <file> --key <AuthKey.p8> --key-id <id> --issuer <id> --wait`,
staple the app and the DMG (`xcrun stapler staple`), re-zip the stapled app, and run the hardened
build once: the hardened runtime can break what the ad-hoc build did. Keep both steps in the build
script behind environment variables so a contributor without the identity still gets a working
ad-hoc build. A notarized build is a new version number.

Always pin the deployment target; a build that inherits the build machine's OS
version will not launch on older systems. Describe signing as it is: `Signature=adhoc`
means "not notarized, first launch needs right-click > Open". Claim Developer ID or
notarization only after `spctl` says accepted.

## 7. README that is true, and that converts

Read the package metadata and entry points first and write only what the code does.
Find the one pain it removes and lead with it. The README is the project's landing
page; order:

1. Hero banner, name, one-sentence hook, badges (licence, release, platform, stars).
2. Why: the problem and the fix in a few lines, with a workflow diagram if it helps.
3. Quick start that works in under 30 seconds: the one install command and a minimal
   copy-paste example, then what happens after install and how to check it.
4. Features, shown with real output or mockup cards, and options and flags.
5. Comparison with the alternatives found in section 1 (a small honest table).
6. How it works (short), uninstall, limitations and tested versions.
7. Credit to prior art, contributing, licence.

Every command, flag, path, environment variable and claim is run or checked against
the code before publishing. Show real output; label a mockup as a mockup. Embed a demo
video from a Release or an issue upload rather than committing it.

Visual assets, saved under `.github/assets/` and linked from the README:

- A hero banner (about 1200x360), a workflow or architecture diagram, and one or two
  feature cards or terminal/UI captures.
- Make them with an image-generation tool when the runtime has one, or author clean
  vector SVG by hand (flat, minimal, dark `#0d1117` / `#161b22` background, subtle
  gradient, crisp type). Never draw them with a plotting or imaging script (PIL,
  matplotlib, OpenCV); the result always looks like a script drew it.
- Check that every image path resolves and renders on the public repo page.

## 8. Publish and set the metadata

```bash
git init -b main && git add . && git commit -m "Initial public release"
gh repo create <owner>/<repo> --public --source=. --remote=origin --push \
  --description "<what it does, for whom, in under 160 characters>"
gh repo edit <owner>/<repo> --homepage "<docs or demo URL>" \
  --add-topic <t1> --add-topic <t2> --add-topic <t3> \
  --enable-secret-scanning --enable-secret-scanning-push-protection
shasum -a 256 dist/* > dist/SHA256SUMS.txt
gh release create v<ver> dist/* --title "<name> <ver>" --notes "<what it does, install line, known limits>"
```

Description, topics, homepage and a licence are part of the release, not polish:

- About (160 characters at most): `<primary keyword> — <action-oriented value with
  secondary keywords>`, optionally led by one emoji.
- Up to 20 topics, real search terms in four groups of about five: ecosystem and
  language (`swift`, `macos`, `cli`), problem and solution (`hotspot`,
  `workflow-automation`), tool compatibility (`mcp`, `claude`, `codex`), and discovery
  intent (`menubar-app`, `zero-config`, `developer-tools`).
- Homepage: the docs, demo or download page. Add
`SECURITY.md`, `CONTRIBUTING.md` and issue templates when outside contributions are
expected. Check the result:

```bash
gh api repos/<owner>/<repo>/community/profile --jq '{health_percentage, files:(.files|with_entries(.value|=(.!=null)))}'
gh repo view <owner>/<repo> --json description,homepageUrl,licenseInfo,repositoryTopics,visibility
```

Store and directory reviewers read everything in the submission: remove retired
content, and do not name competing platforms in release notes. A shipped store version
is closed; a fix needs a new version number.

Where it applies, register it where users look: npm or PyPI, the MCP registry
(`server.json`), Smithery, a Homebrew tap, the plugin marketplace, the skills
registry. Read each registry's current docs before the first publish.

## 9. Install it as a stranger

Done means a person with only the README succeeds. From a clean location, with no
help from the working folder:

1. Run the README's install command exactly as written (fresh shell, temp `HOME` or
   a second machine when possible; otherwise uninstall first).
2. Confirm it runs: `--status`, the log line, the process, `launchctl print`.
3. Trigger the real condition once and watch it act.
4. Kill it and confirm it comes back; where a reboot is the promise, state plainly
   whether a reboot was actually tested.
5. Download the release asset from the public URL, compare its hash with
   `SHA256SUMS.txt`, and launch it. Run every other install route the README
   names (Homebrew, DMG first launch) the same way, and check each command against
   the installed tool version: flags disappear (`brew --no-quarantine` is gone in Homebrew 7).
6. Open the repo page logged out: images render, links resolve, the licence shows.
7. Run `uninstall` and confirm nothing is left.

Fix what fails and repeat from step 1. Then replace the local hand-built copy with
the published install so the owner runs what everyone else gets.

## Closing report

Repo URL, the install command, the release URL with asset hashes, what was verified
at which step above, what was not tested, and the prior-art projects credited.
