# 02 — FIREBASE SETUP
# Auth, Firestore, Realtime DB, FCM, Security Rules, Cloud Functions

> **Firebase Storage is NOT used** — it was removed from the free Spark plan.
> All media files go to Cloudinary. See `03_CLOUDINARY_SETUP.md`.

---

## SERVICES USED (free Spark plan)

| Service | Purpose | Free Limits (plenty for 2 users) |
|---|---|---|
| Firebase Auth | Sign in / sign out | 10K authentications/month |
| Cloud Firestore | Encrypted messages, settings, mood, theme | 50K reads, 20K writes/day |
| Realtime Database | Online presence, typing indicator | 1GB storage, 10GB/month transfer |
| Cloud Messaging (FCM) | Metadata-only push notifications | Unlimited |
| Cloud Functions | FCM trigger, scheduled release, link preview | 2M invocations/month |

---

## FIREBASE INIT (`src/firebase/firebaseApp.ts`)

```typescript
import firebase from '@react-native-firebase/app';

// Firebase auto-initializes from google-services.json (Android)
// and GoogleService-Info.plist (iOS) — no manual init needed with RN Firebase

// Disable analytics on app start:
import analytics from '@react-native-firebase/analytics';
analytics().setAnalyticsCollectionEnabled(false);

export { firebase };
export { default as db }       from '@react-native-firebase/firestore';
export { default as rtdb }     from '@react-native-firebase/database';
export { default as auth }     from '@react-native-firebase/auth';
export { default as messaging } from '@react-native-firebase/messaging';
```

---

## FIRESTORE DATA MODEL

### Collection: `users/{userId}`
```
users/
  {USER_ME_UID}/
    username: string            // "Me"
    publicKey: string           // NaCl public key, base64
    fcmToken: string            // updated on login
    moodEmoji: string           // "🥰"
    moodLabel: string           // "Romantic"
    updatedAt: Timestamp

  {PARTNER_UID}/
    (same structure)
```

### Collection: `conversation/main/messages/{messageId}`
```
conversation/
  main/                         // single document
    theme: string               // "midnight"
    pinnedMessages: string[]    // array of message IDs, max 3
    quickReactions: string[]    // ["❤️","😂","😮","😢","😡","👍"]
    doubleTapReaction: string   // "❤️"
    updatedAt: Timestamp

  main/messages/{messageId}/
    senderId: string            // USER_ME_UID or PARTNER_UID
    type: string                // "text"|"image"|"video"|"voice"|"gif"|"sticker"|"file"|"system"
    
    // E2E encrypted payload (ALL message content is inside these two fields)
    ciphertext: string          // NaCl box output, base64
    nonce: string               // NaCl nonce, base64
    
    // Media (URLs only — actual files on Cloudinary, encrypted)
    mediaUrl: string | null     // Cloudinary secure_url
    mediaPublicId: string | null // Cloudinary public_id (for deletion)
    
    // Message metadata (NOT encrypted — only structural info)
    replyToId: string | null
    effectId: string | null     // "confetti"|"balloons"|etc
    isForwarded: boolean
    isDeleted: boolean
    isSystem: boolean
    isPinned: boolean
    
    // Delivery / read
    status: string              // "scheduled"|"sending"|"sent"|"delivered"|"read"
    readBy: map                 // { [userId]: Timestamp }
    
    // Scheduling
    sendAt: Timestamp | null    // null = immediate, future = scheduled
    
    // Reactions
    reactions: map              // { "❤️": { count: 2, userIds: ["uid1", "uid2"], type: "simple"|"plus", combined: string[] } }
    
    createdAt: Timestamp
    updatedAt: Timestamp
```

### Collection: `linkPreviews/{urlHash}`
```
linkPreviews/
  {md5ofUrl}/
    url: string
    title: string
    description: string
    imageUrl: string
    siteName: string
    favicon: string
    fetchedAt: Timestamp
```

### Realtime Database Structure
```json
{
  "presence": {
    "{USER_ME_UID}": {
      "online": true,
      "lastSeen": 1735000000000,
      "typing": false
    },
    "{PARTNER_UID}": {
      "online": false,
      "lastSeen": 1735000000000,
      "typing": false
    }
  },
  "calls": {
    "active": {
      "callerId": "{USER_ME_UID}",
      "callType": "video",
      "status": "ringing",
      "createdAt": 1735000000000
    },
    "signals": {
      "offer": { "sdp": "...", "type": "offer" },
      "answer": { "sdp": "...", "type": "answer" },
      "iceCandidates": {
        "{pushKey}": { "candidate": "...", "sdpMid": "...", "sdpMLineIndex": 0 }
      }
    }
  }
}
```

