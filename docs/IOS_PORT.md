# AetherMix on iPhone — port study and build guide

*Started 2026-10-26, updated later the same day. The desktop app is Windows/Qt
6.12 + Rust. This document covers what can move to iOS, what cannot, the
architecture that was chosen, and the exact steps to get a working IPA onto a
phone with Sideloadly.*

---

## 0. Status, stated plainly

The Swift app is complete in source: 13 files, ~3,300 lines, iOS 16 target,
iPhone-only, with background audio, lock-screen controls, LAN streaming,
phone-owned files, playlists on both sides, synced lyrics and the desktop theme
set. The desktop side of the link is built, running and **tested end to end**
(LAN bind, route guard, library, playlists, ranged audio, play/seek/volume/pause,
lyric timing data, all 20 themes).

What could **not** be produced here is the binary. This machine is Windows and
has no Swift compiler or iOS SDK; Swift for Windows cannot target arm64/iOS and
no anonymous macOS builder exists. An `.ipa` is a compiled `arm64` Mach-O plus
the bundle, so it has to come from Xcode (a Mac, or the included GitHub macOS
workflow). Everything around that one step is finished:

* `ios/scripts/build_ipa.sh` — one command, Mac or CI, builds and packages.
* `.github/workflows/ios-ipa.yml` — macOS runner, uploads the `.ipa` artifact.
* `ios/scripts/setup_github_build.sh` — creates a private repo from **only** the
  iOS folder and downloads the artifact, for when there is no Mac.
* `ios/scripts/verify_ipa.py` — checks the resulting archive the way Sideloadly
  and iOS will (arm64 slice, plist keys, background audio, LAN description, ATS,
  icon). Tested against good and deliberately broken archives.

Do not treat the app as "verified working" until it has been launched on a
device. Section 8 lists the device tests to run.

---

## 1. Verdict up front

A **native SwiftUI client that talks to the desktop AetherMix over the LAN**, plus
a **local shelf for imported files**, is the only architecture that is both
buildable and honest on iOS.

A straight port of the Qt/QML app cannot work as-is:

| Desktop piece | iOS reality |
|---|---|
| `yt-dlp.exe`, `ffmpeg.exe` spawned as child processes | **Not allowed.** iOS apps cannot spawn subprocesses; the sandbox and code-signing model forbid it. |
| librespot (Spotify Connect) as a dependency | It compiles for iOS (Rust), but Spotify's SDK/Connect terms and the free-tier restriction make it a poor sideload story. |
| Qt Quick / QML UI | Qt 6.8–6.12 supports iOS 16+, but **requires macOS + Xcode** to build and links a very large Qt runtime. A native SwiftUI build is far smaller and gets system behaviours (lock screen, Now Playing, safe areas) for free. |
| A local HTTP bridge on port 3690 | Fine on the desktop; the phone is a client of it. Do not run the bridge *on* the phone as an always-on daemon — iOS suspends background apps. |
| Global hotkeys, mini bar, tray | Not applicable; replaced by Lock Screen controls and the system Now Playing panel. |

YouTube extraction stays on the PC. The phone either:

1. **streams a file row from the PC** (`/audio?id=…`, byte ranges), or
2. **plays files imported into its own Documents folder**, or
3. **controls the PC** for rows that only exist as web streams. (A YouTube row
   has no file to serve, so it is remote-controlled — deliberately.)

That is the same split the desktop already has between downloaded rows and
streamed rows, just spread across two devices.

---

## 2. The researched constraints

### 2.1 Sideloadly and free signing (confirmed 2026)

- Sideloadly runs on Windows and needs the **standalone/web iTunes + iCloud**
  builds (not the Microsoft Store versions), plus Developer Mode on the phone.
- It accepts a **plain unsigned IPA** — a zip whose root is `Payload/App.app` —
  and re-signs it for the connected device. `CODE_SIGNING_ALLOWED=NO` output is
  exactly what it expects, as long as the executable is a real arm64 iOS binary.
- A **free Apple ID (Personal Team)** is enough. The limits are unchanged:
  **7-day profile expiry**, **3 apps per device**, **10 App IDs per 7 days**.
  Sideloadly can re-sign on a schedule while the phone is connected.
