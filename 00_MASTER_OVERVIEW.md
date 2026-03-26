# 00 — MASTER OVERVIEW
# Private Two-Person Encrypted Chat App

> **Always attach this file to every Copilot session.**
> It is the single source of truth for decisions, stack, and constraints.

---

## WHAT WE ARE BUILDING

A private, self-contained chat app for exactly **2 users** (me + my partner).
No public registration. No strangers. No social graph.
Think: Instagram DMs, but end-to-end encrypted and fully owned by us.

---

## FINAL TECH STACK DECISIONS (locked, do not change)

| Layer | Technology | Why |
|---|---|---|
| Mobile | React Native 0.74+ TypeScript | Cross-platform (Android + iOS) |
| UI | Custom StyleSheet + react-native-paper | No NativeWind |
| Navigation | React Navigation v6 Native Stack | Standard RN nav |
| State | Zustand + React Query v5 | Simple, no Redux overhead |
| Auth | Firebase Auth (email/password) | Free tier, 2 users only |
| Database | Cloud Firestore | Real-time, free tier |
| Presence/Typing | Firebase Realtime Database | Low-latency, free tier |
| Media Storage | **Cloudinary** (NOT Firebase Storage — removed from free plan) | 25GB free |
| Local Cache | WatermelonDB | Offline, FTS, high-perf |
| E2E Encryption | TweetNaCl (messages) + AES-256-GCM (media) | True E2E |
| Key Storage | react-native-keychain | Android Keystore / iOS Secure Enclave |
| Animations | react-native-reanimated v3 + gesture-handler | Smooth, native |
| Bottom Sheets | @gorhom/bottom-sheet v5 | Performant sheets |
| Camera | react-native-vision-camera v4 | Modern, maintained |
| Video Playback | react-native-video v6 | Inline player |
| Voice | react-native-audio-recorder-player | Record + playback |
| Calls | react-native-webrtc | P2P voice/video |
| Push | @react-native-firebase/messaging + notifee | FCM metadata-only |
| Haptics | react-native-haptic-feedback | |
| File System | react-native-fs | |
| Gallery | @react-native-camera-roll/camera-roll | |
| Image Editing | react-native-image-crop-picker + react-native-image-filter-kit | |
| Video Trim | react-native-video-trim | |
| GIFs | react-native-fast-image | Autoplay |
| Emoji Picker | rn-emoji-keyboard | |
| File Picker | react-native-document-picker | |
| Biometrics | react-native-biometrics | App lock |
| In-call Audio | react-native-incall-manager | |
| Call UI | react-native-callkeep | Native call UI |
| Clipboard | @react-native-clipboard/clipboard | |
| Date Picker | @react-native-community/datetimepicker | Scheduled messages |
| SVG | react-native-svg | Waveform draw |
| HTTP | Axios | Cloudinary uploads, GIPHY |

---

## ARCHITECTURE OVERVIEW

```
┌─────────────────────────────────────────────────────┐
│                  REACT NATIVE APP                   │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐ │
│  │  Screens │  │Components│  │  Zustand Stores   │ │
│  └────┬─────┘  └────┬─────┘  └────────┬──────────┘ │
│       │             │                 │             │
│  ┌────▼─────────────▼─────────────────▼──────────┐ │
│  │              Service Layer                    │ │
│  │  firestoreService  cloudinaryService          │ │
│  │  presenceService   encryptionService          │ │
│  │  notificationService  webrtcService           │ │
│  └──────┬─────────────────────────┬──────────────┘ │
│         │                         │                 │
│  ┌──────▼───────┐    ┌────────────▼──────────────┐ │
│  │ WatermelonDB │    │  react-native-keychain    │ │
│  │ (local cache)│    │  (NaCl private key)       │ │
│  └──────────────┘    └───────────────────────────┘ │
└───────────┬──────────────────────┬─────────────────┘
            │                      │
   ┌────────▼──────────┐  ┌───────▼────────────────┐
   │     FIREBASE      │  │       CLOUDINARY        │
   │  Auth             │  │  Raw encrypted blobs    │
   │  Firestore        │  │  (AES-256-GCM files)    │
   │  Realtime DB      │  │  /media/{messageId}/    │
   │  FCM              │  └────────────────────────┘
   └───────────────────┘
```

---

## TWO USERS — HARDCODED UIDs

After creating accounts in Firebase Auth:
```typescript
// src/constants/config.ts
export const CONFIG = {
  USER_ME_UID: 'REPLACE_WITH_REAL_UID',        // Your Firebase UID
  PARTNER_UID: 'REPLACE_WITH_REAL_UID',        // Partner Firebase UID
  CONVERSATION_ID: 'main',                      // Single conversation
  CLOUDINARY_CLOUD_NAME: 'REPLACE',
  CLOUDINARY_UPLOAD_PRESET: 'REPLACE',          // unsigned preset
  GIPHY_API_KEY: 'REPLACE',
};
```

