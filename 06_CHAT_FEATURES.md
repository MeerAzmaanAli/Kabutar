# 06 — CHAT FEATURES FLOW
# Every feature: trigger → state change → Firestore → UI update

---

## SENDING A TEXT MESSAGE

```
1. User types in InputBar TextInput
2. onChange → chatStore.setTyping(true)
   → presenceService.setTyping(myUid, true)
   → Realtime DB sets presence/{myUid}/typing = true
   → Partner's usePresence hook fires → chatStore.setPartnerTyping shown
3. 2s idle → presenceService.setTyping(myUid, false)

4. User presses Send (or Enter):
   → InputBar.onSendText(trimmedText) callback
   → ChatScreen creates optimistic Message object:
       { id: uuid(), sender:'me', type:'text', text, status:'pending', timestamp:now }
   → chatStore.addPendingMessage(msg)   [instantly visible in FlatList]
   → inputStore.setReplyTo(null)
   → FlatList scrollToEnd

5. Background: naclService.encryptMessage(JSON.stringify({text}), partnerPublicKey)
   → firestoreService.sendMessage({ senderId, type:'text', ciphertext, nonce })
   → returns firestoreId

6. chatStore.confirmPendingMessage(tempId, firestoreId)
   → message status: 'pending' → 'sent'
   → UI shows ✓ (sent tick)

7. Partner's subscribeToMessages() onSnapshot fires
   → mapFirestoreToMessage() decrypts → Message object
   → WatermelonDB.upsert(message)
   → FlatList rerenders with new message
   → presenceService reads: partner opened chat? → markRead()
```

---

## SENDING MEDIA (IMAGE/VIDEO/FILE)

```
1. User taps 📎 → AttachSheet opens
2. User selects "Camera" or "Gallery"
   Camera → VisionCamera → captured file path
   Gallery → ImageCropPicker → selected file path(s)

3. For image: navigate to PhotoEditorScreen(filePath)
   → User edits (crop/filter/draw/text/sticker)
   → Returns edited file path

4. For video: navigate to VideoTrimmerScreen(filePath)
   → User trims
   → Returns trimmed file path

5. Back in ChatScreen:
   → show upload progress in input bar (0→100%)
   → messageId = uuid() (used as Cloudinary public_id)
   → aesService.encryptFile(editedPath, tempEncPath)
      → returns { encryptedPath, key, iv }
   → cloudinaryService.uploadEncryptedFile(encryptedPath, messageId, onProgress)
      → returns { secureUrl, publicId }
   → RNFS.unlink(encryptedPath)  [clean up encrypted temp]

6. Build payload: { type:'image', mediaUrl:secureUrl, mediaPublicId, mediaKey:key, mediaIv:iv }
   → naclService.encryptMessage(JSON.stringify(payload), partnerPublicKey)
   → firestoreService.sendMessage({
       ciphertext, nonce, type:'image',
       mediaUrl: secureUrl,    ← stored in Firestore (thumbnail placeholder)
       mediaPublicId
     })

7. Optimistic message shows with placeholder spinner
   → Confirmed on firestoreId received
```

---

## RECEIVING + DISPLAYING MEDIA

```
1. onSnapshot fires → mapFirestoreToMessage() → Message with:
   mediaUrl: Cloudinary URL (stored in Firestore — visible but useless without AES key)
   mediaKey: null   (NOT in Firestore — comes from decrypted NaCl payload)
   
2. MediaMessage component renders:
   - Shows placeholder (blurred preview or loading spinner)
   - For voice: auto-download (small file)
   - For images/video/files: show placeholder until tapped

3. User TAPS image/video:
   → navigate to MediaViewerScreen
   → MediaViewerScreen:
       a. Check WatermelonDB media_cache for this messageId
       b. If cached: use localPath directly
       c. If not cached:
          → cloudinaryService.downloadEncryptedFile(mediaUrl, tempEncPath)
          → aesService.decryptFile(tempEncPath, mediaKey, mediaIv, decryptedPath)
          → RNFS.unlink(tempEncPath)  [clean up encrypted download]
          → MediaCacheModel.create({ messageId, localPath: decryptedPath, ... })
       d. Display decryptedPath using Image / Video component

4. Save to gallery: CameraRoll.save(decryptedPath)
```

