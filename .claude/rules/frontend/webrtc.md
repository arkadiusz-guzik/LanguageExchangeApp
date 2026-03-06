# WebRTC Rules

WebRTC peer connection logic is one of the most complex parts of this codebase. Follow these patterns exactly.

## Layering of WebRTC Code

```
shared/lib/webrtc/PeerConnection.ts    ← framework-agnostic engine (no React)
entities/call/lib/useWebRTC.ts         ← React hook wrapping the engine
features/webrtc-signaling/api/useSignaling.ts ← STOMP subscription + signaling flow
widgets/call-controls/ or pages/call-room/    ← where signaling hook is mounted
```

The `PeerConnection.ts` class has ZERO React imports. It is a plain TypeScript class. React hooks wrap it. This makes it testable in isolation.

---

## The PeerConnection Class

```typescript
// shared/lib/webrtc/PeerConnection.ts
import { ICE_SERVERS } from '@/shared/config/constants';

export type ConnectionStateCallback = (state: RTCPeerConnectionState) => void;
export type IceCandidateCallback = (candidate: RTCIceCandidate) => void;
export type TrackCallback = (stream: MediaStream) => void;

export class VoicePeerConnection {
  private pc: RTCPeerConnection;
  private participantId: string;

  constructor(participantId: string, config: RTCConfiguration = { iceServers: ICE_SERVERS }) {
    this.pc = new RTCPeerConnection(config);
    this.participantId = participantId;
  }

  async createOffer(): Promise<RTCSessionDescriptionInit> {
    const offer = await this.pc.createOffer();
    await this.pc.setLocalDescription(offer);
    return offer;
  }

  async createAnswer(): Promise<RTCSessionDescriptionInit> {
    const answer = await this.pc.createAnswer();
    await this.pc.setLocalDescription(answer);
    return answer;
  }

  async setRemoteDescription(sdp: RTCSessionDescriptionInit): Promise<void> {
    await this.pc.setRemoteDescription(new RTCSessionDescription(sdp));
  }

  async addIceCandidate(candidate: RTCIceCandidateInit): Promise<void> {
    await this.pc.addIceCandidate(new RTCIceCandidate(candidate));
  }

  addTrack(track: MediaStreamTrack, stream: MediaStream): void {
    this.pc.addTrack(track, stream);
  }

  onIceCandidate(cb: IceCandidateCallback): void {
    this.pc.onicecandidate = (e) => { if (e.candidate) cb(e.candidate); };
  }

  onConnectionStateChange(cb: ConnectionStateCallback): void {
    this.pc.onconnectionstatechange = () => cb(this.pc.connectionState);
  }

  onTrack(cb: TrackCallback): void {
    this.pc.ontrack = (e) => { if (e.streams[0]) cb(e.streams[0]); };
  }

  close(): void {
    this.pc.close();
  }

  get connectionState(): RTCPeerConnectionState {
    return this.pc.connectionState;
  }
}
```

---

## The Signaling Hook

