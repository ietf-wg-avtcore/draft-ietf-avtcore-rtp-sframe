---
title: "RTP Payload Format for SFrame"
docname: draft-ietf-avtcore-rtp-sframe-latest
category: std
date: {DATE}

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Audio/Video Transport Core Maintenance"
keyword: Internet-Draft
venue:
  group: "Audio/Video Transport Core Maintenance"
  type: "Working Group"
  mail: "avt@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/avt/"
  github: "ietf-wg-avtcore/draft-ietf-avtcore-rtp-sframe"
  latest: "https://ietf-wg-avtcore.github.io/draft-ietf-avtcore-rtp-sframe/draft-ietf-avtcore-rtp-sframe.html"

stand_alone: yes
pi: [toc, sortrefs, symrefs]
submissiontype: IETF

author:
 -
    ins: P. Thatcher
    name: Peter Thatcher
    organization: Microsoft
    email: pthatcher@microsoft.com
 -
    ins: Y. Fablet
    name: Youenn Fablet
    organization: Apple
    email: youenn@apple.com
 -
    ins: A. Rosenberg
    name: Aron Rosenberg
    organization: Apple
    email: aron.rosenberg@apple.com

normative:
  WebRTC_Encoded_Transform:
    author:
      org: World Wide Web Consortium
    title: WebRTC Encoded Transform
    date: 2025-05
    target: https://w3c.github.io/webrtc-encoded-transform/

informative:
  TestVectors:
    title: "SFrame Per SSRC Key Derivation Test Vectors"
    target: https://github.com/ietf-wg-avtcore/draft-ietf-avtcore-rtp-sframe/blob/main/test-vectors/test-vectors-per-ssrc-key-derivation.json
    date: 2026-10

--- abstract

This document describes the RTP payload format of SFrame, its use in SDP, and per stream key derivation.

--- middle

# Introduction

SFrame {{!RFC9605}} describes an end-to-end encryption and authentication mechanism
for media data in a multiparty conference call, in which central media servers (SFUs) can access the
media metadata needed to make forwarding decisions without having access to the actual media.

This document describes how to packetize a media frame encrypted using SFrame into RTP packets, how to
signal support in SDP, and how to derive per SSRC SFrame keys from a common key.

# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# SFrame format

An SFrame ciphertext comprises a header and encrypted data.
The SFrame header has a size varying between 1 and 17 bytes.
The encrypted data can be of arbitrary length and is larger than the unencrypted data by a fixed overhead that depends on the encryption algorithm.
The overhead can be up to 16 bytes.

As an SFrame ciphertext can be arbitrarily long, an application may decide to partition the data encrypted with SFrame
into pieces small enough that the SFrame ciphertext fits in a single RTP packet.
We call this per-packet SFrame.
This has the advantage of allowing the content to be decrypted as soon as it is received.

An alternative is to encrypt the data, a media frame typically, and send the SFrame ciphertext over several RTP packets.
We call this per-frame SFrame.
This has the advantage of limiting the SFrame overhead, especially for video frames.
This alternative is also compatible with {{WebRTC_Encoded_Transform}}, which is important for backward compatibility of existing services.

The RTP format presented in this document supports both alternatives.

# RTP Header Usage

The general RTP payload format for SFrame is depicted below.

~~~
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |V=2|P|X|  CC   |M|     PT      |       sequence number         |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                           timestamp                           |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |           synchronization source (SSRC) identifier            |
  +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
  |            contributing source (CSRC) identifiers             |
  |                             ....                              |
  +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
  |S E T x x x x x|                                               |
  |                                                               |
  :                       SFrame payload                          :
  |                                                               |
  |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                               :    OPTIONAL RTP padding       |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The first byte of the RTP payload is the SFrame payload descriptor.

The S bit of the SFrame payload descriptor MUST be 0 for all fragments except for the first one of the SFrame frame.

The E bit of the SFrame payload descriptor MUST be 0 for all fragments except for the last one of the SFrame frame.

