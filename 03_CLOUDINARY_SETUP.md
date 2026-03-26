# 03 — CLOUDINARY SETUP
# Media storage replacing Firebase Storage (removed from free plan)

---

## WHY CLOUDINARY

Firebase Storage was removed from the free Spark plan.
Cloudinary free tier gives us:
- **25 GB storage** — enough for months of photos/videos/voice
- **25 GB bandwidth/month** — plenty for 2 users
- **Direct upload from mobile** — no backend server needed
- **Raw resource type** — stores any binary blob as-is (our AES-encrypted files)

---

## FREE PLAN LIMITS (more than enough for 2 users)

| Resource | Free Limit | Estimated Usage |
|---|---|---|
| Storage | 25 GB | ~500 voice msgs + ~200 photos/month |
| Bandwidth | 25 GB/month | ~2-3 GB/month |
| Transformations | 25 credits/month | NOT USED (we upload raw encrypted) |
| API calls | Unlimited | ✅ |

---

## CLOUDINARY PROJECT SETUP

1. Create account at cloudinary.com (free)
2. Dashboard → Settings → Upload Presets → Add upload preset
   - **Preset name**: `private_chat_media`
   - **Signing Mode**: **Unsigned** (required for direct mobile upload)
   - **Folder**: `private_chat/media`
   - **Resource type**: **Raw** (because files are AES-encrypted binaries)
   - **Access mode**: Authenticated (default)
   - Disable all transformations
3. Note your **Cloud Name** from dashboard
4. Set in `src/config/config.ts`:
   ```typescript
   CLOUDINARY_CLOUD_NAME: 'your_cloud_name',
   CLOUDINARY_UPLOAD_PRESET: 'private_chat_media',
   ```

---

## CLOUDINARY SERVICE (`src/cloudinary/cloudinaryService.ts`)

```typescript
import RNFS from 'react-native-fs';
import { CONFIG } from '../config/config';

const BASE_URL = `https://api.cloudinary.com/v1_1/${CONFIG.CLOUDINARY_CLOUD_NAME}`;

// ── UPLOAD ENCRYPTED BLOB ─────────────────────────────────────────────────────
interface UploadResult {
  secureUrl: string;
  publicId: string;
  bytes: number;
}

export const uploadEncryptedFile = async (
  encryptedFilePath: string,    // local path to AES-encrypted file
  messageId: string,            // used as filename prefix
  onProgress?: (percent: number) => void
): Promise<UploadResult> => {
  const formData = new FormData();

  formData.append('file', {
    uri: `file://${encryptedFilePath}`,
    type: 'application/octet-stream',
    name: `${messageId}.enc`,
  } as any);

  formData.append('upload_preset', CONFIG.CLOUDINARY_UPLOAD_PRESET);
  formData.append('public_id', `private_chat/media/${messageId}`);
  formData.append('resource_type', 'raw');

  // Use XMLHttpRequest for upload progress
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();

    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) {
        onProgress?.(Math.round((e.loaded / e.total) * 100));
      }
    };

    xhr.onload = () => {
      if (xhr.status === 200) {
        const res = JSON.parse(xhr.responseText);
        resolve({
          secureUrl: res.secure_url,
          publicId: res.public_id,
          bytes: res.bytes,
        });
      } else {
        reject(new Error(`Upload failed: ${xhr.status} ${xhr.responseText}`));
      }
    };

    xhr.onerror = () => reject(new Error('Upload network error'));

    xhr.open('POST', `${BASE_URL}/raw/upload`);
    xhr.send(formData);
  });
};

// ── DOWNLOAD ENCRYPTED BLOB ───────────────────────────────────────────────────
export const downloadEncryptedFile = async (
  secureUrl: string,
  destinationPath: string  // local temp path to save the encrypted file
): Promise<string> => {
  const result = await RNFS.downloadFile({
    fromUrl: secureUrl,
    toFile: destinationPath,
    background: false,
  }).promise;

  if (result.statusCode !== 200) {
    throw new Error(`Download failed: status ${result.statusCode}`);
  }

  return destinationPath;
};

// ── DELETE FILE ───────────────────────────────────────────────────────────────
// NOTE: Cloudinary unsigned presets cannot delete via API without a signed request.
// For privacy on message delete, use Cloud Function with API secret.
// Client-side: just remove the Firestore record. Scheduled cleanup via Cloud Function.
export const deleteFile = async (publicId: string): Promise<void> => {
  // This requires a signed request — done via Cloud Function
  // See functions/src/deleteMedia.ts
  // For MVP: skip deletion (files auto-expire if you set expiry rules in Cloudinary)
  console.warn('File deletion requires signed request via Cloud Function', publicId);
};

// ── GENERATE TEMP PATH ────────────────────────────────────────────────────────
export const getTempPath = (messageId: string, ext = 'enc') =>
  `${RNFS.TemporaryDirectoryPath}/${messageId}.${ext}`;

export const getDecryptedTempPath = (messageId: string, ext: string) =>
  `${RNFS.TemporaryDirectoryPath}/${messageId}_dec.${ext}`;
