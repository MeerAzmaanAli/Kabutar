# 08 — BUILD PHASES
# Exact development order, one task at a time

> **Rule:** Never move to the next phase until the current one builds without errors
> and the feature works on an Android device/emulator.
>
> **Copilot tip:** At the start of each phase, say:
> "I am in Phase X: [Phase Name]. Attach: 00_MASTER_OVERVIEW.md + [relevant file]"

---

## PHASE 0 — PROJECT BOOTSTRAP
**Estimated time: 1-2 hours**
**Context files: 00_MASTER_OVERVIEW.md + 01_PROJECT_STRUCTURE.md**

- [ ] `npx react-native init PrivateChat --template react-native-template-typescript`
- [ ] Set `minSdkVersion 24` in `android/app/build.gradle`
- [ ] Enable `multiDexEnabled true`
- [ ] Install core dependencies first (get the app building before adding features):
  ```bash
  npm install @react-native-firebase/app @react-native-firebase/auth
  npm install @react-native-firebase/firestore @react-native-firebase/database
  npm install @react-native-firebase/messaging
  npm install react-native-reanimated react-native-gesture-handler
  npm install @gorhom/bottom-sheet@^5
  npm install zustand @tanstack/react-query
  npm install react-native-linear-gradient
  npm install @nozbe/watermelondb
  npm install tweetnacl tweetnacl-util
  npm install react-native-keychain
  npm install react-native-fs
  npm install axios
  ```
- [ ] Configure `babel.config.js` (reanimated plugin LAST)
- [ ] Configure `metro.config.js` for WatermelonDB
- [ ] Add `google-services.json` to `android/app/`
- [ ] Wrap `App.tsx` in `GestureHandlerRootView`
- [ ] Create `src/config/config.ts` with placeholder values
- [ ] Create `src/theme/themes.ts` (copy from UI files)
- [ ] Create `src/constants/index.ts` (copy from UI files)
- [ ] **VERIFY:** App builds and runs on Android emulator. No red screen.

---

## PHASE 1 — FIREBASE AUTH + ENCRYPTION SETUP
**Estimated time: 2-3 hours**
**Context files: 00_MASTER_OVERVIEW.md + 02_FIREBASE_SETUP.md + 04_ENCRYPTION.md**

- [ ] Create `src/firebase/firebaseApp.ts` (init, disable analytics)
- [ ] Create `src/firebase/authService.ts`:
  - `signIn(email, password)`
  - `signOut()`
  - `onAuthStateChanged(callback)`
- [ ] Create `src/store/authStore.ts` (Zustand)
- [ ] Create `LoginScreen.tsx`:
  - Simple email + password form
  - On submit: `authService.signIn()`
  - Show loading state
  - Show error on failure
- [ ] Create `src/navigation/AppNavigator.tsx`:
  - Listen to `onAuthStateChanged`
  - Show `LoginScreen` if no user
  - Show `ChatScreen` placeholder if authenticated
- [ ] Create `src/crypto/naclService.ts`:
  - `generateAndStoreKeypair()`
  - `encryptMessage()`
  - `decryptMessage()`
  - `getKeyFingerprint()`
- [ ] On first login: call `generateAndStoreKeypair()` if no key exists
- [ ] Upload public key to Firestore `users/{uid}.publicKey`
- [ ] **VERIFY:** Can sign in. Auth state persists on app restart. Keys generated once.

---

## PHASE 2 — WATERMELONDB + FIRESTORE SERVICE
**Estimated time: 2-3 hours**
**Context files: 00_MASTER_OVERVIEW.md + 02_FIREBASE_SETUP.md + 05_DATA_MODELS.md**

- [ ] Create `src/db/schema.ts` (messages, media_cache, link_previews tables)
- [ ] Create `src/db/database.ts` (WatermelonDB instance)
- [ ] Create `src/db/models/MessageModel.ts`
- [ ] Create `src/db/models/MediaCacheModel.ts`
- [ ] Create `src/db/models/LinkPreviewModel.ts`
- [ ] Create `src/firebase/firestoreService.ts`:
  - `sendMessage(payload)` → returns Firestore ID
  - `subscribeToMessages(onMessages, limit)` → returns unsubscribe fn
  - `markRead(messageId, userId)`
  - `deleteMessage(messageId)`
  - `reactToMessage(messageId, emoji, userId, action)`
  - `pinMessage(id)` / `unpinMessage(id)`
  - `subscribeToConversation(onChange)` → returns unsubscribe fn
  - `updateTheme(theme)`
  - `updateMood(userId, emoji, label)`
  - `subscribeToPartnerUser(partnerUid, onChange)`
  - `savePublicKey(userId, publicKey)`
  - `getPartnerPublicKey(partnerUid)`