An SFrame frame is protecting media content that can be either in a packetized format or in a raw format as defined by {{WebRTC_Encoded_Transform}}.
The T bit of the SFrame payload descriptor MUST be 0 if the fragment comes from a raw format as defined by {{WebRTC_Encoded_Transform}}.
The T bit of the SFrame payload descriptor MUST be 1 if the fragment comes from a packetized format.
All fragments of an SFrame frame MUST have the same T bit value.

The 5 remaining bits of the SFrame payload descriptor are reserved for future use.
Senders MUST set them to 0 and receivers MUST ignore them.

The payload type (PT) identifies the format of the media encrypted with SFrame.

The SSRC, timestamp, marker bit, and CSRCs of the SFrame RTP packets MUST be the same
as those of the output of the media-format-specific packetization.
The header extensions of the SFrame RTP packets SHOULD be the same
as those of the output of the media-format-specific packetization, but some may be omitted
if it is known that the omitted header extensions do not need to be duplicated on each SFrame RTP packet.

# RTP Processing of SFrame frames

## RTP Packetization of SFrame

For a given SFrame frame to be sent, the following steps are done:

1. Let content be the SFrame frame payload to be sent.
1. Let RTP metadata be the SFrame frame RTP metadata. It contains information such as payload type, timestamp, synchronization source, contributing sources, marker bit, header extensions, and payload origin.
1. Fragment the content in a list of payloads so that RTP packets generated from them do not exceed the network maximum transmission unit size.
1. For each payload in the list, run the following steps:
   1. Let descriptor be an SFrame payload descriptor, initialized to a zero byte.
   1. Set the first bit S of the descriptor to 1 if the payload is first in the list and 0 otherwise.
   1. Set the second bit E of the descriptor to 1 if the payload is last in the list and 0 otherwise.
   1. Set the third bit T of the descriptor to 1 if the SFrame frame payload origin is packetized, and 0 if it is raw.
   1. Prepend the payload with the descriptor.
   1. Let packet be an RTP packet with payload.
   1. Set the packet's RTP header, including header extensions, using the RTP metadata.
   1. In particular, if the SFrame frame marker bit is set, set the RTP header marker bit to the value of the second bit E of the descriptor.
   1. Send the packet.

SFrame packets can be generated either from RTP media packets or from media frames as defined by {{WebRTC_Encoded_Transform}}.
The two kinds of processing are presented below.

### Per-frame SFrame sending

For per-frame SFrame, the following steps are done on the sender side, with a media frame as input:

1. Generate a list of RTP packets from the media frame using the media-format-specific packetizer.
1. Let packet be the last RTP packet in the list.
1. Let payload be an SFrame ciphertext generated from the media frame data.
1. Let RTP metadata be an SFrame frame RTP metadata.
1. Set the SFrame RTP metadata payload origin to raw.
1. Set the SFrame RTP metadata marker bit if the packet has its marker bit set.
1. Set the SFrame RTP metadata other fields (timestamp, payload type, RTP header extensions...) using the packet RTP header.
1. Send the SFrame frame generated from payload and RTP metadata to the SFrame packetizer.

### Per-packet SFrame sending

For per-packet SFrame, the following steps are done on the sender side, with a media frame as input:

1. Generate a list of RTP packets from the media frame using the media-format-specific packetizer.
   The packetizer can be made aware of the SFrame overhead so that packet payloads do not get further split when doing SFrame packetization,
   to get a 1-1 match between the generated RTP packets and the SFrame encrypted sent RTP packets.
1. For each RTP packet in the list, run the following steps:
   1. Let payload be an SFrame ciphertext generated from the RTP packet payload.
   1. Let RTP metadata be an SFrame frame RTP metadata.
   1. Set the SFrame RTP metadata payload origin to packetized.
   1. Set the SFrame RTP metadata other fields using the packet RTP header, including the marker bit.
   1. Send the SFrame frame generated from payload and RTP metadata to the SFrame packetizer.

