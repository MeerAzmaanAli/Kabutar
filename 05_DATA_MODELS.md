# 05 — DATA MODELS
# TypeScript types, WatermelonDB schema, Firestore document shapes

---

## CORE TYPES (`src/constants/messages.ts`)

```typescript
// ── MESSAGE ───────────────────────────────────────────────────────────────────

export type MessageType =
  | 'text'
  | 'image'
  | 'video'
  | 'voice'
  | 'gif'
  | 'sticker'
  | 'file'
  | 'system';

export type MessageStatus =
  | 'pending'     // optimistic, not yet sent to Firestore
  | 'scheduled'   // scheduled for future send
  | 'sending'     // being uploaded (media messages)
  | 'sent'        // written to Firestore
  | 'delivered'   // partner's device received it (onSnapshot fired)
  | 'read';       // partner opened and viewed

export interface Reaction {
  count: number;
  userIds: string[];
  type: 'simple' | 'plus';
  combined: string[];  // for Reactions Plus: ["❤️","😂","🔥"]
}

export interface Message {
  // Identity
  id: string;                     // Firestore doc ID (or temp UUID for pending)
  tempId?: string;                // client-side temp ID before Firestore confirms
  sender: 'me' | 'partner';       // derived from senderId vs CONFIG.USER_ME_UID
  senderId: string;               // actual Firebase UID

  // Content (all decrypted, ready to display)
  type: MessageType;
  text?: string;                  // for text, sticker (emoji string), system messages

  // Media (populated after decryption)
  mediaUrl?: string;              // Cloudinary URL (for display placeholder)
  mediaLocalPath?: string;        // local decrypted temp path (after download+decrypt)
  mediaKey?: string;              // AES key (hex) — from decrypted NaCl payload
  mediaIv?: string;               // AES IV (hex) — from decrypted NaCl payload
  mediaPublicId?: string;         // Cloudinary public_id
  duration?: number;              // voice/video seconds
  fileName?: string;              // file messages
  fileSize?: number;              // bytes
  mimeType?: string;

  // GIF
  gifUrl?: string;                // GIPHY CDN URL (public, not encrypted)

  // Structure
  replyTo?: Message;              // full message object for display
  replyToId?: string;             // Firestore ref
  isForwarded?: boolean;
  effectId?: string;              // "confetti"|"balloons"|"fireworks"|"hearts"|"snow"|"fire"

  // State
  reactions: Record<string, Reaction>;
  status: MessageStatus;
  isDeleted: boolean;
  isSystem: boolean;
  isPinned: boolean;
  readBy: Record<string, Date>;

  // Timestamps
  timestamp: Date;
  sendAt?: Date;                  // for scheduled messages
}

// ── USER ──────────────────────────────────────────────────────────────────────

export interface User {
  uid: string;
  username: string;
  publicKey: string;              // NaCl public key, base64
  moodEmoji: string | null;
  moodLabel: string | null;
  fcmToken: string | null;
  updatedAt: Date;
}

// ── CONVERSATION SETTINGS ─────────────────────────────────────────────────────

export interface ConversationSettings {
  theme: string;                  // theme ID: "midnight" | "sunset" | etc.
  pinnedMessages: string[];       // array of message IDs, max 3
  quickReactions: string[];       // ["❤️","😂","😮","😢","😡","👍"]
  doubleTapReaction: string;      // "❤️"
}

// ── CALL ──────────────────────────────────────────────────────────────────────

export type CallType = 'voice' | 'video';
export type CallStatus = 'idle' | 'ringing' | 'active' | 'ended';

export interface CallState {
  type: CallType | null;
  status: CallStatus;
  direction: 'incoming' | 'outgoing' | null;
  duration: number;               // seconds
  isMuted: boolean;
  isCameraOff: boolean;
  isSpeakerOn: boolean;
  activeFilter: string;           // "Normal" | "Warm" | etc.
}

// ── LINK PREVIEW ──────────────────────────────────────────────────────────────

export interface LinkPreview {
  url: string;
  title: string;
  description: string;
  imageUrl: string;
  siteName: string;
  favicon: string;
}

// ── SCHEDULED MESSAGE ────────────────────────────────────────────────────────

export interface ScheduledMessage {
  id: string;
  text: string;                   // displayed in scheduled list
  sendAt: Date;
  type: MessageType;
}
```

---

## WATERMELONDB SCHEMA (`src/db/schema.ts`)