- **Developer Mode** is required on iOS 16+ for development-signed apps:
  *Settings → Privacy & Security → Developer Mode*, then restart. The toggle
  appears only after the phone has been used for development/sideloading once.
- Accepting an IPA does **not** mean an arbitrary zip works: a simulator build
  (x86_64/arm64-simulator), a missing executable or a corrupt bundle fails with
  “Invalid IPA” / “Unable to install”. `verify_ipa.py` exists to catch those
  before the cable comes out.

### 2.2 Capabilities that survive a free profile

- **Background audio** (`UIBackgroundModes: audio`) — a plist mode, not a paid
  entitlement. This is what keeps playback alive with the screen off.
- **Now Playing + MPRemoteCommandCenter** — public MediaPlayer APIs.
- **Local network** — `NSLocalNetworkUsageDescription` + the user's permission.
  Bonjour would need `NSBonjourServices`; this app does not browse, so no
  multicast entitlement is involved.
- **Files import / file sharing** — `UIFileSharingEnabled` +
  `LSSupportsOpeningDocumentsInPlace`; the folder shows up under
  *On My iPhone → AetherMix*.
- **App Groups** — effectively unavailable on a free team. Everything the phone
  owns lives in its own Documents folder.
- **Keychain** — app-private items are fine; shared access groups are not.

### 2.3 Consequences

- No background sync daemon, no always-listening server on the phone.
- The LAN link is plain HTTP to a numeric address: ATS allows local networking,
  not the whole internet.
- The app must degrade gracefully when the desktop is gone: the library screen
  shows the phone shelf, the mini player keeps playing imported files.

### 2.4 Rights and distribution

YouTube's terms prohibit downloading or otherwise using content outside the
service's own playback, and Apple guideline 5.2.3 targets apps that save or
convert third-party media without authorisation. Sideloading skips App Review
but **does not grant rights**. Extraction stays on the user's own PC; the phone
streams from it and treats imported copies as the user's own files.

---

## 3. What was built

```
ios/
  project.yml                 XcodeGen manifest: iOS 16, iPhone only, no Mac Catalyst
  AetherMix/Info.plist        background audio, LAN, ATS, file sharing
  AetherMix/Resources/        Assets.xcassets: AppIcon (single 1024, no alpha), AccentColor
  AetherMix/Sources/
    AetherMixApp.swift        @main
    RootView.swift            AppModel, top bar, mini player, glass dock, sheets
    Atmosphere.swift          radial light fields, motes, vinyl, equalizer, dock
    HomeView.swift            vinyl room, transport, doors, room rows
    LibraryView.swift         search, DESKTOP / ON PHONE, fade-masked rows
    PlaylistsView.swift       room shelf, remote + local detail views
    NowPlayingView.swift      immersive cover backdrop, words panel, scrubber
    SettingsView.swift        connection, theme grid, atmosphere, shelf
    Bridge.swift              HTTP client for the desktop bridge
    PlayerEngine.swift        AVPlayer + AVAudioPlayer, session, lock screen, lyric clock
    Store.swift               local shelf (Documents) + library store
    Theme.swift               theme document, palette, glass surface
    Models.swift              bridge payloads
  scripts/                    build_ipa.sh, setup_github_build.sh, verify_ipa.py
.github/workflows/ios-ipa.yml macOS runner that builds and uploads the artifact
```

### 3.1 Feature parity

