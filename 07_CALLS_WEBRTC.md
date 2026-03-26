# 07 — CALLS & WEBRTC
# Voice and video call implementation using WebRTC + Realtime DB signaling

---

## OVERVIEW

Calls are P2P (peer-to-peer) via WebRTC.
Firebase Realtime Database is used ONLY for signaling (exchanging SDP offer/answer/ICE).
Zero call audio/video ever touches Firebase or Cloudinary.

```
┌─── My Phone ────────────────────────────────────────────────────────┐
│                                                                       │
│  react-native-webrtc                                                  │
│    ↕ (audio/video stream — direct P2P, encrypted by DTLS)            │
└──────────────────────────────────────────────────────────────────────┘
         ↕ (signaling only: SDP + ICE candidates)
┌─── Firebase Realtime DB ────────────────────────────────────────────┐
│  calls/active/    (call invite, accept, decline)                      │
│  calls/signals/   (offer, answer, iceCandidates)                     │
└──────────────────────────────────────────────────────────────────────┘
         ↕ (signaling only)
┌─── Partner's Phone ─────────────────────────────────────────────────┐
│                                                                       │
│  react-native-webrtc                                                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## REALTIME DB CALL STRUCTURE

```json
{
  "calls": {
    "active": {
      "callerId": "USER_ME_UID",
      "callType": "video",
      "status": "ringing",    // "ringing" | "accepted" | "declined" | "ended"
      "createdAt": 1735000000000
    },
    "signals": {
      "offer": {
        "type": "offer",
        "sdp": "v=0\r\no=- ..."
      },
      "answer": {
        "type": "answer",
        "sdp": "v=0\r\no=- ..."
      },
      "iceCandidates": {
        "-OaBC123": {
          "candidate": "candidate:...",
          "sdpMid": "0",
          "sdpMLineIndex": 0,
          "senderId": "USER_ME_UID"
        }
      }
    }
  }
}
```

---

## CALL FLOW: OUTGOING

```
CALLER (me) taps 📞 or 📹:
│
├─1. callStore.initCall(type, 'outgoing')
│
├─2. Get local media stream:
│    const stream = await mediaDevices.getUserMedia({
│      audio: true,
│      video: type === 'video' ? { facingMode: 'user', width: 640, height: 480 } : false
│    })
│    callStore.setStreams(stream)
│
├─3. Create RTCPeerConnection:
│    const pc = new RTCPeerConnection({
│      iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
│    })
│    stream.getTracks().forEach(track => pc.addTrack(track, stream))
│    callStore.setPeerConnection(pc)
│
├─4. Listen for ICE candidates:
│    pc.onicecandidate = (event) => {
│      if (event.candidate) {
│        database().ref('calls/signals/iceCandidates').push({
│          ...event.candidate.toJSON(),
│          senderId: myUid
│        })
│      }
│    }
│
├─5. Listen for remote stream:
│    pc.ontrack = (event) => {
│      callStore.setRemoteStream(event.streams[0])
│    }
│
├─6. Write call invite to Realtime DB:
│    database().ref('calls/active').set({
│      callerId: myUid,
│      callType: type,
│      status: 'ringing',
│      createdAt: Date.now()
│    })
│
├─7. Create SDP offer:
│    const offer = await pc.createOffer()
│    await pc.setLocalDescription(offer)
│    database().ref('calls/signals/offer').set({ type: offer.type, sdp: offer.sdp })
│
├─8. Show OutgoingCallScreen (ringing state)
│
├─9. Listen for answer (30s timeout):
│    database().ref('calls/active/status').on('value', (snap) => {
│      if (snap.val() === 'accepted') → handleCallAccepted()
│      if (snap.val() === 'declined') → endCall()
│    })
│    setTimeout(() => endCall('No answer'), 30000)
│
└─10. On accepted:
     database().ref('calls/signals/answer').once('value', (snap) => {
       pc.setRemoteDescription(new RTCSessionDescription(snap.val()))
     })
     // Listen for partner ICE candidates:
     database().ref('calls/signals/iceCandidates').on('child_added', (snap) => {
       const data = snap.val()
       if (data.senderId !== myUid) {
         pc.addIceCandidate(new RTCIceCandidate(data))
       }
     })
     callStore.updateCallState({ status: 'active' })
     // Show ActiveCallScreen