- [ ] Create `src/utils/messageMapper.ts` (Firestore doc → Message type)
- [ ] Create `src/hooks/useMessages.ts`:
  - Firestore `subscribeToMessages` → decrypt → WatermelonDB upsert
  - Returns `messages` array from WatermelonDB `observe()`
  - Handles unsubscribe in cleanup
- [ ] **VERIFY:** Send a test message from Firebase Console → appears decrypted in app.

---

## PHASE 3 — BASIC CHAT SCREEN (TEXT ONLY)
**Estimated time: 3-4 hours**
**Context files: 00_MASTER_OVERVIEW.md + 01_PROJECT_STRUCTURE.md + 06_CHAT_FEATURES.md**

- [ ] Create `src/store/chatStore.ts` (Zustand)
- [ ] Create `src/store/uiStore.ts` (Zustand)
- [ ] Create `src/components/shared/Avatar.tsx`
- [ ] Create `src/components/shared/EncryptionBadge.tsx`
- [ ] Create `src/components/shared/TypingIndicator.tsx` (Reanimated 3-dot bounce)
- [ ] Create `src/components/shared/DateSeparator.tsx`
- [ ] Create `src/components/shared/SystemMessage.tsx`
- [ ] Create `src/components/chat/MessageBubble.tsx` (text only for now):
  - Sent: LinearGradient bubble (theme.sentGradient)
  - Received: flat bubble (theme.bubbleBg)
  - Jumbo emoji detection
  - Read receipt (✓ / ✓✓ / ✓✓ accent)
  - Timestamp below each group
- [ ] Create `src/components/input/InputBar.tsx` (text + send button only)
- [ ] Create `src/screens/ChatScreen.tsx`:
  - FlatList with `useMessages` hook
  - Message grouping logic (same sender < 60s)
  - `EncryptionBadge` below header
  - Basic header (name, back, call icons — non-functional)
  - InputBar at bottom
  - Keyboard avoiding (KeyboardAvoidingView)
  - Auto-scroll to bottom on new message
- [ ] Implement text message send:
  - `encryptMessage` → `firestoreService.sendMessage`
  - Optimistic insert before Firestore confirms
  - Confirm with firestoreId on response
- [ ] **VERIFY:** Two-way text messaging works. Messages persist on app restart.

---

## PHASE 4 — PRESENCE + TYPING + READ RECEIPTS
**Estimated time: 1-2 hours**
**Context files: 02_FIREBASE_SETUP.md + 06_CHAT_FEATURES.md**

- [ ] Create `src/firebase/presenceService.ts`:
  - `initPresence(uid)` → sets online, onDisconnect
  - `setTyping(uid, isTyping)`
  - `subscribeToPartnerPresence(partnerUid, onChange)`
- [ ] Create `src/hooks/usePresence.ts`
- [ ] Call `initPresence` after login
- [ ] Typing: debounced `setTyping` on TextInput.onChange
- [ ] `TypingIndicator` appears in FlatList footer when partner typing
- [ ] Header shows "Active now" or "Last seen X min ago"
- [ ] Implement read receipts:
  - FlatList `onViewableItemsChanged` → `markRead`
  - Tick states (✓ / ✓✓ grey / ✓✓ accent)
- [ ] **VERIFY:** Typing indicator visible. Read receipts update correctly.

---

## PHASE 5 — REACTIONS + LONG-PRESS ACTIONS
**Estimated time: 2-3 hours**
**Context files: 06_CHAT_FEATURES.md + 01_PROJECT_STRUCTURE.md**

- [ ] Install `rn-emoji-keyboard`
- [ ] Create `src/components/chat/ReactionPills.tsx`
- [ ] Create `src/components/sheets/MessageActionsSheet.tsx`:
  - Quick reaction bar (6 emoji from uiStore + "+" button)
  - Action list (Reply, Forward, Pin, Copy, Delete)
- [ ] Add long-press handler to MessageBubble (Animated scale on press)
- [ ] Add double-tap handler (TapGestureHandler numberOfTaps=2)
- [ ] Implement reactions:
  - `firestoreService.reactToMessage()`
  - ReactionPills render below bubble
  - Haptic feedback