```typescript
import { appSchema, tableSchema } from '@nozbe/watermelondb';

export const schema = appSchema({
  version: 1,
  tables: [

    // ── MESSAGES ───────────────────────────────────────────────────────────────
    tableSchema({
      name: 'messages',
      columns: [
        { name: 'firestore_id',       type: 'string',  isOptional: true },
        { name: 'temp_id',            type: 'string',  isOptional: true },
        { name: 'sender_id',          type: 'string'  },
        { name: 'type',               type: 'string'  },

        // Decrypted content (stored locally unencrypted — device is trusted)
        { name: 'text',               type: 'string',  isOptional: true },
        { name: 'media_url',          type: 'string',  isOptional: true },
        { name: 'media_local_path',   type: 'string',  isOptional: true },
        { name: 'media_key',          type: 'string',  isOptional: true },
        { name: 'media_iv',           type: 'string',  isOptional: true },
        { name: 'media_public_id',    type: 'string',  isOptional: true },
        { name: 'duration',           type: 'number',  isOptional: true },
        { name: 'file_name',          type: 'string',  isOptional: true },
        { name: 'file_size',          type: 'number',  isOptional: true },
        { name: 'mime_type',          type: 'string',  isOptional: true },
        { name: 'gif_url',            type: 'string',  isOptional: true },

        // Structure
        { name: 'reply_to_id',        type: 'string',  isOptional: true },
        { name: 'effect_id',          type: 'string',  isOptional: true },
        { name: 'is_forwarded',       type: 'boolean' },

        // State
        { name: 'reactions_json',     type: 'string'  },  // JSON string
        { name: 'status',             type: 'string'  },
        { name: 'is_deleted',         type: 'boolean' },
        { name: 'is_system',          type: 'boolean' },
        { name: 'is_pinned',          type: 'boolean' },
        { name: 'read_by_json',       type: 'string'  },  // JSON string

        // Timestamps
        { name: 'timestamp',          type: 'number'  },  // Unix ms
        { name: 'send_at',            type: 'number',  isOptional: true },
      ],
    }),

    // ── MEDIA CACHE ────────────────────────────────────────────────────────────
    tableSchema({
      name: 'media_cache',
      columns: [
        { name: 'message_id',         type: 'string'  },
        { name: 'cloud_url',          type: 'string'  },
        { name: 'local_path',         type: 'string'  },
        { name: 'decrypted_at',       type: 'number'  },  // Unix ms
        { name: 'file_type',          type: 'string'  },  // 'image'|'video'|'voice'|'file'
        { name: 'size_bytes',         type: 'number'  },
      ],
    }),

    // ── LINK PREVIEWS ──────────────────────────────────────────────────────────
    tableSchema({
      name: 'link_previews',
      columns: [
        { name: 'url',                type: 'string'  },
        { name: 'url_hash',           type: 'string'  },
        { name: 'title',              type: 'string'  },
        { name: 'description',        type: 'string'  },
        { name: 'image_url',          type: 'string'  },
        { name: 'site_name',          type: 'string'  },
        { name: 'favicon',            type: 'string'  },
        { name: 'fetched_at',         type: 'number'  },
      ],
    }),

  ],
});
```

## WATERMELONDB MESSAGE MODEL (`src/db/models/MessageModel.ts`)

```typescript
import { Model } from '@nozbe/watermelondb';
import { field, date, json, readonly } from '@nozbe/watermelondb/decorators';

export class MessageModel extends Model {
  static table = 'messages';

  @field('firestore_id')     firestoreId!: string | null;
  @field('temp_id')          tempId!: string | null;
  @field('sender_id')        senderId!: string;
  @field('type')             type!: string;
  @field('text')             text!: string | null;
  @field('media_url')        mediaUrl!: string | null;
  @field('media_local_path') mediaLocalPath!: string | null;
  @field('media_key')        mediaKey!: string | null;
  @field('media_iv')         mediaIv!: string | null;
  @field('gif_url')          gifUrl!: string | null;
  @field('duration')         duration!: number | null;
  @field('file_name')        fileName!: string | null;
  @field('file_size')        fileSize!: number | null;
  @field('mime_type')        mimeType!: string | null;
  @field('reply_to_id')      replyToId!: string | null;
  @field('effect_id')        effectId!: string | null;
  @field('is_forwarded')     isForwarded!: boolean;
  @field('status')           status!: string;
  @field('is_deleted')       isDeleted!: boolean;
  @field('is_system')        isSystem!: boolean;
  @field('is_pinned')        isPinned!: boolean;
  @field('reactions_json')   reactionsJson!: string;
  @field('read_by_json')     readByJson!: string;
  @date('timestamp')         timestamp!: Date;
  @field('send_at')          sendAt!: number | null;

  // Convenience getters
  get reactions() { return JSON.parse(this.reactionsJson || '{}'); }
  get readBy()    { return JSON.parse(this.readByJson || '{}'); }
}
```

