# Peek: a 3D TV remote with channel preview

A web prototype of a TV remote app that feels physical: sprung 3D keys, click sounds and haptics. Its main feature is a **peek** screen. Pressing CH ▲▼ (or swiping the remote's display) shows a live preview of another channel on the remote. The TV only changes when you press **OK**.

Open `web/index.html` in a browser. There's no build step and nothing to install.

## Try it

| Action | Touch | Keyboard |
|---|---|---|
| Peek at the next or previous channel | CH rocker, D-pad, or swipe the display | ↑ ↓ ← → |
| Tune to the channel you're peeking at | OK, or tap the preview | Enter |
| Type a channel number (previews it first) | 0–9 (ENTER tunes straight away) | 0–9 |
| Guide with live thumbnails of every channel | GUIDE | G |
| Volume, mute, last channel, info, power | rockers and keys | + − M L I P |

Holding CH or VOL repeats the key. A peek closes by itself after 8 seconds.

### Streaming peek (APPS)

| Action | Touch | Keyboard |
|---|---|---|
| Browse streaming titles on the remote's display | APPS, then ◀ ▶ or swipe; CH ▲▼ switches service | A, then ← → / ↑ ↓ |
| Open details: trailer, extras, scores, reviews, where to watch | OK or tap the display | Enter |
| Deep link the title to the TV | **Play on TV** (or OK again) | Enter |

The details panel shows the exact request a real app would send for Roku, Google TV, LG webOS or Samsung. **Open on this phone** shows the universal link that would open the service's own app. The services (Northstar, Tidewell, Kinoroll), titles and reviews are sample data.

### One screen vs two

- **One screen:** details slide up as a sheet over the remote. After Play on TV it closes so you can see the TV.
- **Two panes:** on wide screens (an unfolded or dual-screen phone, a tablet in landscape, a desktop) the TV and details sit in one pane and the remote stays in the other. That way you can watch a trailer and keep using the remote. If the browser reports a hinge (`horizontal-viewport-segments: 2`), each pane fits one screen and nothing lands in the fold.

## What's real and what's simulated

- **Feel**: real. Each key is a cap on a spring above a well cut into the body. It drops on `pointerdown` and springs back past its rest position on release. Clicks are synthesized with WebAudio. Haptics use `navigator.vibrate` on Android. iOS Safari 18+ doesn't support that, so the page toggles a hidden `<input type="checkbox" switch>`, which makes the Taptic Engine tick.
- **TV and channels**: simulated. Nine made-up channels are drawn live on canvas. Their programme guide follows the real clock, so the times and progress bars are realistic.

## Building the real app

### The remote part is straightforward

Phones control TVs over the local network. Most brands publish or have well-documented protocols:

| TV platform | Protocol | Pairing |
|---|---|---|
| Roku | ECP: HTTP on port 8060 (`/keypress/…`, `/query/tv-channels`, `/query/tv-active-channel`) | None, but "Control by mobile apps" must be enabled on the TV |
| Samsung Tizen | WebSocket on 8001/8002 | Allow prompt on the TV, then a token |
| LG webOS | SSAP WebSocket on 3000/3001 (`tv/getChannelList`, `tv/getCurrentChannel`, `tv/getChannelProgramInfo`) | PIN or prompt on the TV, then a client key |
| Android TV / Google TV | Android TV Remote protocol v2 (TLS on 6466/6467) | Code shown on the TV |
| Fire TV | ADB over the network, or the Fire TV Lightning protocol | Prompt on the TV |
| Vizio SmartCast | HTTPS REST on 7345/9000 | PIN on the TV |

Discovery uses SSDP/mDNS. On iOS this needs the *Local Network* permission and Bonjour/multicast entitlements.

### The preview part needs a different approach

**You can't get a live video preview of another channel from the TV itself.** A TV has one tuner showing one channel, and none of these protocols send video back to the phone. What you can do:

1. **Guide-data preview (works with any TV).** Show the channel's current show, artwork, time left and what's next. Data comes from the TV's own guide (LG `tv/getChannelProgramInfo`, Roku `/query/tv-channels`) or from a guide provider: Gracenote (commercial), Schedules Direct (cheap, personal use) or XMLTV. This is the realistic v1.
2. **Live preview from a separate tuner.** Devices like HDHomeRun have 2–4 network tuners and stream each channel over HTTP (`/lineup.json`, then `http://<ip>:5004/auto/v<ch>`), so the app can decode a few frames of a channel the TV isn't showing. Channels DVR, Plex Live TV and Jellyfin offer the same through their APIs. This gives you exactly what the prototype shows, but only for people who own the hardware, which would make a good premium tier.
3. **Streaming "channels" are apps.** Most viewing today happens in apps, not on broadcast channels. Here a peek means app art plus "continue watching", and it's mostly limited to what each platform exposes (Roku `/query/apps`, app icons and so on).

### Streaming: trailers, reviews and deep links

| Need | Source | Notes |
|---|---|---|
| Which service has a title, plus deep links | Watchmode, Streaming Availability (Movie of the Night), JustWatch partner API | Paid. These return web and iOS/Android links, so you don't map each service's IDs yourself |
| Trailers, teasers, featurettes, behind the scenes | TMDB `/movie/{id}/videos`, `/tv/{id}/videos` | Mostly YouTube keys; play with the official YouTube player. Commercial apps need a TMDB commercial agreement |
| Scores | OMDb (IMDb, Rotten Tomatoes, Metacritic); TMDB user reviews | Full critic reviews need a licensing deal |
| Open the title on the phone | The service's universal link (e.g. `netflix.com/title/<id>`) | iOS opens the app if it's installed |
| Open the title on the TV | Roku `POST /launch/<appId>?contentId=…&mediaType=…`; Google TV / Fire TV app link; LG `system.launcher/launch`; Samsung `ed.apps.launch` | Support varies by app; Netflix is the most locked-down |

A user's own watch history and "continue watching" aren't available from any service.

### Recommended native stack (iOS)

- SwiftUI for the layout. Keys can be SwiftUI views with layered shadows and a spring animation, which looks like this prototype. For true 3D, use RealityKit or SceneKit (that's how Calki's "gone 3D" style works).
- Core Haptics for custom press and release patterns: a sharp transient on press and a softer one on release.
- `AVAudioEngine` for low-latency click samples.
- `Network.framework` for SSDP discovery and WebSockets.

### Before you commit to this niche

- The chart's $613K–$2.1M per month is the top 6 apps only. Most TV remote apps make far less.
- Apple rejects many remote apps under guideline 4.3 (spam / saturated category). A real point of difference, like tactile design or peek, is what gets you past review.
- Don't use TV brand names or logos in a way that suggests the app is official. "Works with Samsung" is fine; a Samsung logo as your icon is not.
- Many top earners rely on aggressive weekly-subscription paywalls. Reviews punish this, so price carefully.
