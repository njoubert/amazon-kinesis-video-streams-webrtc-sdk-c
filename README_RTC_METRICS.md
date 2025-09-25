# WebRTC Peer Connection Metrics Guide

This guide explains how to use `rtcPeerConnectionGetMetrics` in the Amazon Kinesis Video Streams WebRTC SDK to monitor connection health and quality. It summarizes the metrics that are exposed, how frequently they are updated, and offers practical guidance for detecting connection loss and packet loss conditions.

## Collecting Metrics

`rtcPeerConnectionGetMetrics` takes an initialized `RtcStats` structure whose `requestedTypeOfStats` field is set to one of the `RTC_STATS_TYPE` values. On success, the call populates the corresponding member of the `RtcStatsObject` union and timestamps the sample with `GETTIME()` in 100 ns units.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Include.h†L1584-L2205】【F:src/source/Metrics/Metrics.c†L228-L286】 The API does not run on its own; you choose how often to poll it from your application, usually on a timer that matches how quickly you need to react to state changes.

Internally, metrics are updated as packets are processed:

* Inbound RTP counters (packet counts, jitter, timestamps) are refreshed whenever packets are received and decrypted.【F:src/source/PeerConnection/PeerConnection.c†L224-L311】【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L471-L509】
* Remote inbound statistics such as `fractionLost` and `roundTripTime` are updated each time a Receiver Report is parsed from the remote peer.【F:src/source/PeerConnection/Rtcp.c†L100-L142】【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L401-L417】
* ICE candidate/candidate pair data are filled from the ICE agent’s diagnostics cache when requested.【F:src/source/Metrics/Metrics.c†L8-L118】

### Expected Freshness of Data

* **Outbound RTCP sender reports:** Each transceiver schedules its first report ~3 seconds after it starts, then reschedules future reports every 100–300 ms (200 ms ±100 ms jitter).【F:src/source/PeerConnection/PeerConnection.c†L733-L806】【F:src/source/Rtcp/RtcpPacket.h†L35-L38】 These reports drive the remote peer’s packet loss calculations.
* **Inbound RTP statistics:** Updated immediately as packets arrive; if the remote stops sending, the last update is the time of the final packet.
* **Remote inbound statistics:** Updated whenever the remote sends Receiver Reports. The cadence depends on the remote peer’s RTCP policy, but with the SDK default schedule above you can expect new loss/RTT samples roughly every 100–300 ms.

Because `rtcPeerConnectionGetMetrics` simply returns the latest snapshot, call it often enough (for example every 200 ms) to detect timeouts or loss events within your desired thresholds.

## Metrics by `RTC_STATS_TYPE`

| Stats Type | Structure | Key Fields | Notes |
|------------|-----------|------------|-------|
| `RTC_STATS_TYPE_CANDIDATE_PAIR` | `RtcIceCandidatePairStats` | Packet/byte counters, round-trip time, selected pair state | Current nominated ICE pair used for media.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L220-L292】 Useful for monitoring transport bytes and detecting network path switches. |
| `RTC_STATS_TYPE_LOCAL_CANDIDATE` / `RTC_STATS_TYPE_REMOTE_CANDIDATE` | `RtcIceCandidateStats` | Candidate address, protocol, type | Reports the most recently selected local or remote ICE candidate details.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L197-L218】 |
| `RTC_STATS_TYPE_ICE_SERVER` | `RtcIceServerStats` | Requests sent, responses received, total round-trip time | One entry per configured ICE server; must set `iceServerIndex` before calling.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L176-L194】 |
| `RTC_STATS_TYPE_INBOUND_RTP` | `RtcInboundRtpStreamStats` | `packetsReceived`, `packetsDiscarded`, `lastPacketReceivedTimestamp`, `bytesReceived`, `jitter` | Represents packets received from the remote peer after jitter buffering.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L450-L509】 |
| `RTC_STATS_TYPE_OUTBOUND_RTP` | `RtcOutboundRtpStreamStats` | `packetsSent`, `bytesSent`, `packetsDiscardedOnSend`, retransmission counters, encoding info | Represents packets sent to the remote, per SSRC.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L317-L383】 |
| `RTC_STATS_TYPE_REMOTE_INBOUND_RTP` | `RtcRemoteInboundRtpStreamStats` | `fractionLost`, `roundTripTime`, `reportsReceived` | Remote peer’s view of the SSRC you are sending. Use this for end-to-end packet loss as reported by the far end.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L401-L417】 |
| `RTC_STATS_TYPE_DATA_CHANNEL` | `RtcDataChannelStats` | `messagesSent/Received`, `bytesSent/Received`, `state` | Aggregated per SCTP data channel id.【F:src/source/Metrics/Metrics.c†L202-L226】 |