---

## VOICE MESSAGE RECORD + SEND

```
1. User HOLDS mic button
   → RecordingBar appears (replaces InputBar)
   → react-native-audio-recorder-player.startRecorder(tempPath)
   → Interval: update recSecs every second
   → At 4:50: warning toast "Voice message ending soon"
   → At 5:00: auto-stop

2. User RELEASES mic button (or taps Send in RecordingBar):
   → react-native-audio-recorder-player.stopRecorder() → returns filePath
   → aesService.encryptFile(filePath, encPath)
   → cloudinaryService.uploadEncryptedFile(encPath, messageId)
   → payload = { type:'voice', mediaUrl, mediaPublicId, mediaKey, mediaIv, duration: recSecs }
   → encrypt + send to Firestore

3. User SWIPES LEFT during recording:
   → react-native-audio-recorder-player.stopRecorder()
   → RNFS.unlink(tempPath)  [discard]
   → RecordingBar disappears, InputBar returns

VOICE PLAYBACK:
1. VoiceMessage component renders waveform (32 seeded bars)
2. For own messages: localPath available from send
3. For partner messages: auto-download on receive (voice is small)
   → aesService.decryptFile → cache to CachesDirectory/voice/{id}.m4a
4. Tap ▶: react-native-audio-recorder-player.startPlayer(localPath)
5. PanResponder on waveform → seekToPlayer(ms)
6. onPlaybackComplete → reset state
```

---

## REACTIONS

```
ADDING REACTION:
1. Long-press bubble → MessageActionsSheet opens
   → ReactionBar shows 6 quick reactions + "+" button
2. Tap emoji:
   → Haptic.trigger('impactLight')
   → firestoreService.reactToMessage(messageId, emoji, myUid, 'add')
   → Firestore updates reactions.{emoji}.count += 1
   → Both users' onSnapshot fires → reactions update in UI

DOUBLE-TAP REACTION:
1. Double-tap bubble (TapGestureHandler numberOfTaps=2)
   → Same as tapping first quick reaction (doubleTapReaction from uiStore)

REMOVE REACTION:
1. Tap own reaction pill → call reactToMessage with 'remove'
   → count -= 1
   → If count === 0: pill disappears (filter in ReactionPills component)

REACTIONS PLUS:
1. Long-press own reaction pill → "Reactions Plus" mode in sheet
2. Select up to 3 more emoji
3. reactions.{baseEmoji}.type = 'plus'
   reactions.{baseEmoji}.combined = ["❤️","😂","🔥"]
4. ReactionPills renders cycling animation through combined emoji
```

---

## REPLY / QUOTE

```
TRIGGER REPLY:
1. Swipe right on bubble (Swipeable from react-native-gesture-handler)
   OR long-press → "Reply" in MessageActionsSheet
2. chatStore.setReplyTo(message)
3. ReplyBar renders above InputBar:
   - sender name (accent color)
   - truncated content or "[Photo]"/"[Voice]" label
   - × dismiss button

SEND WITH REPLY:
1. onSendText called
2. payload includes replyToId: replyTo.id
3. Firestore message has replyToId field
4. On receive: mapFirestoreToMessage fetches replyTo message from WatermelonDB
   (by replyToId) and attaches as nested replyTo object

SCROLL TO ORIGINAL:
1. Tap inset card in bubble
2. onScrollToReply(replyToId) callback
3. ChatScreen: find index in messages array
   → FlatList.scrollToIndex({ index, animated: true, viewPosition: 0.5 })
4. chatStore.highlightMessage(replyToId)
   → bubble background flashes for 1500ms (Reanimated)
5. chatStore.clearHighlight() after 1500ms
```