## RTP depacketization of SFrame {#rtp-depacketization-of-sframe}

When receiving an RTP SFrame packet, the following steps are done:

1. Add the packet to the set of received RTP packets.
1. Sort the set in order of the RTP sequence number.
1. Search for the smallest list of RTP packets in the set such that:
   1. The packets are consecutive in sequence number.
   1. The first packet of the list has its S bit set to 1.
   1. The last packet of the list has its E bit set to 1.
1. If there is no such list, abort these steps.
1. Remove the selected RTP packets from the set of received RTP packets.
1. If the T bits of each RTP packet of the list do not have the same value, abort these steps.
1. If the payload types of each RTP packet of the list do not have the same value, abort these steps.
1. For each RTP packet of the list, remove the first byte of the packet's payload.
1. Let the SFrame payload be the concatenation of the payloads of each RTP packet of the list.
1. Let the SFrame RTP metadata be computed from the RTP headers of the RTP packets of the list, in particular:
   1. The RTP metadata's marker bit is set if the marker bit of the last RTP packet of the list is set.
   1. The RTP metadata's payload origin is packetized if the T bit value is 1 and raw otherwise.
1. Decrypt the SFrame payload to obtain the media decrypted data. If decryption fails, abort these steps.
1. If the frame RTP metadata payload origin is raw, i.e., the T bits of each RTP packet of the list are 0, send the media frame created from the media decrypted data and SFrame RTP metadata to the receiving pipeline.
1. Otherwise, run the following sub-steps:
   1. Let depacketizer be a media-format-specific depacketizer selected from the frame payload type.
   1. Send the media decrypted data and SFrame RTP metadata to the depacketizer.
   1. If the depacketizer generates a media frame, send the media frame to the receiving pipeline.

Errors like inconsistent T bits, inconsistent payload types or decryption failures cause depacketization to fail and the corresponding frame to be discarded.
The receiver SHOULD treat a discarded frame as a lost/corrupted frame and rely on its normal recovery mechanisms (e.g., PLI/FIR for video) even though no RTP packet was lost.

## SFrame metadata {#sframe-metadata}

The SFrame AEAD encryption and decryption operations of {{!RFC9605}} take a `metadata` input, which is authenticated as part of the AAD along with the SFrame header but is not transmitted in the SFrame ciphertext.
This metadata is unrelated to the RTP metadata used in the packetization algorithms above.

By default, the `metadata` input for the SFrame encryption and decryption operations is empty.
RTP header fields and header extension values can be modified by RTP middleboxes such as SFUs, and some are not identical across the RTP packets of a frame.
Applications that want to use a non-empty SFrame metadata need to define how it is made available to both sender and receiver out of band of the RTP headers.

# SFrame SDP negotiation {#sframe-sdp-negotiation}

SFrame packetization is indicated via a new "a=sframe" SDP attribute defined in this specification.
This attribute is used at media level, it does not appear at session level.

The presence of the "a=sframe" attribute in a media section indicates that the media carried by the RTP packets of the media section is encrypted with SFrame in both directions.
An answerer that supports SFrame includes the attribute in its answer only if it was present in the offer.
An answerer that requires SFrame MUST reject m-sections that do not have the "a=sframe" SDP attribute.
If the offer contains "a=sframe" and the answer does not, the offerer MUST NOT send media for that m-section and MUST NOT fall back to sending unencrypted RTP.
The offerer MUST also discard any RTP received for that m-section and SHOULD close it in subsequent offers.
If an m-section has the "a=sframe" SDP attribute in an offer, it MUST appear in any reoffer unless the m-section is closed.

