# 01 — PROJECT STRUCTURE
# Every file, its purpose, and its dependencies

---

## FULL DIRECTORY TREE

```
private-chat/
│
├── android/                          # Auto-generated, minimal manual edits
│   └── app/
│       ├── src/main/AndroidManifest.xml   # Permissions added here
│       └── build.gradle              # minSdkVersion 24, multiDex
│
├── ios/                              # Auto-generated
│
├── src/
│   │
│   ├── config/
│   │   └── config.ts                 # ← UIDs, API keys, constants (NEVER commit secrets)
│   │
│   ├── firebase/
│   │   ├── firebaseApp.ts            # Firebase init (analytics disabled)
│   │   ├── firestoreService.ts       # All Firestore CRUD + listeners
│   │   ├── presenceService.ts        # Realtime DB: online status + typing
│   │   ├── authService.ts            # Sign in, sign out, current user
│   │   └── messagingService.ts       # FCM token registration
│   │
│   ├── cloudinary/
│   │   └── cloudinaryService.ts      # Upload encrypted blobs, get download URLs
│   │
│   ├── crypto/
│   │   ├── naclService.ts            # NaCl keypair gen, box encrypt/decrypt
│   │   └── aesService.ts             # AES-256-GCM media encrypt/decrypt
│   │
│   ├── db/                           # WatermelonDB (local cache)
│   │   ├── database.ts               # DB instance
│   │   ├── schema.ts                 # Table definitions + migrations
│   │   └── models/
│   │       ├── MessageModel.ts       # Local message row
│   │       ├── MediaCacheModel.ts    # Cached decrypted media paths
│   │       └── LinkPreviewModel.ts   # Cached OG preview data
│   │
│   ├── store/                        # Zustand global state
│   │   ├── authStore.ts              # currentUser, partnerUser, isLoading
│   │   ├── chatStore.ts              # messages, typing, replyTo, selectedMsg, effect
│   │   ├── callStore.ts              # callType, callStatus, localStream, remoteStream
│   │   └── uiStore.ts                # themeId, myMood, partnerMood, quickReactions
│   │
│   ├── hooks/
│   │   ├── useMessages.ts            # Firestore onSnapshot → WatermelonDB sync
│   │   ├── usePresence.ts            # Realtime DB online/typing
│   │   ├── useWebRTC.ts              # WebRTC peer connection lifecycle
│   │   ├── useMediaUpload.ts         # Encrypt → Cloudinary upload → progress
│   │   ├── useEncryption.ts          # Wrap nacl/aes services, load keys
│   │   └── useBiometricLock.ts       # AppState listener + BiometricPrompt
│   │
│   ├── navigation/
│   │   ├── AppNavigator.tsx          # Root: AuthStack or MainStack based on auth
│   │   ├── AuthStack.tsx             # LoginScreen only
│   │   └── MainStack.tsx             # ChatScreen, MediaViewer, PhotoEditor, VideoTrimmer, SearchScreen
│   │
│   ├── screens/
│   │   ├── LoginScreen.tsx           # Firebase Auth sign-in form
│   │   ├── ChatScreen.tsx            # Main chat (FlatList + all sheets)
│   │   ├── SearchScreen.tsx          # Full-text search over WatermelonDB
│   │   ├── MediaViewerScreen.tsx     # Fullscreen image/video + pinch-zoom
│   │   ├── PhotoEditorScreen.tsx     # Crop, filter, text, sticker, draw
│   │   └── VideoTrimmerScreen.tsx    # Trim video before send
│   │
│   ├── components/
│   │   │
│   │   ├── shared/
│   │   │   ├── Avatar.tsx            # Gradient avatar + online dot + ring
│   │   │   ├── EncryptionBadge.tsx   # 🔒 E2E badge row
│   │   │   ├── TypingIndicator.tsx   # 3-dot animated bounce
│   │   │   ├── PinnedBar.tsx         # Cycling pinned messages bar
│   │   │   ├── SystemMessage.tsx     # Centered italic messages
│   │   │   └── DateSeparator.tsx     # "Today", "Yesterday", date string
│   │   │
│   │   ├── chat/
│   │   │   ├── MessageBubble.tsx     # Master bubble: all types + reactions + reply ctx
│   │   │   ├── VoiceMessage.tsx      # Waveform + PanResponder scrub + audio
│   │   │   ├── MediaMessage.tsx      # Image/Video bubble with tap handler
│   │   │   ├── FileMessage.tsx       # File icon + name + size + download
│   │   │   ├── GifMessage.tsx        # Animated GIF via react-native-fast-image
│   │   │   ├── LinkPreviewCard.tsx   # OG preview card below text
│   │   │   └── ReactionPills.tsx     # Reaction badges below bubble
│   │   │
│   │   ├── input/
│   │   │   ├── InputBar.tsx          # Main input row (text field + buttons + send)
│   │   │   ├── RecordingBar.tsx      # Recording state bar (replaces input)
│   │   │   ├── ReplyBar.tsx          # Reply-to preview above input
│   │   │   └── ScheduledChip.tsx     # "Scheduled (N)" chip above input
│   │   │
│   │   ├── sheets/                   # All @gorhom/bottom-sheet panels
│   │   │   ├── EmojiSheet.tsx
│   │   │   ├── GifSheet.tsx
│   │   │   ├── StickerSheet.tsx
│   │   │   ├── AttachSheet.tsx
│   │   │   ├── EffectsSheet.tsx
│   │   │   ├── MessageActionsSheet.tsx
│   │   │   ├── MoodSheet.tsx
│   │   │   ├── ThemeSheet.tsx
│   │   │   ├── SettingsSheet.tsx
│   │   │   └── ScheduledSheet.tsx
│   │   │
│   │   └── call/
│   │       ├── IncomingCallScreen.tsx
│   │       ├── ActiveCallScreen.tsx
│   │       └── CallControls.tsx
│   │
│   ├── theme/
│   │   └── themes.ts                 # Theme interface + 7 theme objects
│   │
│   ├── constants/
│   │   ├── index.ts                  # Moods, stickers, effects, emojis, sample data
│   │   └── messages.ts               # Message type definitions (also in models)
│   │
│   └── utils/
│       ├── formatters.ts             # fmtTime, fmtDuration, fmtFileSize
│       ├── linkDetector.ts           # URL regex, OG fetch
│       ├── permissions.ts            # Camera, mic, storage permission helpers
│       └── emojiUtils.ts             # isEmojiOnly detection regex
│
├── functions/                        # Firebase Cloud Functions (TypeScript)
│   ├── src/
│   │   ├── index.ts                  # Export all functions
│   │   ├── onNewMessage.ts           # Firestore trigger → FCM metadata push
│   │   ├── releaseScheduled.ts       # Scheduler: every 1 min → release due messages
│   │   └── getLinkPreview.ts         # Callable: OG scraping with cheerio
│   ├── package.json
│   └── tsconfig.json
│
├── App.tsx                           # GestureHandlerRootView → AppNavigator
├── index.js                          # AppRegistry
├── app.json
├── babel.config.js                   # reanimated plugin LAST
├── metro.config.js
├── tsconfig.json                     # strict: true
├── google-services.json              # Android Firebase config (gitignored)
├── GoogleService-Info.plist          # iOS Firebase config (gitignored)
└── .env                              # Cloudinary + GIPHY keys (gitignored)
```