| Desktop feature | iOS status |
|---|---|
| Library browsing, search | Yes (desktop rows over LAN; phone files offline) |
| Playlists / rooms | Yes: switch, create, append, remove on the desktop; create, add, remove, delete, cover on the phone |
| Playlist link import | Desktop BROWSE room does the import; the result appears in ROOMS on the phone |
| Cover art | Yes, full rows and the 4-up mosaic for playlist cards |
| Playback | Phone files through AVAudioPlayer; desktop file rows streamed through `/audio` with ranges |
| Web-only rows (YouTube/SoundCloud) | Controlled on the desktop (play/pause/next/seek); cannot be decoded on-device |
| Synced lyrics | Yes for desktop rows (`/lyrics`), with per-song offset via `/lyricoffset`; phone files show the LRC prompt |
| Immersive room | Yes: blurred artwork backdrop, self-timed words, scrub, transport |
| Themes | Yes: every desktop theme through `/themes` + preview, applied with `/theme`; six built-ins offline |
| Theme creation | Desktop Studio owns the forge; the phone selects and previews |
| Import audio | Yes: Files picker → Documents/AetherMix, plus Finder/iCloud drop through file sharing |
| Background / lock screen | Yes: audio session, Now Playing, headphone and lock-screen commands |
| Global hotkeys / tray | N/A on iOS |

### 3.2 The two desktop additions this port needed

**`GET /audio?id=N`** — serves the row's file with HTTP byte ranges (206 +
`Content-Range`), `HEAD`, and mime by extension. AVPlayer needs ranges to seek.
Read-only, validated against the library, never an arbitrary path.

**LAN binding with a route allowlist** — the bridge now listens on `0.0.0.0:3690`
by default (`AETHER_LAN=0` restores loopback-only) and prints
`[http] phone: http://<LAN-IP>:3690` at boot. Requests from a non-loopback peer
are limited to the routes a phone needs:

```
/state /library /playlists /lyrics /lyricoffset /themes /theme /theme/current
/audio /art /cover /play /pause /resume /toggle /next /prev /seek /volume /stop
/playlist/switch /playlist/append /playlist/remove /playlist/new /track/save
```

Everything that can spawn an extractor, import an arbitrary file, swap an
account or shut the app down answers
`{"ok":false,"error":"that route is local only"}` to the LAN. Playlist edits and
`/track/save` *are* allowed, because that is how the phone builds rooms and saves
a song for offline play. The desktop's own QML keeps using `127.0.0.1` and is
unaffected. This is deliberately not token-authenticated: a home LAN can change
the song and the rooms, but cannot write outside the library or run extractors.

---

## 4. Build and install

### 4.1 Requirements

- **Route A:** any Mac with Xcode 15/16 + XcodeGen (`brew install xcodegen`).
- **Route B:** a GitHub account and the `gh` CLI; the macOS runner does the rest.
- **Sideloading:** Windows PC with standalone iTunes + iCloud, a cable, an Apple
  ID (free is fine), and Developer Mode enabled on the phone.

### 4.2 Route A — one command on a Mac

```bash
cd ios
./scripts/build_ipa.sh
# -> ios/dist/AetherMix-unsigned.ipa
```

The script generates the project, builds `generic/platform=iOS` in Release with
`CODE_SIGNING_ALLOWED=NO`, then asserts: bundle id and executable exist, the
Mach-O contains **arm64**, `UIBackgroundModes` contains `audio`, and
`NSLocalNetworkUsageDescription` is present. It zips `Payload/AetherMix.app` and
prints the sha256.

To build in Xcode instead: `xcodegen generate && open AetherMix.xcodeproj`,
select a personal team if you want a signed build, then Archive → Distribute →
Custom → Copy App and zip `Payload/`.

### 4.3 Route B — no Mac

```bash
cd ios
./scripts/setup_github_build.sh          # private repo, workflow, download
```

The helper copies **only** `ios/` (minus `dist`, `build`, `.xcodeproj`) and the
workflow into a temporary repo. Your music library, `target/`, caches and the
desktop sources are never uploaded. It then runs **iOS IPA**, waits, and puts
the artifact in `ios/dist/`.

Manually: push the repository, open **Actions → iOS IPA → Run workflow**, and
download `AetherMix-unsigned-ipa` when it finishes. The workflow runs on
`macos-15` (Xcode 16) and needs no secrets because the build is unsigned.

### 4.4 Check the archive

```bash
python ios/scripts/verify_ipa.py ios/dist/AetherMix-unsigned.ipa
```

Checks the absolute blockers — `Payload/*.app`, arm64 Mach-O (not a simulator
slice), `CFBundle*` keys, `MinimumOSVersion`, background audio, the local-network
description, ATS, file sharing, launch screen and icon — and warns about the
rest. Exit code 0 means Sideloadly should accept it.