When SFrame is in use for that media section, it will apply to the relevant media encodings defined for that media section.
This includes RTP payload types bound to media packetizers and media depacketizers as defined in {{!RFC7656}}, typically audio formats such as Opus and RTP video formats such as H264.
This notably includes RTP payload types representing {{WebRTC_Encoded_Transform}} [encoded video frame formats](https://w3c.github.io/webrtc-encoded-transform/#dom-rtcencodedvideoframe-data) and [encoded audio frame formats](https://w3c.github.io/webrtc-encoded-transform/#dom-rtcencodedaudioframe-data).

This does not include RTP-Based Redundancy mechanisms as defined in {{!RFC7656}}.
For instance, RTX defined in {{!RFC4588}} will retransmit SFrame-based packets.
Forward error correction formats as defined in {{!RFC5109}} will protect the encrypted content.
For Redundant Audio Data, known as RED, as defined in {{!RFC2198}}, a RED packetizer will take as input SFrame encrypted media data instead of unencrypted media data.

If BUNDLE is in use and the "a=sframe" attribute is present for a media section but not for another media section of the same BUNDLE,
payload types for media encodings that are relevant for SFrame MUST NOT be reused between the two media sections.

Here is an example of SFrame being negotiated for audio (opus and CN) and for video (H264 and VP8):

~~~
m=audio 50000 RTP/SAVPF 10 11
a=sframe
a=rtpmap:10 opus/48000/2
a=rtpmap:11 CN/8000

m=video 50002 RTP/SAVPF 100 101
a=sframe
a=rtpmap:100 H264/90000
a=rtpmap:101 VP8/90000
~~~

# SFrame SDP RTP Key Management {#sframe-sdp-rtp-key-management}

When SFrame is used with SDP and specifies RTP as the media transport, an additional key derivation step MAY be applied to produce a unique key per SSRC for a m= line.
The resulting per SSRC stream key is used as the initial key in the session for that SSRC and becomes the input to the SFrame `derive_key_salt` function.

This initial derivation step starts with the `base_key` of the session. Then, each SSRC stream involved in the SDP session, MUST perform this derivation step to produce
the initial SFrame `ssrc_key` for that stream in the SDP session. This step is performed using HMAC-based Key Derivation Function (HKDF) {{!RFC5869}} as follows:

~~~
ssrc_key = HKDF-Expand(HKDF-Extract(SSRC, base_key),
                       "SFrame 1.0 RTP Stream", CipherSuite.Nh)
~~~

In the derivation of ssrc_key:

* The SSRC is encoded as a 4-byte big-endian byte sequence and passed as the salt value to HKDF-Extract.

* The same CipherSuite is used for this step as for the per packet / frame step the resulting key will be used with.

# SFrame Ratcheting with per SSRC keys

If ratcheting is used with this per SSRC key derivation algorithm, the per SSRC key derivation step MUST be done once at the start of the session, and then each subsequent
ratchet is done on the per SSRC specific keys using the SFrame ratcheting algorithm described in section 5.1 of {{!RFC9605}}.
In this per SSRC key derivation algorithm, the same algorithm defined in section 5.1 of {{!RFC9605}} is used to produce the KID from the key_generation and ratchet_steps inputs. This results in all streams using the same KID, but different keys.
Each stream maintains its own CTR value, so the same CTR value can be used by several streams.
Since the keys are different, this does not result in nonce reuse.

This results in a key tree that looks like the following for an offer that has three different SSRC's a, b, and c.

~~~
+--------+
|base_key|
+---+----+ Initial
    |     Derivation       Ratchet 1               Ratchet N
    |    +----------+   +-------------+         +-------------+
    |--->|ssrc_key_a|-->|ssrc_key_a[1]|-->...-->|ssrc_key_a[N]|
    |    +----------+   +-------------+         +-------------+
    |    +----------+   +-------------+         +-------------+
    |--->|ssrc_key_b|-->|ssrc_key_b[1]|-->...-->|ssrc_key_b[N]|
    |    +----------+   +-------------+         +-------------+
    |    +----------+   +-------------+         +-------------+
    +--->|ssrc_key_c|-->|ssrc_key_c[1]|-->...-->|ssrc_key_c[N]|
         +----------+   +-------------+         +-------------+
~~~

# Security Considerations

This document is subject to the security considerations of SFrame {{!RFC9605}}.
How keys are established and distributed between endpoints is out of scope of this document.

## Scope of SFrame Protection {#scope-of-sframe-protection}

SFrame provides end-to-end confidentiality and integrity of the media payload only.
The RTP header, RTP header extensions, CSRCs and the SFrame payload descriptor are not protected end-to-end by SFrame.
RTCP is not protected by SFrame.
Hop-by-hop protection mechanisms, for instance SRTP {{!RFC3711}} and cryptex {{?RFC9335}}, are still needed to protect these fields from attackers on the network path.
SRTP alone does not encrypt RTP header extensions, which can be done using cryptex.
These mechanisms are terminated at each middlebox, which can still access and modify these fields.

## Unauthenticated SFrame Payload Descriptor

The S, E and T bits of the SFrame payload descriptor are not covered by the SFrame authentication.
A middlebox modifying the S or E bits can prevent a frame from being reassembled, or can cause RTP packets of different frames to be reassembled together.
In the latter case, SFrame decryption fails and the frame is discarded, which is a denial of service but does not compromise the media content.

A middlebox modifying the T bit of all packets of a frame does not cause SFrame decryption to fail.
The authentic decrypted data is then processed either as a raw media frame or as a packetized payload, depending on the T bit value chosen by the middlebox.
Receivers cannot expect that the decrypted data is well-formed for the processing selected by the T bit and need to handle it as robustly as data received from the network.
Applications that need to prevent this can bind the payload origin to the SFrame ciphertext by including it in the SFrame `metadata` input, as described in {{sframe-metadata}}.

## Payload Type Confusion

The payload type is not authenticated by SFrame and can be legitimately rewritten by middleboxes such as SFUs, which may use different payload type mappings on each leg.
A malicious middlebox can thus cause authentic decrypted media to be processed by a depacketizer or decoder of a different media format than the one used by the sender.
Media depacketizers and decoders are known to be a significant attack surface.
Receivers cannot expect that the decrypted data matches the media format identified by the payload type and need to handle it as robustly as data received from the network.

## Metadata Exposed to Middleboxes

By design, SFrame exposes metadata to middleboxes so that they can make forwarding decisions.
This includes the RTP header, the RTP header extensions, the SFrame header, the SFrame payload descriptor and the RTP packet sizes.
This information can reveal frame boundaries, frame sizes, key frames, video layer structure, and in the case of audio, voice activity, for instance via the audio level header extension {{?RFC6464}}.
These leaks are potentially larger with per-packet SFrame, where the SFrame boundaries follow the media-format-specific packetization.
Senders are expected to only send the header extensions that are needed by the middleboxes and the receivers to reduce privacy risks.
RTP padding can be used to reduce the information leaked by packet sizes.

## Per SSRC Key Derivation

The SFrame encryption nonce is derived from the key and the CTR value.
If several RTP streams are encrypted with the same key and KID while maintaining independent CTR values, the same nonce will be used with the same key for different content, which breaks the confidentiality and integrity guarantees of the AEAD algorithms used by SFrame.
The per SSRC key derivation defined in {{sframe-sdp-rtp-key-management}} prevents this by producing a different key for each SSRC.

The per SSRC key derivation relies on the uniqueness of the SSRC among the RTP streams using the same base_key.
If two senders use the same base_key and pick the same SSRC, they will derive the same ssrc_key and may reuse nonces.
A base_key is not expected to be shared between senders unless the application guarantees that their SSRCs are distinct.
When an SSRC changes, for instance after an SSRC collision as described in {{!RFC3550}}, a new ssrc_key is derived from the new SSRC and the CTR value can be reset.

The SSRC used for the derivation is the SSRC chosen by the sender.
A middlebox that rewrites SSRCs, as is commonly done by SFUs when forwarding simulcast streams, prevents receivers from deriving the correct key, unless the receivers know the original SSRC through other means.
In that case, decryption fails, which is a denial of service but does not compromise the media content.
Similarly, a middlebox swapping the SSRCs of two RTP streams causes decryption failures instead of media of one stream being presented as media of the other stream.

When the per SSRC key derivation is not used, for instance because SSRCs are not preserved end-to-end as described above, the application ensures that each RTP stream encrypted with the same base_key uses distinct KID values, or that the streams share a single counter.
Since the SFrame key and salt are derived from both the base_key and the KID {{!RFC9605}}, using a distinct KID for each RTP stream results in distinct keys and salts for each stream, even if all streams share the same base_key.
The KID is part of the SFrame header, which is authenticated by SFrame, and is therefore not affected by middleboxes rewriting RTP header fields.
The KID allocation scheme needs to guarantee that the KID values used by the RTP streams of all senders sharing a base_key are distinct.

Retransmission packets as defined in {{!RFC4588}} carry SFrame packets that were encrypted with the key derived from the original SSRC, not from the retransmission SSRC.

Sender and receiver need to agree, using an out-of-band mechanism, on whether the per SSRC key derivation is used.
A mismatch results in decryption failures for all media.

## Downgrade

An attacker able to modify the SDP can remove the "a=sframe" attribute.
The negotiation rules of {{sframe-sdp-negotiation}} prevent an endpoint that offered SFrame from sending unencrypted media in that case, but they do not prevent an attacker from removing "a=sframe" from an offer.
In many deployments, the signaling server is operated by the same entity as the SFU, which is the entity SFrame protects against.
Applications that require end-to-end encryption cannot rely solely on the SDP negotiation to decide whether to use SFrame.
Instead, they have to enforce SFrame usage based on their own policy.

## Resource Exhaustion

The depacketization algorithm of {{rtp-depacketization-of-sframe}} buffers RTP packets until a complete SFrame frame is received.
An attacker sending packets that never complete a frame can cause unbounded memory use.
Receivers are expected to bound the number of buffered packets and discard incomplete frames after a limited amount of time, for instance when newer frames have been fully received.

# IANA Considerations

This document updates the "attribute-name (formerly "att-field")" subregistry of the "Session Description Protocol (SDP) Parameters" registry (see Section 8.2.4 of {{!RFC8866}}).
Specifically, it adds the SDP "a=sframe" attribute for use at the media level.

Contact name: IETF AVT Working Group

Contact email address: avt@ietf.org

Attribute name: sframe

Attribute syntax: This attribute takes no values

Attribute semantics: N/A

Attribute value: N/A

Usage Level: media

Charset dependent: No

Purpose: The presence of this attribute in the SDP indicates that the media carried by the RTP packets of the m-section is encrypted by SFrame

O/A procedures: SDP O/A procedures are defined in section 6

Mux Category: NORMAL

--- back

# Test Vectors

This section provides a set of test vectors that implementations can use to
verify that they correctly implement SFrame per SSRC stream key derivation,
and ratcheting of the resulting per SSRC keys, with registered SFrame cipher
suites.

For each SSRC, `ssrc_key_a` is the initial key produced by the derivation in
{{sframe-sdp-rtp-key-management}}.  `ssrc_key_a_1` and `ssrc_key_a_2` are the
keys after the first and second ratchet steps described in
{{sframe-ratcheting-with-per-ssrc-keys}}, corresponding to `ssrc_key_a[1]`
and `ssrc_key_a[N]` with N = 2 in the key tree.  The values for `ssrc_key_b` follow the
same pattern.

All values are either numeric or byte strings.  Numeric values are represented
as hex values, prefixed with `0x`.  Byte strings are represented in hex
encoding.

Line breaks and whitespace within values are inserted to conform to the width
requirements of the RFC format.  They should be removed before use.

These test vectors are also available in JSON format at {{TestVectors}}.  In the
JSON test vectors, numeric values are JSON numbers and byte string values are
JSON strings containing the hex encoding of the byte strings.

{::include test-vectors/sframe-per-ssrc-key-derivation.md}
