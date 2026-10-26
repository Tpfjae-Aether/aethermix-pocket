# AetherMix Pocket

The iPhone client for **AetherMix** — a native SwiftUI app that turns a phone
into a second window onto the desktop player.

It streams your PC's library over the local network, plays files imported into
its own Documents folder, keeps playing with the screen off, and wears the
desktop's themes.

| Room | What is there |
|---|---|
| HOME | spinning vinyl that slides out of the sleeve, transport, quick doors |
| LIBRARY | the room's songs, the PC's whole library, and files on the phone |
| ROOMS | playlist shelf with cover mosaics; create rooms, add and remove songs |
| VIBE | full screen: blurred artwork backdrop and synced words with per-song offset |
| STUDIO | desktop address, full theme grid, bloom and tint, imports |

**Streaming.** Tapping a desktop song opens `/audio?id=N` and plays it through
AVPlayer, so seeking works: the bridge serves HTTP byte ranges. Phone-owned
files play through AVAudioPlayer and need no desktop at all.

**Lyrics** come from the same `.lrc` files the desktop uses. The phone reads the
bridge's position twice a second and interpolates on the frame clock, so the
highlighted line tracks the audio instead of stepping.

**No Mac needed to build.** `.github/workflows/ios-ipa.yml` builds an unsigned
IPA on a GitHub macOS runner and attaches it as the
`AetherMix-unsigned-ipa` artifact.

---

## Build the IPA

1. **Actions → iOS IPA → Run workflow.**
2. Wait for the green tick, then open the run and download
   `AetherMix-unsigned-ipa`.
3. Check the archive before installing:

   ```bash
   python scripts/verify_ipa.py dist/AetherMix-unsigned.ipa
   ```

On a Mac, `./scripts/build_ipa.sh` does the same thing locally and writes
`dist/AetherMix-unsigned.ipa`.

macOS runner minutes are free on a public repository. On a private one they are
billed against your plan at a premium multiplier, and a build only takes a few
minutes either way.

---

## Install with Sideloadly

1. Windows: install **standalone** iTunes and iCloud (not the Microsoft Store
   builds), connect the phone, trust the computer.
2. Phone: **Settings → Privacy & Security → Developer Mode → on**, then restart.
   The toggle only appears after the phone has been connected once.
3. Sideloadly: drag the `.ipa` in, enter your Apple ID, Start.
4. Phone: **Settings → General → VPN & Device Management** → trust your Apple ID.
5. First launch: allow **Local Network**.

A free Apple ID signs the app for **7 days**, up to 3 apps at once; Sideloadly
re-signs over USB before it expires. No paid account is needed for any feature
here.

---

## Point it at the desktop

Start AetherMix on the PC — its log prints
`[http] phone: http://192.168.x.x:3690`. On the phone, open
**STUDIO → THE DESKTOP**, type that address and tap **LINK**. Both devices must
be on the same network, and Windows Firewall has to allow AetherMix on the
profile that network uses.

The bridge listens on `0.0.0.0:3690`. Callers from the network may only use the
phone's routes; anything that writes files, shells out to `yt-dlp`, switches
accounts or quits the app answers `that route is local only`.

---

## Checks that run without a Mac

```bash
python tools/swift_check.py        # every Swift file parses, every member resolves
python scripts/test_verify_ipa.py  # prove the verifier rejects broken archives
python tools/contract_test.py      # the live desktop bridge against the Swift models
```

---

## Layout

```
.github/workflows/ios-ipa.yml   builds the IPA on a macOS runner
project.yml                     XcodeGen manifest, iOS 16, iPhone only
AetherMix/Info.plist            background audio, local network, file sharing, ATS
AetherMix/Resources/            app icon and accent colour
AetherMix/Sources/              the app: RootView, Atmosphere, Home, Library,
                                Playlists, NowPlaying, Settings, Bridge,
                                PlayerEngine, Store, Theme, Models
scripts/                        build, verify, and bundle tooling
tools/                          static check and live contract test
docs/IOS_PORT.md                the port study: constraints, findings, device tests
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| OFFLINE pill, "No answer from the desktop" | wrong address, different network, desktop not running, firewall |
| Connects but playback is silent | check the volume; the audio session is `.playback`, so the mute switch does not matter |
| Stops launching after a week | free-signing expiry, re-run Sideloadly |
| "Unable to install" | the 3-app limit, Developer Mode off, or a bundle id clash |
| Local Network prompt never appeared | Settings → Privacy & Security → Local Network → enable AetherMix |
| Blank home-screen icon | archive built without the asset catalog; run `verify_ipa.py` |