- [ ] Implement delete (for me + for everyone with 10-min check)
- [ ] Implement copy (Clipboard)
- [ ] Create `src/components/sheets/EmojiSheet.tsx`
- [ ] Create `src/components/input/ReplyBar.tsx`
- [ ] Implement reply:
  - Swipe right (Swipeable) → setReplyTo
  - ReplyBar above InputBar
  - replyToId in Firestore message
  - Tap inset → scrollToIndex + highlight
- [ ] **VERIFY:** Long-press opens sheet. Reactions appear for both users. Reply works.

---

## PHASE 6 — THEMES + MOOD + SETTINGS
**Estimated time: 2-3 hours**
**Context files: 06_CHAT_FEATURES.md + 02_FIREBASE_SETUP.md**

- [ ] Create `src/components/sheets/ThemeSheet.tsx` (7 themes with preview)
- [ ] Create `src/components/sheets/MoodSheet.tsx` (7 moods)
- [ ] Create `src/components/sheets/SettingsSheet.tsx`
- [ ] Implement theme change:
  - `uiStore.setTheme()` → immediate visual update
  - `firestoreService.updateTheme()` → Firestore
  - `subscribeToConversation()` → partner's app updates
  - Reanimated color transition
- [ ] Implement mood:
  - Tap own avatar → MoodSheet
  - `firestoreService.updateMood()` → Firestore
  - `subscribeToPartnerUser()` → partner mood updates in header
  - Reanimated pulse on mood change
- [ ] Connect settings sheet to theme/mood actions
- [ ] **VERIFY:** Theme changes for both users in real-time. Mood shows in header.

---

## PHASE 7 — PINS + SEARCH + SCHEDULED MESSAGES
**Estimated time: 2-3 hours**
**Context files: 06_CHAT_FEATURES.md**

- [ ] Create `src/components/shared/PinnedBar.tsx`
- [ ] Implement pin/unpin (Firestore + UI)
- [ ] Create `src/screens/SearchScreen.tsx` (WatermelonDB FTS)
- [ ] Create `src/components/sheets/ScheduledSheet.tsx`
- [ ] Create `src/components/input/ScheduledChip.tsx`
- [ ] Implement scheduled messages:
  - Long-press send → DateTimePicker
  - Firestore message with sendAt + status='scheduled'
  - ScheduledChip in UI
  - Cancel scheduled message
- [ ] **VERIFY:** Pins visible on both devices. Search finds messages. Scheduled chip shows.

---

## PHASE 8 — VOICE MESSAGES
**Estimated time: 2-3 hours**
**Context files: 03_CLOUDINARY_SETUP.md + 04_ENCRYPTION.md + 06_CHAT_FEATURES.md**

- [ ] Install `react-native-audio-recorder-player`
- [ ] Create `src/components/input/RecordingBar.tsx`
- [ ] Create `src/components/chat/VoiceMessage.tsx` (waveform + scrub)
- [ ] Implement recording:
  - Hold mic → startRecorder
  - RecordingBar appears
  - Swipe left → cancel
  - Release / Send → stopRecorder
- [ ] Create `src/cloudinary/cloudinaryService.ts`
- [ ] Create `src/crypto/aesService.ts`
- [ ] Implement voice send:
  - AES encrypt → Cloudinary upload → Firestore message
- [ ] Implement voice receive + auto-download + cache
- [ ] **VERIFY:** Record, send, partner receives. Playback works with seek.

---

## PHASE 9 — IMAGE + VIDEO + FILE SHARING
**Estimated time: 3-4 hours**
**Context files: 03_CLOUDINARY_SETUP.md + 04_ENCRYPTION.md**

- [ ] Install: `react-native-vision-camera`, `react-native-image-crop-picker`,
      `react-native-video`, `react-native-document-picker`,
      `react-native-image-filter-kit`, `react-native-video-trim`,
      `@react-native-camera-roll/camera-roll`
- [ ] Create `src/components/sheets/AttachSheet.tsx`
- [ ] Create `src/screens/PhotoEditorScreen.tsx` (crop, filters, text, sticker, draw)
- [ ] Create `src/screens/VideoTrimmerScreen.tsx`
- [ ] Create `src/screens/MediaViewerScreen.tsx` (pinch-zoom, swipe dismiss)
- [ ] Create `src/hooks/useMediaUpload.ts` (encrypt → upload → progress)
- [ ] Implement image send pipeline (Section: MEDIA SEND FLOW in 03_CLOUDINARY_SETUP.md)
- [ ] Implement video send pipeline
- [ ] Implement file send pipeline
- [ ] Implement media receive + lazy decrypt on tap
- [ ] Save to gallery button in MediaViewerScreen
- [ ] **VERIFY:** Send photo. Partner receives and can view. Video plays. File downloads.