```typescript
// features/webrtc-signaling/api/useSignaling.ts
import { useEffect, useRef } from 'react';
import type { StompSubscription } from '@stomp/stompjs';
import { subscribeToTopic, publishMessage } from '@/shared/api/stompClient';
import { STOMP_TOPICS, STOMP_DESTINATIONS } from '@/shared/config/constants';
import { VoicePeerConnection } from '@/shared/lib/webrtc/PeerConnection';
import { useStore } from '@/app/store';
import type { SignalingMessageDTO } from '@/shared/types/springTypes';

export function useSignaling(roomId: string, localStream: MediaStream | null) {
  const subsRef = useRef<StompSubscription[]>([]);
  const pcMapRef = useRef<Map<string, VoicePeerConnection>>(new Map());
  const addParticipantStream = useStore(s => s.addParticipantStream);
  const setCallStatus = useStore(s => s.setCallStatus);

  const getOrCreatePC = (participantId: string): VoicePeerConnection => {
    if (!pcMapRef.current.has(participantId)) {
      const pc = new VoicePeerConnection(participantId);
      
      pc.onConnectionStateChange((state) => {
        if (state === 'failed') setCallStatus('failed');
      });

      pc.onTrack((stream) => {
        addParticipantStream(participantId, stream);
      });

      pc.onIceCandidate((candidate) => {
        publishMessage(STOMP_DESTINATIONS.ICE, { toUserId: participantId, candidate });
      });

      // Add local tracks to the new connection
      if (localStream) {
        localStream.getTracks().forEach(track => pc.addTrack(track, localStream));
      }

      pcMapRef.current.set(participantId, pc);
    }
    return pcMapRef.current.get(participantId)!;
  };

  useEffect(() => {
    if (!roomId) return;

    // Subscribe to incoming SDP offer
    subsRef.current.push(
      subscribeToTopic<SignalingMessageDTO>(STOMP_TOPICS.OFFER, async (msg) => {
        const pc = getOrCreatePC(msg.fromUserId);
        await pc.setRemoteDescription(msg.sdp!);
        const answer = await pc.createAnswer();
        publishMessage(STOMP_DESTINATIONS.ANSWER, { toUserId: msg.fromUserId, sdp: answer });
      })
    );

    // Subscribe to incoming SDP answer
    subsRef.current.push(
      subscribeToTopic<SignalingMessageDTO>(STOMP_TOPICS.ANSWER, async (msg) => {
        const pc = pcMapRef.current.get(msg.fromUserId);
        if (pc) await pc.setRemoteDescription(msg.sdp!);
      })
    );

    // Subscribe to incoming ICE candidates
    subsRef.current.push(
      subscribeToTopic<SignalingMessageDTO>(STOMP_TOPICS.ICE, async (msg) => {
        const pc = pcMapRef.current.get(msg.fromUserId);
        if (pc && msg.candidate) await pc.addIceCandidate(msg.candidate);
      })
    );

    // Notify Spring Boot we joined
    publishMessage(STOMP_DESTINATIONS.JOIN, { roomId });
    setCallStatus('connecting');

    return () => {
      // Cleanup: unsubscribe, close all peer connections, notify server
      subsRef.current.forEach(sub => sub.unsubscribe());
      subsRef.current = [];
      pcMapRef.current.forEach(pc => pc.close());
      pcMapRef.current.clear();
      publishMessage(STOMP_DESTINATIONS.LEAVE, { roomId });
    };
  }, [roomId]);

  // When a new participant joins, we initiate the offer
  const initiateCallWithPeer = async (participantId: string) => {
    const pc = getOrCreatePC(participantId);
    const offer = await pc.createOffer();
    publishMessage(STOMP_DESTINATIONS.OFFER, { toUserId: participantId, sdp: offer });
  };

  return { initiateCallWithPeer };
}
```

---

## Local Media Stream

```typescript
// features/join-call/lib/getLocalStream.ts
export async function getLocalStream(constraints: MediaStreamConstraints = {
  audio: true,
  video: false,
}): Promise<MediaStream> {
  try {
    return await navigator.mediaDevices.getUserMedia(constraints);
  } catch (err) {
    if (err instanceof DOMException && err.name === 'NotAllowedError') {
      throw new Error('Microphone permission denied');
    }
    throw err;
  }
}
```

---

## Audio Level Analysis

```typescript
// entities/audio-stream/lib/analyzeAudioLevel.ts
export function createAudioLevelAnalyzer(
  stream: MediaStream,
  onLevel: (level: number) => void,  // 0-100
  intervalMs = 100
): () => void {
  const audioCtx = new AudioContext();
  const source = audioCtx.createMediaStreamSource(stream);
  const analyser = audioCtx.createAnalyser();
  analyser.fftSize = 256;
  source.connect(analyser);

  const dataArray = new Uint8Array(analyser.frequencyBinCount);
  const intervalId = setInterval(() => {
    analyser.getByteFrequencyData(dataArray);
    const avg = dataArray.reduce((a, b) => a + b, 0) / dataArray.length;
    onLevel(Math.round((avg / 255) * 100));
  }, intervalMs);

  // Return cleanup function
  return () => {
    clearInterval(intervalId);
    audioCtx.close();
  };
}
```

---

## WebRTC Rules Summary

- The `VoicePeerConnection` class lives in `shared/lib/webrtc/` — zero React
- React hooks wrapping it live in `entities/call/lib/` or `features/webrtc-signaling/api/`
- ALWAYS clean up: close peer connections and unsubscribe STOMP in `useEffect` return
- ALWAYS handle `failed` and `disconnected` ICE states — don't ignore them
- NEVER store `RTCPeerConnection` directly in Zustand — store it in a `useRef` or a `Map` in `useRef`, then put only the derived state (status, streams) in Zustand
- NEVER create peer connections outside of the signaling hook
- ICE candidates can arrive before setRemoteDescription — queue them if needed