---

## FIRESTORE SECURITY RULES

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Helper: only our two known UIDs
    function isAuthorized() {
      return request.auth != null
          && request.auth.uid in [
               'USER_ME_UID',     // ← replace with real UID
               'PARTNER_UID'      // ← replace with real UID
             ];
    }

    // User profiles: read by both, write only own
    match /users/{userId} {
      allow read: if isAuthorized();
      allow write: if isAuthorized()
                  && request.auth.uid == userId;
    }

    // Conversation settings document
    match /conversation/main {
      allow read, write: if isAuthorized();
    }

    // Messages: both can read/create; only sender can delete
    match /conversation/main/messages/{messageId} {
      allow read:   if isAuthorized();
      allow create: if isAuthorized()
                    && request.resource.data.senderId == request.auth.uid;
      allow update: if isAuthorized();
      allow delete: if isAuthorized()
                    && resource.data.senderId == request.auth.uid;
    }

    // Link previews: read by both, write only by Cloud Functions
    match /linkPreviews/{previewId} {
      allow read:  if isAuthorized();
      allow write: if false; // Cloud Functions admin SDK bypasses this
    }

    // Deny everything else
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

## REALTIME DATABASE SECURITY RULES

```json
{
  "rules": {
    "presence": {
      "$uid": {
        ".read": "auth != null && (auth.uid === 'USER_ME_UID' || auth.uid === 'PARTNER_UID')",
        ".write": "auth != null && auth.uid === $uid"
      }
    },
    "calls": {
      ".read": "auth != null && (auth.uid === 'USER_ME_UID' || auth.uid === 'PARTNER_UID')",
      ".write": "auth != null && (auth.uid === 'USER_ME_UID' || auth.uid === 'PARTNER_UID')"
    }
  }
}
```

---

## FIRESTORE SERVICE (`src/firebase/firestoreService.ts`)