### 4.5 Sideload

1. Windows: standalone iTunes/iCloud, plug in, trust.
2. Phone: Settings → Privacy & Security → **Developer Mode** → on → restart.
3. Sideloadly: drag the `.ipa`, Apple ID, Start.
4. Phone: Settings → General → VPN & Device Management → trust the certificate.
5. Launch, allow **Local Network**.

Free-team notes: lasts **7 days**; re-run Sideloadly (auto-refresh can do it over
Wi-Fi) before it stops launching. A second copy counts toward the 3-app limit.

### 4.6 Point it at the desktop

Start AetherMix; the log prints `[http] phone: http://192.168.x.x:3690`, and the
app ships with `192.168.x.x:3690` pre-filled (this machine's address on the
network it was built for). Edit it in **STUDIO → THE DESKTOP** and press LINK.
Both devices must be on the same network and Windows Firewall must allow the app
on private networks.

---

## 5. Why not CI-produce a *signed* IPA?

Signing needs a certificate and a provisioning profile tied to a team, stored as
secrets, plus an App Store Connect key or a `.p12`. That is only worth doing with
a paid account. Sideloadly re-signs the unsigned archive with the user's Apple ID
on the PC, which is the whole point of the free route.

Cloud Macs (MacStadium, Scaleway, Codemagic) follow Route A verbatim.

---

## 6. Verification status

**Verified in this environment**

| Check | Method |
|---|---|
| Rust bridge compiles | `cargo check` + `cargo build --release` (target `x86_64-pc-windows-gnu`) |
| LAN bind | boot log: `bridge live on 0.0.0.0:3690`, `phone: http://192.168.x.x:3690` |
| Route guard | `/state`, `/library`, `/themes`, `/audio` answer from the LAN IP; `/watch` returns `that route is local only`; `/accounts` still answers on loopback |
| Streaming | `Range: bytes=0-1023` on `/audio?id=0` returned **206** with 1024 bytes |
| Transport over LAN | `/play` started a song, position advanced, `/seek?ms=60000` moved to 64 s, `/volume?v=40`, `/pause` reported `playing:false` |
| Playlist switch | encoded absolute path switched to the second room (84 songs) and back to the main room (106 songs) |
| Lyrics payload | `/lyrics?id=5` returned `synced:true`, 2,188 chars of LRC with `[mm:ss.xx]` stamps |
| `/theme/current` | falls back to `frutiger`/`aurora` on a fresh profile instead of `{"error":"theme missing"}` |
| Swift model contract | `.scratch/ios_contract.py` validates every required key/type in `/state`, all 106 tracks, both playlists, all 20 themes and `/lyrics` — **OK** |
| Swift syntax | tree-sitter Swift parse of all 13 files — **0 errors** |
| IPA verifier | tested against 6 synthetic archives (good, x86 slice, missing exe, empty plist, missing LAN key, no background audio) — correct verdicts |
| Plist | valid XML; bundle id/executable hard-coded so a build-setting expansion failure cannot produce an unbundleable app |

**Not verified (no Swift toolchain, no device)**

- Compilation of the Swift sources. They were written against iOS 16 APIs,
  audited line by line, and parsed, but the first `xcodebuild` may still surface
  a fix-it. Expect type-level nits, not architectural ones.
- Any on-device behaviour: launch, local-network prompt, background audio, lock
  screen, streaming quality, lyric sync on a real speaker.

---

## 7. Known limitations

1. **No on-device YouTube decoding.** By design; a subprocess is impossible and
   an in-process extractor would need a signed FFmpeg framework.
2. **Streamed rows are remote-controlled** unless the desktop saved them.
   `/track/save` on the desktop turns a web row into a file row, which the phone
   can then stream.
3. **No discovery.** The host is typed once (pre-filled here). The next step is
   Bonjour on the desktop plus `NWBrowser` on the phone, which would need
   `NSBonjourServices` and the local-network permission (already declared).