---

## TYPING INDICATOR

```
SENDER SIDE (me):
1. TextInput.onChangeText fires
2. If text non-empty AND not already flagged typing:
   → presenceService.setTyping(myUid, true)
   → start 2s debounce timer
3. On every keystroke: reset the 2s timer
4. When 2s passes without keystroke:
   → presenceService.setTyping(myUid, false)
5. On text clear: immediate setTyping(false)

RECEIVER SIDE (partner):
1. usePresence hook has Realtime DB listener:
   database().ref('presence/{partnerUid}/typing').on('value', ...)
2. typing = true → chatStore.setPartnerTyping(true)
3. TypingIndicator component appears at bottom of FlatList
4. typing = false → TypingIndicator disappears (Reanimated FadeOut)
```

---

## READ RECEIPTS

```
MARK AS READ:
1. FlatList viewabilityConfig:
   { itemVisiblePercentThreshold: 80, minimumViewTime: 100 }
2. onViewableItemsChanged fires
3. For each visible received message where readBy doesn't have myUid:
   → firestoreService.markRead(messageId, myUid)
   → Firestore: readBy.{myUid} = serverTimestamp(), status = 'read'

DISPLAY:
Sender side — each message shows:
- ✓ grey: status = 'sent'
- ✓✓ grey: status = 'delivered' (partner device received onSnapshot)
- ✓✓ accent color: status = 'read' (partner marked it read)
- "Seen 3:45 PM": text below last read message

DELIVERY STATUS:
When partner's device receives the message via onSnapshot:
→ Partner client updates readBy with partial (delivered flag)
→ We detect this by checking if readBy has partner's uid without a read timestamp
→ Show ✓✓ grey
```

---

## PINS

```
PIN MESSAGE:
1. Long-press → "Pin" in MessageActionsSheet
2. Check: firestoreService.getPinnedCount()
   - If < 3: proceed
   - If = 3: show "Replace a pinned message?" sheet
3. firestoreService.pinMessage(messageId)
   → Firestore: isPinned = true
   → conversation/main.pinnedMessages array updated (arrayUnion)
4. Both users' conversation onSnapshot fires
   → PinnedBar re-renders with new pinned list

PINNED BAR:
- Shown between header and message list
- Cycles through pinned messages if > 1 (tap › to cycle)
- Tap: scrollToMessage + highlight
- ✕ button: unpin that message

UNPIN:
1. ✕ on PinnedBar OR "Unpin" in MessageActionsSheet
2. firestoreService.unpinMessage(messageId)
   → isPinned = false, arrayRemove from pinnedMessages
```

---

## THEMES

```
CHANGE THEME:
1. Settings → "Chat Theme" → ThemeSheet opens
2. User selects theme → uiStore.setTheme(themeId)
   → immediate visual update (all components use theme from uiStore)
3. firestoreService.updateTheme(themeId)
   → Firestore: conversation/main.theme = themeId
4. subscribeToConversation() onSnapshot fires on partner's device
   → uiStore.setTheme(themeId) on partner's device too
   → Reanimated color interpolation (300ms fade)

THEME OBJECT:
Every component receives theme prop.
No hardcoded colors anywhere.
```

---

## MOOD

```
SET MOOD:
1. Tap own avatar in header → MoodSheet opens
2. Select mood emoji
3. uiStore.setMyMood(mood)  [immediate local update]
4. firestoreService.updateMood(myUid, emoji, label)
   → Firestore: users/{myUid}.moodEmoji = emoji, moodLabel = label
5. subscribeToPartnerUser() on partner side fires
   → partner sees mood emoji next to my name in their header
   → emoji pulses once (Reanimated scale 1→1.5→1)

DISPLAY:
Header: "Love 💖 🥰"  (name + partner's mood emoji)
Tooltip on long-press mood: "Feeling Romantic"
My mood shown as badge on SettingsSheet profile section
```