```typescript
import firestore, { FirebaseFirestoreTypes } from '@react-native-firebase/firestore';
import { CONFIG } from '../config/config';

const CONV = `conversation/${CONFIG.CONVERSATION_ID}`;
const MSGS = `${CONV}/messages`;

// ── SEND MESSAGE ──────────────────────────────────────────────────────────────
export const sendMessage = async (payload: {
  senderId: string;
  type: string;
  ciphertext: string;
  nonce: string;
  mediaUrl?: string;
  mediaPublicId?: string;
  replyToId?: string;
  effectId?: string;
  isForwarded?: boolean;
  sendAt?: Date;
}): Promise<string> => {
  const docRef = await firestore().collection(MSGS).add({
    ...payload,
    reactions: {},
    readBy: {},
    isDeleted: false,
    isSystem: false,
    isPinned: false,
    status: payload.sendAt ? 'scheduled' : 'sent',
    createdAt: firestore.FieldValue.serverTimestamp(),
    updatedAt: firestore.FieldValue.serverTimestamp(),
  });
  return docRef.id;
};

// ── SUBSCRIBE TO MESSAGES ─────────────────────────────────────────────────────
// Returns unsubscribe function — CALL in useEffect cleanup
export const subscribeToMessages = (
  onMessages: (msgs: FirebaseFirestoreTypes.QuerySnapshot) => void,
  limit = 50
): (() => void) => {
  return firestore()
    .collection(MSGS)
    .where('status', '!=', 'scheduled')
    .orderBy('status')
    .orderBy('createdAt', 'asc')
    .limitToLast(limit)
    .onSnapshot(onMessages);
};

// ── MARK READ ─────────────────────────────────────────────────────────────────
export const markRead = (messageId: string, userId: string) =>
  firestore().collection(MSGS).doc(messageId).update({
    [`readBy.${userId}`]: firestore.FieldValue.serverTimestamp(),
    status: 'read',
  });

// ── DELETE MESSAGE ────────────────────────────────────────────────────────────
export const deleteMessage = (messageId: string) =>
  firestore().collection(MSGS).doc(messageId).update({
    isDeleted: true,
    updatedAt: firestore.FieldValue.serverTimestamp(),
  });

// ── REACT TO MESSAGE ──────────────────────────────────────────────────────────
export const reactToMessage = (
  messageId: string,
  emoji: string,
  userId: string,
  action: 'add' | 'remove'
) => {
  const batch = firestore().batch();
  const ref = firestore().collection(MSGS).doc(messageId);
  if (action === 'add') {
    batch.update(ref, {
      [`reactions.${emoji}.count`]: firestore.FieldValue.increment(1),
      [`reactions.${emoji}.userIds`]: firestore.FieldValue.arrayUnion(userId),
    });
  } else {
    batch.update(ref, {
      [`reactions.${emoji}.count`]: firestore.FieldValue.increment(-1),
      [`reactions.${emoji}.userIds`]: firestore.FieldValue.arrayRemove(userId),
    });
  }
  return batch.commit();
};

// ── PIN MESSAGE ───────────────────────────────────────────────────────────────
export const pinMessage = (messageId: string) =>
  firestore().collection(MSGS).doc(messageId).update({ isPinned: true });

export const unpinMessage = (messageId: string) =>
  firestore().collection(MSGS).doc(messageId).update({ isPinned: false });

// ── CONVERSATION SETTINGS ─────────────────────────────────────────────────────
export const subscribeToConversation = (
  onChange: (data: any) => void
): (() => void) =>
  firestore().doc(CONV).onSnapshot(snap => onChange(snap.data()));

export const updateTheme = (theme: string) =>
  firestore().doc(CONV).update({ theme, updatedAt: firestore.FieldValue.serverTimestamp() });

export const updateQuickReactions = (reactions: string[]) =>
  firestore().doc(CONV).update({ quickReactions: reactions });

// ── MOOD ──────────────────────────────────────────────────────────────────────
export const updateMood = (userId: string, emoji: string, label: string) =>
  firestore().collection('users').doc(userId).update({ moodEmoji: emoji, moodLabel: label });

export const subscribeToPartnerUser = (
  partnerUid: string,
  onChange: (data: any) => void
): (() => void) =>
  firestore().collection('users').doc(partnerUid).onSnapshot(snap => onChange(snap.data()));

// ── PUBLIC KEY ────────────────────────────────────────────────────────────────
export const savePublicKey = (userId: string, publicKey: string) =>
  firestore().collection('users').doc(userId).set({ publicKey }, { merge: true });

export const getPartnerPublicKey = async (partnerUid: string): Promise<string> => {
  const snap = await firestore().collection('users').doc(partnerUid).get();
  return snap.data()?.publicKey ?? '';
};
```

---

## PRESENCE SERVICE (`src/firebase/presenceService.ts`)

```typescript
import database from '@react-native-firebase/database';
import { AppState } from 'react-native';
import { CONFIG } from '../config/config';

const presenceRef = (uid: string) => database().ref(`presence/${uid}`);

export const initPresence = (uid: string) => {
  const ref = presenceRef(uid);

  // Set online on connect
  ref.set({ online: true, lastSeen: Date.now(), typing: false });

  // Set offline on disconnect (server-side)
  ref.onDisconnect().update({ online: false, lastSeen: Date.now() });

  // AppState listener for background
  const sub = AppState.addEventListener('change', state => {
    if (state === 'background' || state === 'inactive') {
      ref.update({ online: false, lastSeen: Date.now() });
    } else if (state === 'active') {
      ref.update({ online: true });
    }
  });

  return () => sub.remove();
};

export const setTyping = (uid: string, isTyping: boolean) =>
  presenceRef(uid).update({ typing: isTyping });

export const subscribeToPartnerPresence = (
  partnerUid: string,
  onChange: (data: { online: boolean; lastSeen: number; typing: boolean }) => void
): (() => void) => {
  const ref = presenceRef(partnerUid);
  const handler = ref.on('value', snap => onChange(snap.val() ?? { online: false, lastSeen: 0, typing: false }));
  return () => ref.off('value', handler);
};
```

---

## CLOUD FUNCTIONS