---

## PHASE 10 — GIFs + STICKERS + MESSAGE EFFECTS
**Estimated time: 2 hours**
**Context files: 06_CHAT_FEATURES.md**

- [ ] Install `react-native-fast-image`
- [ ] Create `src/components/sheets/GifSheet.tsx` (GIPHY trending + search)
- [ ] Create `src/components/sheets/StickerSheet.tsx` (6 packs)
- [ ] Create `src/components/sheets/EffectsSheet.tsx` (6 effects)
- [ ] Implement GIF send (public URL, no encryption)
- [ ] Implement sticker send
- [ ] Implement message effects:
  - ✨ button in InputBar when text is typed
  - Particle system overlay (Reanimated) on receive
  - AsyncStorage tracks played effects (don't replay on scroll)
- [ ] **VERIFY:** GIFs autoplay. Stickers render large. Effect plays once on receive.

---

## PHASE 11 — PUSH NOTIFICATIONS + BIOMETRICS
**Estimated time: 2-3 hours**
**Context files: 02_FIREBASE_SETUP.md**

- [ ] Install `notifee`, `react-native-biometrics`
- [ ] Create `src/firebase/messagingService.ts`
- [ ] Deploy Cloud Functions: `onNewMessage` (FCM data-only trigger)
- [ ] Implement foreground + background notification handling
- [ ] Implement biometric app lock (`useBiometricLock` hook)
- [ ] **VERIFY:** Notification appears when app in background. Biometric prompt on return.

---

## PHASE 12 — VOICE + VIDEO CALLS
**Estimated time: 4-5 hours**
**Context files: 07_CALLS_WEBRTC.md**

- [ ] Install `react-native-webrtc`, `react-native-incall-manager`, `react-native-callkeep`
- [ ] Create `src/hooks/useWebRTC.ts`
- [ ] Create `src/hooks/useIncomingCall.ts`
- [ ] Create `src/store/callStore.ts`
- [ ] Create `src/components/call/IncomingCallScreen.tsx`
- [ ] Create `src/components/call/ActiveCallScreen.tsx` (voice + video)
- [ ] Create `src/components/call/CallControls.tsx`
- [ ] Implement outgoing call flow
- [ ] Implement incoming call flow
- [ ] Implement call controls (mute, camera, flip, speaker, end)
- [ ] Implement video filters strip
- [ ] **VERIFY:** Voice call connects. Video call shows streams. Filters apply.

---

## PHASE 13 — POLISH + DEPLOY
**Estimated time: 2-3 hours**

- [ ] Deploy Firebase security rules (Firestore + Realtime DB)
- [ ] Deploy all Cloud Functions
- [ ] Set real UIDs in `config.ts`
- [ ] Set real Cloudinary credentials
- [ ] Set real GIPHY API key
- [ ] Test all 51 features end-to-end on two physical Android devices
- [ ] Build signed APK:
  ```bash
  cd android && ./gradlew assembleRelease
  ```
- [ ] Install APK on both devices

---

## KNOWN DEPENDENCY CONFLICTS TO WATCH

| Conflict | Resolution |
|---|---|
| `react-native-reanimated` + `@gorhom/bottom-sheet` | Both require gesture-handler — ensure only one version |
| WatermelonDB + Hermes | Add JSI support: `hermesEnabled: true` + watermelondb JSI config |
| `react-native-webrtc` minSdkVersion | Must be 24+ — set in build.gradle |
| `react-native-vision-camera` v4 API | Different from v3 — use v4 docs |
| Firebase + react-native 0.74 | Use `@react-native-firebase/*` v20+ |

---

## HOW TO GIVE CONTEXT TO COPILOT

**At the start of each session:**
```
"I'm working on [feature]. My stack is React Native 0.74 TypeScript with Firebase (Auth, Firestore, Realtime DB), Cloudinary for media, TweetNaCl for E2E encryption, WatermelonDB for local cache, and Zustand for state. I'm in Phase [X]. Here is my context: [attach relevant MD files]"
```

**When hitting a bug:**
```
"I have this error: [paste error]. I'm working in [filename]. 
Stack: [same as above]. Context: [attach 00_MASTER_OVERVIEW.md + relevant file]"
```

**When asking for a new component:**
```
"Create [ComponentName] following the patterns in my project. 
- All colors from `theme` prop (type Theme from src/theme/themes.ts)
- Use StyleSheet.create, no inline styles
- Use react-native-reanimated for animations
- Accept props: [list props]
Context: [00_MASTER_OVERVIEW.md + 05_DATA_MODELS.md]"
```