---

## KEY FILE RESPONSIBILITIES

### `src/config/config.ts`
```typescript
export const CONFIG = {
  USER_ME_UID: '',           // Set after Firebase Auth account creation
  PARTNER_UID: '',           // Set after Firebase Auth account creation
  CONVERSATION_ID: 'main',
  CLOUDINARY_CLOUD_NAME: '',
  CLOUDINARY_UPLOAD_PRESET: '', // unsigned preset, resource_type: raw
  GIPHY_API_KEY: '',
} as const;
```

### `App.tsx`
```typescript
// MUST have GestureHandlerRootView as the outermost wrapper
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { AppNavigator } from './src/navigation/AppNavigator';

export default function App() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <AppNavigator />
    </GestureHandlerRootView>
  );
}
```

### `babel.config.js`
```javascript
// reanimated plugin MUST be the last plugin
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: [
    ['module:react-native-dotenv'],
    'react-native-reanimated/plugin',  // ← ALWAYS LAST
  ],
};
```

### `src/navigation/AppNavigator.tsx`
```
Listens to Firebase Auth state:
  - No user → AuthStack (LoginScreen)
  - User authenticated → MainStack (ChatScreen as root)

Does NOT check UIDs here — Firestore security rules enforce access.
```

### `src/firebase/firebaseApp.ts`
```
Single Firebase app init.
Sets automaticDataCollectionEnabled: false.
Exports: app, db (Firestore), rtdb (Realtime DB), auth, storage reference.
All other services import from here — never call firebase.initializeApp() more than once.
```