---

## SEARCH

```
1. Tap 🔍 in header → SearchScreen (modal stack push)
2. TextInput autoFocus
3. onChange → WatermelonDB query:
   db.collections.get('messages')
     .query(
       Q.where('is_deleted', false),
       Q.where('text', Q.like(`%${searchTerm}%`))
     )
   → filtered by type chip (All/Text/Photos/Videos/Voice/Files)
4. Results in SectionList grouped by date
5. Matched text highlighted: split text by match, render spans with accent background
6. Tap result:
   → pop SearchScreen
   → chatStore.highlightMessage(id)
   → ChatScreen: scrollToMessage(id)
```

---

## SCHEDULED MESSAGES

```
SCHEDULE:
1. Long-press send button → bottom sheet shows "Send now" / "Schedule"
2. "Schedule" → DateTimePicker → user picks date + time
3. Validate: must be > 5 minutes from now
4. naclService.encryptMessage(payload)
5. firestoreService.sendMessage({
     ...,
     status: 'scheduled',
     sendAt: Timestamp.fromDate(scheduledDate)
   })
6. Message NOT visible in chat list (subscribeToMessages filters status != 'scheduled')
7. ScheduledChip shows "Scheduled (1)" above InputBar

RELEASE (Cloud Function):
1. releaseScheduledMessages runs every 1 minute (Cloud Scheduler)
2. Queries messages WHERE status='scheduled' AND sendAt <= now
3. Updates status to 'sent'
4. Both users' onSnapshot fires → message appears in chat

CANCEL:
1. Open ScheduledSheet → tap "Cancel" on a message
2. firestoreService.deleteMessage(id)  [hard delete — it was never sent]
3. ScheduledChip count decrements
```

---

## MESSAGE EFFECTS

```
SEND WITH EFFECT:
1. Type message text
2. Tap ✨ button → EffectsSheet opens (6 effects)
3. Tap effect → effectId stored in chatStore.pendingEffect
4. EffectsSheet closes
5. Send button pressed:
   → payload includes effectId
   → firestoreService.sendMessage({..., effectId: 'confetti'})

RECEIVE WITH EFFECT:
1. onSnapshot → decrypted message has effectId
2. MessageBubble renders with effectId
3. On FIRST render of this message (not on scroll):
   → chatStore checks if effectId already played for this messageId
   → If not: trigger ParticleEffect overlay for 2.5s
   → Mark as played (stored in AsyncStorage: Set<messageId>)
```

---

## DELETE MESSAGE

```
DELETE FOR ME:
1. Long-press → "Delete" → "Delete for me"
2. WatermelonDB.update(message, { isDeleted: true })
3. Firestore NOT touched — partner still sees message
4. Local FlatList re-renders (message shows deleted placeholder)

DELETE FOR EVERYONE:
1. Only available within 10 minutes of send (own messages)
2. Check: Date.now() - message.timestamp.getTime() < 10 * 60 * 1000
3. firestoreService.deleteMessage(messageId) → isDeleted = true
4. Both users' onSnapshot fires
5. Both devices: message replaced with "This message was deleted"
6. If message had media: cloudinaryService.deleteFile(publicId)
   → via Cloud Function (needs signed request)
```

---

## APP LOCK (BIOMETRICS)

```
1. useBiometricLock hook listens to AppState:
   'active' → 'background': record backgroundedAt = Date.now()
   
2. 'background' → 'active':
   if (Date.now() - backgroundedAt > 2 * 60 * 1000):
     → show BiometricLockScreen (modal, blurs content)
     → react-native-biometrics.simplePrompt({ promptMessage: 'Unlock chat' })
     → on success: hide BiometricLockScreen
     → on fail: keep showing lock screen (allow retry)

3. BiometricLockScreen prevents any screenshot or content view
   (FLAG_SECURE + blur overlay)
```