---

## CRITICAL RULES (never violate these)

1. **Server never sees plaintext** — every message NaCl-encrypted before writing to Firestore
2. **Media never stored unencrypted** — AES-256-GCM encrypt before Cloudinary upload
3. **Private key never leaves device** — stored only in react-native-keychain
4. **Firebase Storage = NOT USED** — removed from free plan, use Cloudinary only
5. **No third-party chat SDKs** — no Stream, no Sendbird, no Pusher
6. **No analytics** — `automaticDataCollectionEnabled: false` in Firebase init
7. **Only 2 known UIDs can access data** — enforced in Firestore security rules
8. **TypeScript strict mode** — no `any`, no `@ts-ignore`
9. **Reanimated for all animations** — no setTimeout-based UI animations
10. **All async = async/await** — no callbacks, no RxJava patterns

---

## FEATURES LIST (51 confirmed)

### Messaging
- Plain text, Photos, Videos, Voice (5min), GIFs, Static stickers
- Emoji-only jumbo (1-3 emoji = 52px, no bubble)
- Link rich previews (Cloudinary function or simple OG fetch)
- File/Document sharing
- Animated message effects (Confetti, Balloons, Fireworks, Hearts, Snow, Fire)
- Scheduled messages (Firestore + Cloud Function cron)
- Message forwarding (re-send with ↪ Forwarded label)
- Reply/Quote (swipe right gesture)

### Interactions
- Long-press → Reaction bar + actions (Reply, Forward, Pin, Copy, Delete)
- Double-tap → Quick ❤️ react
- React with any emoji (full picker)
- Reactions Plus (combine up to 4 emoji)
- Delete for me / Delete for everyone (10 min window)
- Pin up to 3 messages per conversation
- Copy text to clipboard
- Search messages (local WatermelonDB FTS)

### Media
- Camera shortcut in input bar
- Gallery multi-select (up to 10)
- Photo editor (crop, filters, text overlay, sticker, draw)
- Video trimmer (max 2 min output)
- Fullscreen media viewer (pinch-to-zoom, swipe dismiss)
- Save received media to gallery
- Voice message waveform visualizer + scrub/seek

### Privacy
- E2E encryption badge permanently visible
- Mute partner notifications (1h/8h/24h/forever)
- App lock via biometrics (after 2min background)

### Calls
- Voice call (WebRTC P2P)
- Video call (WebRTC P2P, draggable PiP self-view)
- Mute mic, toggle camera, flip camera, speaker toggle
- Video call filters (Normal, Warm, Cool, B&W, Glow, Vintage)

### Personalization
- Mood emoji (7 moods) shown next to partner name in header
- Chat themes (7: Midnight, Sunset, Ocean, Forest, Rose, Mono, Galaxy)
- Both users see same theme in real-time (Firestore sync)
- Default 6-emoji quick reaction bar (customizable)
- Double-tap reaction customizable

### Notifications
- Push notifications (FCM data-only, decrypt on device)
- Typing indicator (Realtime DB)
- Read receipts (✓ sent / ✓✓ delivered / ✓✓ accent = read)
- Online/Active status

---

## CONTEXT FILES INDEX

When working on a specific area, give Copilot the relevant file:

| Working on... | Attach this file |
|---|---|
| Project setup / structure | `01_PROJECT_STRUCTURE.md` |
| Firebase config / security rules | `02_FIREBASE_SETUP.md` |
| Cloudinary media upload/download | `03_CLOUDINARY_SETUP.md` |
| E2E encryption logic | `04_ENCRYPTION.md` |
| Data models / types | `05_DATA_MODELS.md` |
| Chat screen / messages | `06_CHAT_FEATURES.md` |
| Call screen / WebRTC | `07_CALLS_WEBRTC.md` |
| Build phases / what to do next | `08_BUILD_PHASES.md` |
| Any bug / integration issue | `00_MASTER_OVERVIEW.md` + relevant file |

---

## KNOWN PITFALLS (lessons learned)

- `react-native-reanimated` plugin MUST be last in `babel.config.js`
- `@gorhom/bottom-sheet` requires `GestureHandlerRootView` wrapping the entire app
- WatermelonDB schema changes require `addColumns` migrations, not schema rewrites
- Firestore `onSnapshot` unsubscribers MUST be called in `useEffect` cleanup
- NaCl keys must be generated ONCE and persisted — never regenerate on login
- Cloudinary unsigned uploads need a preset with `resource_type: raw` for encrypted blobs
- `react-native-webrtc` requires `minSdkVersion 24` on Android
- FCM `onMessage` only fires when app is in foreground — use `setBackgroundMessageHandler` for background
- WatermelonDB `observe()` must be used with `withObservables` HOC or `useObservable` hook