```

---

## CALL FLOW: INCOMING

```
PARTNER (them) receives call:
│
├─1. usePresence / useCall hook listens to:
│    database().ref('calls/active').on('value', ...)
│    When status = 'ringing' AND callerId ≠ myUid:
│      → callStore.initCall(type, 'incoming')
│      → Show IncomingCallScreen
│      → react-native-callkeep.displayIncomingCall(...)  ← native call UI
│      → Vibrate + ringtone via react-native-incall-manager
│
├─2. User taps DECLINE:
│    database().ref('calls/active/status').set('declined')
│    callStore.endCall()
│
└─3. User taps ACCEPT:
     ├─a. Get local media stream (same as caller step 2)
     │
     ├─b. Create RTCPeerConnection (same config)
     │    stream.getTracks().forEach(track => pc.addTrack(track, stream))
     │    pc.onicecandidate = ... (same as caller, pushes to iceCandidates)
     │    pc.ontrack = ... (sets remoteStream)
     │
     ├─c. Read the offer:
     │    database().ref('calls/signals/offer').once('value', (snap) => {
     │      pc.setRemoteDescription(new RTCSessionDescription(snap.val()))
     │    })
     │
     ├─d. Create answer:
     │    const answer = await pc.createAnswer()
     │    await pc.setLocalDescription(answer)
     │    database().ref('calls/signals/answer').set({ type: answer.type, sdp: answer.sdp })
     │
     ├─e. Update call status:
     │    database().ref('calls/active/status').set('accepted')
     │
     ├─f. Listen for caller's ICE candidates (same as caller step 10)
     │
     └─g. Show ActiveCallScreen
```

---

## END CALL

```
Either user presses End:
1. database().ref('calls/active/status').set('ended')
2. database().ref('calls').remove()  [clean up signals too]
3. peerConnection.close()
4. localStream.getTracks().forEach(t => t.stop())
5. callStore.endCall()   [resets all call state]
6. Navigation: pop call screen, return to ChatScreen
7. react-native-callkeep.endCall(...)  [dismiss native call UI]
```

---

## WEBRTC HOOK (`src/hooks/useWebRTC.ts`)

```typescript
import { useEffect, useRef, useCallback } from 'react';
import {
  RTCPeerConnection,
  RTCIceCandidate,
  RTCSessionDescription,
  mediaDevices,
} from 'react-native-webrtc';
import database from '@react-native-firebase/database';
import { useCallStore } from '../store/callStore';
import { CONFIG } from '../config/config';

const ICE_SERVERS = [
  { urls: 'stun:stun.l.google.com:19302' },
  { urls: 'stun:stun1.l.google.com:19302' },
];

export const useWebRTC = () => {
  const pcRef = useRef<RTCPeerConnection | null>(null);
  const { call, setStreams, updateCallState, endCall } = useCallStore();

  const createPC = useCallback(() => {
    const pc = new RTCPeerConnection({ iceServers: ICE_SERVERS });

    pc.onicecandidate = (event) => {
      if (event.candidate) {
        database().ref('calls/signals/iceCandidates').push({
          ...event.candidate.toJSON(),
          senderId: CONFIG.USER_ME_UID,
        });
      }
    };

    pc.ontrack = (event) => {
      if (event.streams[0]) {
        setStreams(pcRef.current?.getLocalStreams()[0]!, event.streams[0]);
      }
    };

    pc.onconnectionstatechange = () => {
      if (pc.connectionState === 'disconnected' || pc.connectionState === 'failed') {
        handleEndCall();
      }
    };

    pcRef.current = pc;
    return pc;
  }, []);

  const startOutgoingCall = useCallback(async (callType: 'voice' | 'video') => {
    const stream = await mediaDevices.getUserMedia({
      audio: true,
      video: callType === 'video' ? { facingMode: 'user', width: 640, height: 480 } : false,
    });

    const pc = createPC();
    stream.getTracks().forEach(track => pc.addTrack(track, stream));
    setStreams(stream);

    await database().ref('calls/active').set({
      callerId: CONFIG.USER_ME_UID,
      callType,
      status: 'ringing',
      createdAt: Date.now(),
    });

    const offer = await pc.createOffer({});
    await pc.setLocalDescription(offer);
    await database().ref('calls/signals/offer').set({
      type: offer.type,
      sdp: offer.sdp,
    });

    // Listen for answer
    const answerRef = database().ref('calls/signals/answer');
    answerRef.on('value', async (snap) => {
      if (snap.val() && !pc.currentRemoteDescription) {
        await pc.setRemoteDescription(new RTCSessionDescription(snap.val()));
        answerRef.off('value');
      }
    });

    listenForIceCandidates(pc);
  }, [createPC]);

  const acceptIncomingCall = useCallback(async () => {
    const callSnap = await database().ref('calls/active').once('value');
    const callType = callSnap.val()?.callType ?? 'voice';

    const stream = await mediaDevices.getUserMedia({
      audio: true,
      video: callType === 'video' ? { facingMode: 'user', width: 640, height: 480 } : false,
    });

    const pc = createPC();
    stream.getTracks().forEach(track => pc.addTrack(track, stream));
    setStreams(stream);

    const offerSnap = await database().ref('calls/signals/offer').once('value');
    await pc.setRemoteDescription(new RTCSessionDescription(offerSnap.val()));

    const answer = await pc.createAnswer();
    await pc.setLocalDescription(answer);

    await database().ref('calls/signals/answer').set({
      type: answer.type,
      sdp: answer.sdp,
    });

    await database().ref('calls/active/status').set('accepted');

    listenForIceCandidates(pc);
  }, [createPC]);

  const listenForIceCandidates = (pc: RTCPeerConnection) => {
    database().ref('calls/signals/iceCandidates').on('child_added', (snap) => {
      const data = snap.val();
      if (data.senderId !== CONFIG.USER_ME_UID) {
        pc.addIceCandidate(new RTCIceCandidate(data));
      }
    });
  };

  const handleEndCall = useCallback(async () => {
    await database().ref('calls/active/status').set('ended');
    await database().ref('calls').remove();
    pcRef.current?.close();
    pcRef.current = null;
    endCall();
  }, []);

  const toggleMute = useCallback(() => {
    const stream = pcRef.current?.getLocalStreams()[0];
    stream?.getAudioTracks().forEach(t => { t.enabled = !t.enabled; });
    updateCallState({ isMuted: !useCallStore.getState().call.isMuted });
  }, []);

  const toggleCamera = useCallback(() => {
    const stream = pcRef.current?.getLocalStreams()[0];
    stream?.getVideoTracks().forEach(t => { t.enabled = !t.enabled; });
    updateCallState({ isCameraOff: !useCallStore.getState().call.isCameraOff });
  }, []);

  const flipCamera = useCallback(async () => {
    const stream = pcRef.current?.getLocalStreams()[0];
    const videoTrack = stream?.getVideoTracks()[0];
    if (videoTrack) {
      // react-native-webrtc has _switchCamera method
      (videoTrack as any)._switchCamera();
    }
  }, []);

  return {
    startOutgoingCall,
    acceptIncomingCall,
    handleEndCall,
    toggleMute,
    toggleCamera,
    flipCamera,
  };
};
```

---

## INCOMING CALL DETECTION HOOK

```typescript
// src/hooks/useIncomingCall.ts
// Use this hook in the root navigator or ChatScreen