```

---

## COMPLETE MEDIA SEND FLOW

```
┌─── User picks/captures media ───────────────────────────────────────┐
│                                                                       │
│  1. Get file path (from VisionCamera / ImageCropPicker / DocPicker)  │
│                                                                       │
│  2. Optional editing:                                                 │
│     - Image → PhotoEditorScreen → returns edited file path           │
│     - Video → VideoTrimmerScreen → returns trimmed file path          │
│                                                                       │
│  3. Compress (before encryption):                                     │
│     - Image: resize to max 1280px, JPEG quality 0.8                  │
│     - Video: warn if > 50MB                                           │
│     - Voice: already compressed (AAC 64kbps from recorder)           │
│                                                                       │
│  4. aesService.encryptFile(filePath)                                  │
│     → Returns: { encryptedPath, key (32 bytes), iv (12 bytes) }      │
│     → key + iv will travel INSIDE the NaCl-encrypted message         │
│                                                                       │
│  5. cloudinaryService.uploadEncryptedFile(encryptedPath, messageId)  │
│     → Returns: { secureUrl, publicId }                                │
│                                                                       │
│  6. Build message payload:                                            │
│     const plaintext = JSON.stringify({                                │
│       type: 'image',                                                  │
│       mediaUrl: secureUrl,                                            │
│       mediaPublicId: publicId,                                        │
│       mediaKey: Buffer.from(key).toString('base64'),                  │
│       mediaIv: Buffer.from(iv).toString('base64'),                    │
│       // For non-media: text goes here                                │
│     });                                                               │
│                                                                       │
│  7. naclService.encryptMessage(plaintext, partnerPublicKey)           │
│     → Returns: { ciphertext, nonce }                                  │
│                                                                       │
│  8. firestoreService.sendMessage({                                    │
│       ciphertext, nonce, type: 'image',                               │
│       mediaUrl: secureUrl,  ← stored unencrypted for thumbnail later │
│       mediaPublicId: publicId,                                        │
│     })                                                                │
│                                                                       │
│  9. Clean up: RNFS.unlink(encryptedPath)                              │
└───────────────────────────────────────────────────────────────────────┘
```

## COMPLETE MEDIA RECEIVE FLOW

```
┌─── Partner sends media → onSnapshot fires ──────────────────────────┐
│                                                                       │
│  1. Firestore onSnapshot delivers new message doc                     │
│     → { ciphertext, nonce, type: 'image', mediaUrl, ... }            │
│                                                                       │
│  2. naclService.decryptMessage(ciphertext, nonce, partnerPublicKey)   │
│     → Returns plaintext JSON string                                   │
│                                                                       │
│  3. Parse plaintext:                                                  │
│     const { type, mediaUrl, mediaKey, mediaIv } = JSON.parse(...)    │
│                                                                       │
│  4. Store in WatermelonDB with mediaUrl (for display placeholder)     │
│     and decryptedMediaKey (encrypted at rest in WatermelonDB)         │
│                                                                       │
│  5. When user VIEWS the media (taps bubble):                          │
│     a. cloudinaryService.downloadEncryptedFile(mediaUrl, tempPath)    │
│     b. aesService.decryptFile(tempPath, key, iv)  → decryptedPath     │
│     c. Display decryptedPath in MediaViewerScreen                     │
│     d. After viewing: RNFS.unlink(decryptedPath) from temp             │
│        OR cache for N minutes (configurable)                          │
│                                                                       │
│  6. Download only happens on-demand (tap) — not automatically         │
│     This saves bandwidth and storage                                  │
└───────────────────────────────────────────────────────────────────────┘
```

---

## VOICE MESSAGE SPECIFICS

```typescript
// Voice messages are small (max 5min ≈ 2.4MB at 64kbps)
// Always download + cache on receive (auto-play is expected)

// On send:
// 1. react-native-audio-recorder-player records → .m4a file path
// 2. aesService.encryptFile(.m4a path) → .enc path
// 3. cloudinaryService.uploadEncryptedFile(.enc path, messageId)
// 4. Send message with duration in payload

// On receive:
// Auto-download voice messages (they're small)
// Cache decrypted .m4a in: RNFS.CachesDirectoryPath/voice/{messageId}.m4a
// Check cache first before downloading
```

---

## FILE SIZE LIMITS & WARNINGS

```typescript
export const FILE_LIMITS = {
  IMAGE_MAX_BYTES:  10 * 1024 * 1024,   // 10 MB
  VIDEO_MAX_BYTES:  100 * 1024 * 1024,  // 100 MB
  FILE_MAX_BYTES:   100 * 1024 * 1024,  // 100 MB
  FILE_WARN_BYTES:  50 * 1024 * 1024,   // warn at 50 MB
  VOICE_MAX_SECS:   300,                 // 5 minutes
  VIDEO_TRIM_MAX_SECS: 120,              // 2 minutes after trim
};
```

---

## MEDIA CACHE STRATEGY

```typescript
// WatermelonDB MediaCache table:
// - url: string (Cloudinary URL)
// - localPath: string (decrypted file in CachesDirectory)
// - decryptedAt: number (timestamp)
// - messageId: string

// Cache policy:
// - Voice messages: auto-download, cache for 7 days
// - Images: download on tap, cache for 24 hours
// - Videos: download on tap, NO auto-cache (too large)
// - Files: download on demand, save to Downloads (permanent)

// Cache cleanup: on app start, delete entries older than their TTL
```

---

## CLOUDINARY FOLDER STRUCTURE

```
private_chat/
  media/
    {messageId}.enc      ← all files (images, videos, voice, files)
```

No subfolders by type — everything is an opaque encrypted blob anyway.
The type information is inside the NaCl-encrypted message payload.