4. **No offline mirror of the desktop library.** Only files imported by hand.
5. **Free signing expires weekly.** Inherent to a Personal Team.
6. **The phone files live in the app container.** Deleting the app deletes them;
   the Files app folder is the backup path.
7. **Windows Firewall may need one approval** the first time the bridge binds to
   `0.0.0.0`; without it, the phone sees the PC as offline while the desktop
   itself keeps working.

### Device test list (run after installing)

1. Launch with the desktop off — no crash, OFFLINE pill, phone files play.
2. Start the desktop, LINK, library count matches the PC (106 here).
3. Stream a local file row; seek near the end; let it finish → next row plays.
4. Lock the phone; check the lock-screen art, title, scrub, next/prev.
5. Receive a call mid-song; playback pauses and resumes.
6. Play a desktop row with synced words; the highlight must lead/lag by less
   than ~150 ms; use ± to correct and reopen the song to confirm it persisted.
7. Switch rooms on the phone; confirm the desktop's active playlist changed.
8. Create a phone room, add two files, remove one, delete the room, relaunch.
9. Change the theme in STUDIO; background, dock and words recolour; desktop
   follows.
10. Turn Wi-Fi off; the app must not hang (4 s timeouts) and must fall back to
    the phone shelf.

---

## 8. Sources

- Sideloadly FAQ and limits — https://sideloadly.io/faq.html
- Sideloadly unsigned-IPA flow — https://support.corellium.com/features/apps/sideloadly
- Unsigned IPA build example — https://dev.to/oivoodoo/build-unsigned-ios-ipa-to-install-via-sideloadly-236f
- Apple, Personal Team limits (7 days / 3 apps / 10 App IDs) — https://developer.apple.com/help/account/basics/about-your-developer-account/
- Apple, Developer Mode — https://developer.apple.com/documentation/xcode/enabling-developer-mode-on-a-device
- Apple, distributing to registered devices — https://developer.apple.com/documentation/xcode/distributing-your-app-to-registered-devices
- Apple, background execution modes — https://developer.apple.com/documentation/xcode/configuring-background-execution-modes
- Apple, `UIBackgroundModes` — https://developer.apple.com/documentation/bundleresources/information-property-list/uibackgroundmodes
- Apple, media playback configuration — https://developer.apple.com/documentation/avfoundation/configuring-your-app-for-media-playback
- Apple, Now Playing — https://developer.apple.com/documentation/mediaplayer/mpnowplayinginfocenter
- Apple, local network privacy — https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy
- Apple, multicast entitlement (not used) — https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.networking.multicast
- Apple, `UIFileSharingEnabled` — https://developer.apple.com/documentation/bundleresources/information-property-list/uifilesharingenabled
- Apple, `NavigationStack` — https://developer.apple.com/documentation/swiftui/navigationstack
- Apple, `presentationDetents` — https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:)
- Apple, `toolbarBackground` — https://developer.apple.com/documentation/swiftui/view/toolbarbackground(_:for:)
- Apple, `scrollContentBackground` — https://developer.apple.com/documentation/swiftui/view/scrollcontentbackground(_:)
- Apple, async AVAsset loading — https://developer.apple.com/documentation/avfoundation/avasset-async-properties
- Apple, `UTType.mp3` — https://developer.apple.com/documentation/uniformtypeidentifiers/uttype-swift.struct/mp3
- Apple, signature troubleshooting — https://developer.apple.com/library/archive/technotes/tn2318/_index.html
- Apple DTS on subprocesses in iOS apps — https://developer.apple.com/forums/thread/747499
- Apple, AVPlayer — https://developer.apple.com/documentation/avfoundation/avplayer
- Apple App Review Guidelines 5.2.3 — https://developer.apple.com/app-store/review/guidelines/
- YouTube Terms of Service — https://www.youtube.com/static?template=terms
- GitHub Actions macOS runners — https://docs.github.com/en/actions/concepts/runners/github-hosted-runners
- IPA format (Library of Congress) — https://www.loc.gov/preservation/digital/formats/fdd/fdd000593.shtml

---

## 9. Verification pass — 2026-10-26