import { useEffect } from 'react';
import database from '@react-native-firebase/database';
import { CONFIG } from '../config/config';
import { useCallStore } from '../store/callStore';

export const useIncomingCall = () => {
  const { initCall, call } = useCallStore();

  useEffect(() => {
    // Only listen if not already in a call
    if (call.status !== 'idle') return;

    const ref = database().ref('calls/active');
    const handler = ref.on('value', (snap) => {
      const data = snap.val();
      if (
        data &&
        data.status === 'ringing' &&
        data.callerId !== CONFIG.USER_ME_UID
      ) {
        initCall(data.callType, 'incoming');
        // Navigation to IncomingCallScreen handled in AppNavigator
        // based on callStore.call.status === 'ringing' && direction === 'incoming'
      }
    });

    return () => ref.off('value', handler);
  }, [call.status]);
};
```

---

## VIDEO CALL FILTERS

```typescript
// Applied client-side using react-native-image-filter-kit
// Wrapped around the self-view RTCView only
// Partner sees filtered version (filter applied before WebRTC encodes)

// NOTE: react-native-image-filter-kit works on Image components.
// For live camera feed (RTCView), use a custom OpenGL approach:
// - react-native-gl-image-filters wrapping RTCView
// - OR: capture each frame, apply filter, re-stream

// For MVP: apply filter CSS-equivalent via RTCView style
// (post-processing on the rendered view, not on the stream)

const FILTER_STYLES = {
  Normal:   {},
  Warm:     { tintColor: 'rgba(255,200,100,0.15)' },  // approximate
  Cool:     { tintColor: 'rgba(100,150,255,0.15)' },
  'B&W':    {},   // requires grayscale shader
  Glow:     { opacity: 0.95 },
  Vintage:  { tintColor: 'rgba(180,120,60,0.2)' },
};
// Full implementation: use react-native-gl-image-filters for real-time shader
```

---

## IMPORTANT ANDROID CONFIG FOR WEBRTC

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-feature android:name="android.hardware.camera" />
<uses-feature android:name="android.hardware.camera.autofocus" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
<uses-permission android:name="android.permission.BLUETOOTH" />
```

```gradle
// android/app/build.gradle
android {
  defaultConfig {
    minSdkVersion 24  // react-native-webrtc requires 24+
  }
}
```

---

## CALL CLEANUP ON APP KILL

```typescript
// If app is killed during a call, the Realtime DB onDisconnect handles cleanup:

// Set in initPresence():
database().ref('calls/active').onDisconnect().remove();
// This ensures calls are cleaned up even if app crashes
```