---

## DATA FLOW: Message Send

```
User types → InputBar
  → onSendText(text)
  → chatStore.addOptimisticMessage(tempId, text)  [immediately visible]
  → encryptionService.encryptMessage(text)         [NaCl box]
  → firestoreService.sendMessage({ciphertext, nonce, type, ...})
  → Firestore writes to /conversation/main/messages/{id}
  → Cloud Function onNewMessage triggers
     → sends FCM data-only to partner
  → Partner's onSnapshot fires
     → encryptionService.decryptMessage(ciphertext, nonce)
     → chatStore.upsertMessage(decryptedMsg)
     → WatermelonDB.upsert(msg)                   [local cache]
  → Sender's onSnapshot fires with server ID
     → chatStore.confirmMessage(tempId, serverId)  [status: sent]
```

## DATA FLOW: Media Send

```
User picks/captures media
  → PhotoEditorScreen or VideoTrimmerScreen
  → aesService.encryptFile(fileBuffer)   → {encryptedBuffer, key, iv}
  → cloudinaryService.uploadRaw(encryptedBuffer, messageId)  → {secureUrl}
  → nacl.encryptMessage({ type, mediaUrl: secureUrl, encryptedKey: key+iv })
  → firestoreService.sendMessage({ciphertext, nonce, type: 'image'})
  
Partner receives:
  → decryptMessage → gets { mediaUrl, encryptedKey }
  → cloudinaryService.downloadRaw(mediaUrl)  → encryptedBuffer
  → aesService.decryptFile(encryptedBuffer, encryptedKey)  → plainBuffer
  → save to temp dir → display
```

---

## COMPONENT PROP CONTRACTS

### All components accept `theme: Theme`
Never use hardcoded color strings. Always: `{ color: theme.accent }`.

### All sheets accept `sheetRef: React.RefObject<BottomSheet>`
Open: `sheetRef.current?.expand()`
Close: `sheetRef.current?.close()`

### MessageBubble props
```typescript
interface MessageBubbleProps {
  message: Message;
  isGrouped: boolean;     // same sender, <60s apart
  theme: Theme;
  onLongPress: (msg: Message) => void;
  onDoubleTap: (msg: Message) => void;
  onScrollToReply: (id: string) => void;
  onMediaPress: (msg: Message) => void;
}
```

### ChatScreen sheet refs (all defined here, passed down as needed)
```typescript
// 10 refs, one per sheet
const emojiSheetRef     = useRef<BottomSheet>(null);
const gifSheetRef       = useRef<BottomSheet>(null);
const stickerSheetRef   = useRef<BottomSheet>(null);
const attachSheetRef    = useRef<BottomSheet>(null);
const effectsSheetRef   = useRef<BottomSheet>(null);
const actionsSheetRef   = useRef<BottomSheet>(null);
const moodSheetRef      = useRef<BottomSheet>(null);
const themeSheetRef     = useRef<BottomSheet>(null);
const settingsSheetRef  = useRef<BottomSheet>(null);
const scheduledSheetRef = useRef<BottomSheet>(null);
```

---

## NAMING CONVENTIONS

| Type | Convention | Example |
|---|---|---|
| Components | PascalCase.tsx | `MessageBubble.tsx` |
| Hooks | camelCase, `use` prefix | `useMessages.ts` |
| Services | camelCase, `Service` suffix | `firestoreService.ts` |
| Stores | camelCase, `Store` suffix | `chatStore.ts` |
| Types/Interfaces | PascalCase | `Message`, `Theme` |
| Constants | SCREAMING_SNAKE | `CONFIG`, `THEMES` |
| Utils | camelCase | `formatters.ts` |

---

## ANDROID MANIFEST PERMISSIONS REQUIRED

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="29" />
<uses-permission android:name="android.permission.USE_BIOMETRIC" />
<uses-permission android:name="android.permission.USE_FINGERPRINT" />
<uses-permission android:name="android.permission.VIBRATE" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

## ANDROID build.gradle (app level)
```gradle
android {
  defaultConfig {
    minSdkVersion 24        // required by react-native-webrtc
    targetSdkVersion 34
    multiDexEnabled true
  }
}
```