---

## ZUSTAND STORES

### authStore (`src/store/authStore.ts`)
```typescript
interface AuthState {
  myUid: string | null;
  myPublicKey: string | null;
  partnerPublicKey: string | null;
  isAuthenticated: boolean;
  isLoadingAuth: boolean;

  setUser: (uid: string) => void;
  setPublicKeys: (mine: string, partner: string) => void;
  signOut: () => void;
}
```

### chatStore (`src/store/chatStore.ts`)
```typescript
interface ChatState {
  // Optimistic messages (before Firestore confirms)
  pendingMessages: Map<string, Message>;
  
  // UI state
  replyTo: Message | null;
  selectedMessage: Message | null;   // for action sheet
  pendingEffect: Effect | null;      // effect to apply on next send
  isPartnerTyping: boolean;
  isPartnerOnline: boolean;
  partnerLastSeen: Date | null;
  highlightedMessageId: string | null;
  
  // Actions
  setReplyTo: (msg: Message | null) => void;
  setSelectedMessage: (msg: Message | null) => void;
  setPendingEffect: (effect: Effect | null) => void;
  setTypingState: (isTyping: boolean) => void;
  setOnlineState: (online: boolean, lastSeen: Date) => void;
  addPendingMessage: (msg: Message) => void;
  confirmPendingMessage: (tempId: string, firestoreId: string) => void;
  highlightMessage: (id: string) => void;
}
```

### uiStore (`src/store/uiStore.ts`)
```typescript
interface UIState {
  themeId: string;
  myMood: Mood | null;
  partnerMood: Mood | null;
  quickReactions: string[];
  doubleTapReaction: string;
  
  setTheme: (themeId: string) => void;
  setMyMood: (mood: Mood | null) => void;
  setPartnerMood: (mood: Mood | null) => void;
  setQuickReactions: (reactions: string[]) => void;
}
```

### callStore (`src/store/callStore.ts`)
```typescript
interface CallStoreState {
  call: CallState;
  localStream: MediaStream | null;
  remoteStream: MediaStream | null;
  peerConnection: RTCPeerConnection | null;
  
  initCall: (type: CallType, direction: 'incoming' | 'outgoing') => void;
  setStreams: (local: MediaStream, remote?: MediaStream) => void;
  updateCallState: (updates: Partial<CallState>) => void;
  endCall: () => void;
}
```

---

## FIRESTORE → LOCAL MESSAGE MAPPER

```typescript
// src/utils/messageMapper.ts
import { CONFIG } from '../config/config';
import { Message, MessageStatus } from '../constants/messages';
import { naclService } from '../crypto/naclService';

export const mapFirestoreToMessage = async (
  doc: FirebaseFirestoreTypes.DocumentData,
  id: string,
  partnerPublicKey: string
): Promise<Message> => {
  const data = doc;
  const isMe = data.senderId === CONFIG.USER_ME_UID;

  // Decrypt the payload
  let payload: any = {};
  try {
    const senderPublicKey = isMe
      ? await getMyPublicKey()   // my own messages — I encrypted them too
      : partnerPublicKey;
    const plaintext = await naclService.decryptMessage(
      data.ciphertext,
      data.nonce,
      senderPublicKey
    );
    payload = JSON.parse(plaintext);
  } catch (e) {
    console.error('Decryption failed for message', id, e);
    payload = { text: '[Could not decrypt]' };
  }

  return {
    id,
    sender: isMe ? 'me' : 'partner',
    senderId: data.senderId,
    type: data.type,
    text: payload.text ?? payload.sticker ?? undefined,
    gifUrl: payload.gifUrl,
    mediaUrl: data.mediaUrl,             // Cloudinary URL (from Firestore)
    mediaKey: payload.mediaKey,          // AES key (from decrypted payload)
    mediaIv: payload.mediaIv,            // AES IV (from decrypted payload)
    mediaPublicId: data.mediaPublicId,
    duration: payload.duration,
    fileName: payload.fileName,
    fileSize: payload.fileSize,
    mimeType: payload.mimeType,
    replyToId: data.replyToId,
    effectId: data.effectId,
    isForwarded: data.isForwarded ?? false,
    reactions: data.reactions ?? {},
    status: data.status as MessageStatus,
    isDeleted: data.isDeleted ?? false,
    isSystem: data.isSystem ?? false,
    isPinned: data.isPinned ?? false,
    readBy: Object.fromEntries(
      Object.entries(data.readBy ?? {}).map(([k, v]: any) => [k, v.toDate()])
    ),
    timestamp: data.createdAt?.toDate() ?? new Date(),
    sendAt: data.sendAt?.toDate(),
  };
};
```
