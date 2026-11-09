# Projects

A single index of the things I have built, with what each one does, how it works, and where the code lives. The three native Mac apps at the top are the ones I use every day; the four systems below them each started as a question about how software behaves under load, noise or adversarial input.

Each folder in this repo holds the full README for that project, so the write-ups stay together even as the individual repos move on.

| # | Project | What it is | Stack | Code |
|---|---|---|---|---|
| 01 | [Jarvis](#01--jarvis) | Voice assistant that lives in the MacBook notch, with Claude as the brain and permissions enforced in code | Swift, SwiftUI, ONNX Runtime, Claude API | [samarthpaliwal/Jarvis](https://github.com/samarthpaliwal/Jarvis) |
| 02 | [MacStream](#02--macstream) | Mac to iPad screen streaming at 20 ms end to end, with touch, Pencil and keyboard control flowing back | Swift, VideoToolbox, ScreenCaptureKit, Network.framework | [samarthpaliwal/MacStream](https://github.com/samarthpaliwal/MacStream) |
| 03 | [Spotify Widget](#03--spotify-widget) | A vinyl record on the macOS desktop that spins with the current Spotify track | Electron, TypeScript, AppleScript | [samarthpaliwal/SpotifyWidget](https://github.com/samarthpaliwal/SpotifyWidget) |
| 04 | [Adaptive market-making simulator](#04--adaptive-market-making-simulator) | Quotes both sides of a market while volatility shifts underneath it | Python, asyncio | [samarthpaliwal/adaptive-market-making-simulator](https://github.com/samarthpaliwal/adaptive-market-making-simulator) |
| 05 | [Event-processing platform](#05--event-processing-platform) | Distributed pipeline that ingests, validates and routes high-volume event streams | Python, asyncio, Docker, AWS | [samarthpaliwal/event-processing-platform](https://github.com/samarthpaliwal/event-processing-platform) |
| 06 | [RAG-based AI assistant](#06--rag-based-ai-assistant) | Retrieval-augmented assistant with an evaluation harness built in from the start | Python, LangChain, vector DB | [samarthpaliwal/nova-ai](https://github.com/samarthpaliwal/nova-ai) |
| 07 | [Credit card fraud detection](#07--credit-card-fraud-detection) | Real-time anomaly scoring tuned on the precision/recall trade-off | Python, scikit-learn, Pandas | [samarthpaliwal/CreditCardFraud](https://github.com/samarthpaliwal/CreditCardFraud) |

Portfolio site: [samarthpaliwal.github.io](https://samarthpaliwal.github.io/projects.html)

---

## 01 · Jarvis

**A voice assistant in the MacBook notch.** Menu bar app, no Dock icon. Wakes to "Hey Jarvis" using an on-device wake word model, transcribes speech on-device, and hands the thinking to Claude behind 29 tools. Every tool is gated by a permission tier that is enforced in the dispatcher, not in the prompt.

**Problem.** An assistant that can run shell commands, read email and change system settings is only useful if you can trust what it will *not* do. Prompt-level guardrails are not a safety model.

**Approach.** Swift and SwiftUI. openWakeWord on ONNX Runtime and Apple's SpeechAnalyzer, so audio never leaves the Mac. Every tool call passes through a dispatcher in this order: the denylist (paths, apps, shell substrings), the current mode's allow-list, then a Tier 2 approval card in the notch with a 12-second timeout. Content from files, email, web and calendar is wrapped in `<untrusted_content>` tags so Claude treats it as data, never instructions. Every call is logged as a JSON line you can ask about ("what did you do today?").

**Result.** Feature complete across six milestones: calendar, Gmail, Notion, morning briefings, automatic context modes (home, class, focus), proactive nudges, Mac maintenance, an undo stack, and an offline fallback to a local Ollama model. Free neural voice via Kokoro, with ElevenLabs and OpenAI voices optional.

| Tier | Behaviour | Examples |
|---|---|---|
| 0 | Automatic, read-only | `system_status`, `calendar_read`, `email_search`, `web_search` |
| 1 | Free, reversible, logged, undoable | `open_app`, `set_volume`, `calendar_create`, `email_draft` |
| 2 | Needs explicit approval | `run_shell`, `restart_app`, `clear_caches` |
| 3 | Never offered | denylist + hard-coded floor |

Full write-up: [jarvis/README.md](jarvis/README.md) · Code: [samarthpaliwal/Jarvis](https://github.com/samarthpaliwal/Jarvis)

---

## 02 · MacStream

**Low-latency screen streaming from a Mac to an iPad** over local Wi-Fi, with touch, Apple Pencil and keyboard control flowing back. Native Swift and SwiftUI on both sides, no third-party runtime dependencies.

**Problem.** Remote-desktop tools treat the iPad as a dumb display: latency in the 100 ms range, gestures that do not map to macOS, and a pairing story that is either insecure or painful.

**Approach.** ScreenCaptureKit feeds a hardware HEVC encoder through VideoToolbox. Frames are chunked over UDP with three layers of loss recovery, in order of preference: selective retransmission (the iPad NACKs exactly the missing chunks, the Mac keeps the last eight frames), drop-and-keyframe, and backpressure so a slow link adds loss rather than latency. An NTP-style clock sync over the ping channel gives true capture-to-display latency in the overlay. Adaptive bitrate cuts 20% on loss and steps back up after clean seconds. Pairing bootstraps a TLS pre-shared key from a six-digit code, then installs a 256-bit long-term key in both Keychains; unknown devices are dropped even if they somehow had the key.

**Result.**

| Preset | Size | End to end | Capture | Encode | Network | Decode |
|---|---|---|---|---|---|---|
| Balanced (75%) | 2268×1472 | 20 ms | 3 ms | 12.7 ms | 1.5 ms | 2.8 ms |
| Performance (50%) | 1512×982 | 12 ms | 4 ms | 7.5 ms | 0 ms | 2.7 ms |

Under 3% simulated packet loss, retransmission recovered 122 of 137 lost packets and held bitrate at 12 to 15 Mb/s instead of collapsing to the 3 Mb/s floor. Full gesture map: tap, drag, two-finger scroll, pinch, Pencil hover, three-finger Spaces and Mission Control swipes, hardware and on-screen keyboards with latchable modifiers.

Full write-up: [macstream/README.md](macstream/README.md) · Code: [samarthpaliwal/MacStream](https://github.com/samarthpaliwal/MacStream)

---

## 03 · Spotify Widget

**A Now Playing widget for the Spotify desktop app.** A circular album cover that spins at 33⅓ RPM, eases to a stop on pause, and sits on the macOS desktop-widget layer (above the wallpaper, below every window) with a background derived from each album's artwork. Lives in the menu bar. No login, no API keys, no OAuth.

**Problem.** Spotify's own window is the wrong shape for a glance. Third-party widgets want OAuth and API keys, and still miss the moment a track changes.

**Approach.** Electron and TypeScript, talking to Spotify through its AppleScript interface only. A long-lived JXA helper subscribes to Spotify's `PlaybackStateChanged` distributed notification for instant updates, backed by a sleep-aware poll (1 s playing, 5 s paused, 15 s hidden). Palette extraction is a dependency-free saturation-weighted histogram over the decoded JPEG. The spin is a single Web Animations API rotation whose `playbackRate` eases to zero on pause, so the angle is preserved and there is no per-frame JavaScript. The window snaps onto macOS's own desktop-widget grid at exact system widget sizes.

**Result.** Scratch the record to seek (one turn = 30 s, Spotify follows live), synced lyrics from LRCLIB, click or drag the progress bar, float-above-windows and click-through modes, launch at login. Zero runtime dependencies. Exactly two network requests: album art and lyrics.

Full write-up: [spotify-widget/README.md](spotify-widget/README.md) · Code: [samarthpaliwal/SpotifyWidget](https://github.com/samarthpaliwal/SpotifyWidget)

---

## 04 · Adaptive market-making simulator

A financial simulation engine that quotes both sides of a market while volatility shifts underneath it, built to test whether an adaptive spread beats a fixed one.

**Problem.** Market-making strategies look fine on calm data and blow up in stressed regimes. The harness needed to replay volatility spikes and measure PnL variance honestly.

**Approach.** Python asyncio so order flow, quoting and risk checks run as concurrent tasks. Order book lookups use a sorted structure for O(log n) access.

**Result.** 35.9% reduction in simulated PnL volatility across stress-tested market regimes compared with the fixed-spread baseline.

Code: [samarthpaliwal/adaptive-market-making-simulator](https://github.com/samarthpaliwal/adaptive-market-making-simulator)

---

## 05 · Event-processing platform

A distributed pipeline that ingests, validates and routes high-volume event streams, deployed on AWS with a CI/CD path from commit to container.

**Problem.** High-volume streams fail quietly: a malformed record slips through and corrupts everything downstream. Throughput and integrity usually fight each other.

**Approach.** Asyncio workers in Docker containers on AWS EC2 with S3 storage, custom schema validation layers at the ingest boundary, CI/CD building and shipping each change.

**Result.** Sustained 10,000 records per second with validation on, with data integrity held across concurrent streams.

Code: [samarthpaliwal/event-processing-platform](https://github.com/samarthpaliwal/event-processing-platform)

---

## 06 · RAG-based AI assistant

A retrieval-augmented generation assistant with an evaluation harness built in from the start, so every change to retrieval or prompting is measured rather than eyeballed.

**Problem.** RAG demos answer confidently and wrongly. Without a test set and a failure taxonomy you cannot tell whether a change helped.

**Approach.** LangChain pipeline over a vector database behind a REST API, with modular evaluation logic and structured failure-mode analysis across a fixed set of test queries.

**Result.** Measurably fewer hallucinated and irrelevant responses across the test query sets after each retrieval and prompt iteration.

Code: [samarthpaliwal/nova-ai](https://github.com/samarthpaliwal/nova-ai)

---

## 07 · Credit card fraud detection

An anomaly-detection service that scores transactions in real time, tuned for the precision and recall trade-off that fraud teams actually live with.

**Problem.** Fraud is rare, so accuracy is meaningless: a model that flags nothing scores 99%. The real target is catching fraud without burying analysts in false alarms.

**Approach.** Feature engineering over transaction history, a scikit-learn anomaly-detection model, precision/recall optimisation on held-out data, served through a REST API.

**Result.** Real-time scoring of high-risk transaction patterns, with thresholds chosen from the held-out precision/recall curve.

Code: [samarthpaliwal/CreditCardFraud](https://github.com/samarthpaliwal/CreditCardFraud)