Additional stats types defined in the WebRTC specification (codec, track, peer connection, etc.) are not yet implemented and will return `STATUS_NOT_IMPLEMENTED` if requested.【F:src/source/Metrics/Metrics.c†L248-L286】

## Monitoring Connection Aliveness

1. Poll `RTC_STATS_TYPE_INBOUND_RTP` for each receiving transceiver. Compare `pRtcMetrics->timestamp` with `inboundStats.lastPacketReceivedTimestamp` (in milliseconds). If the difference exceeds your 1 second threshold, report the connection as stalled for that media stream. The timestamp is refreshed on every packet, so it is sensitive to real media gaps.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L495-L509】【F:src/source/PeerConnection/PeerConnection.c†L224-L311】
2. Also monitor `RtcIceCandidatePairStats.lastPacketReceivedTimestamp`. If both inbound RTP and candidate pair timestamps stop moving, the transport path is idle and the peer is effectively disconnected.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L231-L244】
3. Consider integrating signaling callbacks (e.g., `peerConnectionOnConnectionStateChange`) for immediate state transitions, using metrics as a secondary guard for stream-level stalls.

### Example Pseudocode

```c
RtcStats stats = {.requestedTypeOfStats = RTC_STATS_TYPE_INBOUND_RTP};
CHK_STATUS(rtcPeerConnectionGetMetrics(pc, NULL, &stats));
UINT64 nowMs = KVS_CONVERT_TIMESCALE(stats.timestamp, HUNDREDS_OF_NANOS_IN_A_MILLISECOND, 1);
UINT64 lastMediaMs = stats.rtcStatsObject.inboundRtpStreamStats.lastPacketReceivedTimestamp;
if (nowMs - lastMediaMs > 1000) {
    reportConnectionLost();
}
```

## Monitoring Packet Loss

### Remote View (recommended)

* Request `RTC_STATS_TYPE_REMOTE_INBOUND_RTP`. `fractionLost` represents the remote peer’s current estimate of the fraction of packets lost since the last report (0.0 – 1.0). Multiply by 100 to convert to a percentage. Because Receiver Reports arrive roughly every 100–300 ms, average the values over your 2 second evaluation window. If the mean exceeds 0.5, trigger your alert.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L401-L417】【F:src/source/PeerConnection/Rtcp.c†L100-L142】
* You can also compute loss rate over time by tracking `reportsReceived` and comparing successive `fractionLost` samples.

### Local View

* For inbound media, compute `(expected - packetsReceived)` using `RtcInboundRtpStreamStats.received.packetsReceived` together with the remote sender’s sequence numbers if you track them, or rely on jitter-buffer discard counters (`packetsDiscarded`) to detect drops.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L450-L489】
* For outbound media, monitor `RtcOutboundRtpStreamStats.packetsDiscardedOnSend`, `retransmittedPacketsSent`, and `nackCount`. Increases indicate that the network is struggling even if the far end has not yet reported loss.【F:src/include/com/amazonaws/kinesis/video/webrtcclient/Stats.h†L317-L383】

### Sliding Window Strategy

1. Sample `RtcRemoteInboundRtpStreamStats` every ~200 ms and push `fractionLost` into a ring buffer sized for 10 samples (≈2 s).  
2. Compute the average or maximum loss over the buffer. Trigger an alert if any sample exceeds 0.5 or if the average exceeds 0.5, depending on your policy.  
3. Optionally cross-validate with outbound retransmission counters to avoid false positives during ramp-up.

## Putting It Together

* Establish a repeating timer (e.g., 200 ms) that polls both inbound RTP and remote inbound RTP stats for each media transceiver.  
* Track timestamps to detect media stalls >1 s.  
* Track loss ratios over a 2 s sliding window using `fractionLost`, backed up by retransmission counters and jitter metrics to diagnose root cause.  
* Use candidate pair stats to log total bytes and spot transport switches; use ICE server stats to monitor TURN/STUN health if connectivity degrades.  
* Because metrics are updated by real-time packet processing, alerts will closely follow actual network conditions as long as your polling interval is shorter than the thresholds you care about.

By combining these metrics with application-level timers you can implement robust monitoring for both aliveness and media quality on top of the Amazon Kinesis Video Streams WebRTC SDK.