There is no macOS toolchain on the PC that owns this project, so the Swift
client cannot be compiled here. Everything that *can* be proven off a Mac was
proven, and every failure mode that would make the IPA install and then not
work was looked for directly.

### What was proven

1. **Every Swift source parses.** `ios/tools/swift_check.py` (tree-sitter-swift)
   over all 13 files in `ios/AetherMix/Sources`: 0 syntax errors.
2. **No unresolved symbols in the data path.** The same script cross-checks every
   `Bridge.shared.<member>`, `theme.<key>`, `engine.<member>`,
   `store.<member>` and `shelf.<member>` reference against the declarations in
   the same sources: 0 unknown members (one false positive, the local `shelf`
   URL constant).
3. **The wire contract holds against the live desktop.** `ios/tools/contract_test.py`
   reads the `Codable` structs out of the Swift sources and validates the real
   `/state`, `/library`, `/library?scope=playlist`, `/playlists`, `/lyrics`,
   `/theme/current` and `/theme?name=…` payloads against them — a missing
   non-optional field is exactly the silent "empty room" bug. 44 checks pass.
   It also opens real 206 byte-range streams (`Content-Range`, `audio/mp4`),
   proxies real covers through `/cover`, restores transport/volume, and runs a
   temporary room through create → switch → append → remove → switch back,
   deleting the temp file afterwards.
4. **No iOS 17-only API.** The deployment target is 16.0 with no iOS-17
   symbols (`@Observable`, `symbolEffect`, `contentMargins`, two-parameter
   `onChange`, `ContentUnavailableView`…) in the sources.
5. **The app icon is a real 1024×1024 RGB PNG** with no alpha, matching the
   single-size `AppIcon` set Xcode 15+ expects.

### Bugs found and fixed by this pass

- A stream borrowed from a room loaded the **desktop's** lyric id, so the words
  belonged to whatever the PC was playing. Words are now keyed to the stream id.
- `next`/`previous` asked the phone's own shelf first, so the queue was
  hijacked by local files. A borrowed stream now asks the room first.
- The LRC reader understood one timestamp per line. It now handles repeated
  stamps, `[offset:]`, metadata tags, `mm:ss:xx`, `hh:mm:ss.xx` and enhanced
  `<..>` word tags.
- A phone import copied the file on the main actor (freezing the room) and
  ignored a sidecar `.lrc`. Both are fixed.
- The immersive room showed a grey disc for every phone stream instead of the
  artwork it already had.
- `/library` mixes the active room with the PC's local vault, so the phone
  showed 106 songs for a 73-song room. LIBRARY now has three honest filters —
  THE ROOM, PC FILES, ON PHONE — with counts that match the room card.
- A failed create closed the sheet silently; a new desktop run was invisible
  until relaunch; "Save for offline play" actually downloaded to the PC and is
  now labelled "Download to the desktop".
- Polling hammered an unreachable bridge every 0.75 s; it now backs off to 3 s.

### The one manual step this PC needs

The iPhone reaches the PC over the LAN, and Windows currently has two
**inbound Block rules** for `aethermix.exe` on the Public profile (created when
the firewall prompt was cancelled). The bridge answers on `192.168.x.x:3690`
from the PC itself, but a phone on the same Wi-Fi will time out until that is
cleared. As administrator on the PC:

```bat
netsh advfirewall firewall delete rule name=aethermix
```

then relaunch AetherMix and choose **Allow** on **Private networks**. The
phone's STUDIO screen now says this in plain words when the link fails.

### Still requires a Mac (or the GitHub runner)

`python ios/scripts/test_verify_ipa.py` proves the gate itself (10 synthetic
archives: arm64 vs simulator, fat binaries, wrong layout, missing audio mode,
missing ATS, missing local-network key) before it judges the real one.
`ios/scripts/build_ipa.sh` is the whole build: XcodeGen → `xcodebuild -sdk
iphoneos CODE_SIGNING_ALLOWED=NO` → `Payload/AetherMix.app` → zip. The workflow
in `.github/workflows/ios-ipa.yml` runs it on `macos-15`, then runs
`verify_ipa.py` on the archive before uploading it. The install is then
Sideloadly with a free Apple ID.