### `functions/src/onNewMessage.ts`
```typescript
import * as functions from 'firebase-functions/v2';
import * as admin from 'firebase-admin';

// Triggers on every new message document in Firestore
export const onNewMessage = functions.firestore
  .onDocumentCreated('conversation/main/messages/{messageId}', async (event) => {
    const data = event.data?.data();
    if (!data) return;

    // Skip scheduled (not yet released), system messages
    if (data.status === 'scheduled' || data.isSystem) return;

    const recipientUid = data.senderId === 'USER_ME_UID'
      ? 'PARTNER_UID'
      : 'USER_ME_UID';

    const recipientDoc = await admin.firestore()
      .collection('users').doc(recipientUid).get();
    const fcmToken = recipientDoc.data()?.fcmToken;
    if (!fcmToken) return;

    // DATA-ONLY payload — ZERO message content sent to FCM
    await admin.messaging().send({
      token: fcmToken,
      data: {
        type: 'new_message',
        messageType: data.type,   // just "text", "image" etc — not content
        messageId: event.params.messageId,
      },
      android: { priority: 'high' },
      apns: { payload: { aps: { contentAvailable: true } } },
    });
  });
```

### `functions/src/releaseScheduled.ts`
```typescript
import * as functions from 'firebase-functions/v2/scheduler';
import * as admin from 'firebase-admin';

export const releaseScheduledMessages = functions.onSchedule(
  'every 1 minutes',
  async () => {
    const now = admin.firestore.Timestamp.now();
    const snap = await admin.firestore()
      .collection('conversation/main/messages')
      .where('status', '==', 'scheduled')
      .where('sendAt', '<=', now)
      .get();

    const batch = admin.firestore().batch();
    snap.docs.forEach(doc => {
      batch.update(doc.ref, { status: 'sent', updatedAt: now });
    });
    await batch.commit();
  }
);
```

### `functions/src/getLinkPreview.ts`
```typescript
import * as functions from 'firebase-functions/v2/https';
import * as admin from 'firebase-admin';
import fetch from 'node-fetch';
import * as cheerio from 'cheerio';
import * as crypto from 'crypto';

export const getLinkPreview = functions.onCall(async (request) => {
  const { url } = request.data as { url: string };
  const urlHash = crypto.createHash('md5').update(url).digest('hex');

  // Check cache first
  const cached = await admin.firestore().collection('linkPreviews').doc(urlHash).get();
  if (cached.exists) return cached.data();

  try {
    const res = await fetch(url, { signal: AbortSignal.timeout(3000) });
    const html = await res.text();
    const $ = cheerio.load(html);

    const preview = {
      url,
      title: $('meta[property="og:title"]').attr('content') || $('title').text() || '',
      description: $('meta[property="og:description"]').attr('content') || '',
      imageUrl: $('meta[property="og:image"]').attr('content') || '',
      siteName: $('meta[property="og:site_name"]').attr('content') || new URL(url).hostname,
      favicon: `https://www.google.com/s2/favicons?domain=${new URL(url).hostname}`,
      fetchedAt: admin.firestore.FieldValue.serverTimestamp(),
    };

    await admin.firestore().collection('linkPreviews').doc(urlHash).set(preview);
    return preview;
  } catch {
    return null;
  }
});
```

---

## FCM SETUP ON MOBILE (`src/firebase/messagingService.ts`)

```typescript
import messaging from '@react-native-firebase/messaging';
import notifee, { AndroidImportance } from '@notifee/react-native';
import { firestoreService } from './firestoreService';
import { naclService } from '../crypto/naclService';

export const initMessaging = async (userId: string) => {
  // Request permission (iOS)
  await messaging().requestPermission();

  // Get + save FCM token
  const token = await messaging().getToken();
  await firestoreService.saveFcmToken(userId, token);

  // Token refresh
  messaging().onTokenRefresh(newToken => {
    firestoreService.saveFcmToken(userId, newToken);
  });

  // Create notification channels
  await notifee.createChannel({
    id: 'messages',
    name: 'Messages',
    importance: AndroidImportance.HIGH,
    sound: 'default',
  });
  await notifee.createChannel({
    id: 'calls',
    name: 'Calls',
    importance: AndroidImportance.HIGH,
    sound: 'ringtone',
    vibration: true,
  });

  // Foreground message handler
  messaging().onMessage(async remoteMessage => {
    if (remoteMessage.data?.type === 'new_message') {
      // Message will be delivered by Firestore onSnapshot
      // Only show notification if app is NOT on ChatScreen
      // (navigation state check needed)
    }
  });
};

// Background handler (set in index.js, outside component)
messaging().setBackgroundMessageHandler(async remoteMessage => {
  // For data-only messages, construct notification locally
  // Actual decryption happens when app opens
  await notifee.displayNotification({
    title: 'Love 💖',
    body: 'New message',
    android: { channelId: 'messages', smallIcon: 'ic_notification' },
  });
});
```
