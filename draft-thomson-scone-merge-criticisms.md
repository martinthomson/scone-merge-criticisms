---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "A comparative analysis of SCONE relative to TRAIN"
abbrev: "SCONE Criticisms"
category: info

docname: draft-todo-yourname-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Your Name Here
    organization: Mozilla
    email: your.email@example.com

normative:

informative:
  SCONE: I-D.joras-scone-quic-protocol
  TRAIN: I-D.thomson-scone-train-protocol


--- abstract

A merge of the rate availability signaling schemes in the SCONE and TRAIN proposals is analysed and found to be infeasible.

--- middle

# Introduction

There are a few items where we might need to discuss the advantages of SCONE {{SCONE}}, but overall there are not a lot of advantages to the proposed design over the one that TRAIN {{TRAIN}} proposes.

The idea that these designs might be merged does not make a lot of sense. The two designs are largely incompatible, as each relies on a fundamentally different mechanism.

The one major question about the nature of the signal is separable from a decision about how to design the signaling mechanism.

# Communication Model

TRAIN is symmetric.  SCONE is initiated by a client only.

TRAIN is negotiated as part of a connection.  SCONE is unilaterally implemented by clients.

TRAIN endpoints coalesce a carriage packet with other packets, network elements modify the payload of that carriage.

SCONE requires that clients announce their willingness to receive signals to network elements.  Thereafter network elements can send a signal any time to that client.

## Analysis

Like ECN, TRAIN signals follow the flow in the direction in which it might be affected. SCONE signals flow in the reverse direction, which means that if flows take a different path in each direction, the wrong set of network elements are involved.

SCONE packets have far weaker authentication. Once a packet is captured, any entity can generate a packet that will be accepted.  In comparison, a TRAIN packet is carried along with real packets and so is only good for approximately as long as it takes to get to its destination.  Spoofing requires that the packet be raced.

The requirement for negotiation in TRAIN makes it a tiny bit more complex.  However, it also means that both client and server agree to its use before it is used.  Negotiation is necessary to enable coalescing in TRAIN.  Whether this is an advantage to TRAIN or SCONE seems subjective.

Arguably that a network element can send an updated signal at any time is an advantage for SCONE.

# Signal

TRAIN presently proposes a signal that carries a choice from an (as-yet-unspecified) selection of 64 options.  That’s not a lot.

SCONE has a 32-bit rate (in kbps) and a window (in ms), which is massively more flexible.

## Analysis

This is not a serious difference.  TRAIN could change to accommodate a richer signal, but that comes with costs, see below.

# Network Element Processing

TRAIN requires processing similar to ECN: network elements recognize opportunities and they flip a few bits.  This is designed to be as simple as possible to process, such that it could be done easily at line rate. This is a strong reason to limit the expressiveness of the signal: a more complex signal would be harder to apply.

SCONE involves setting up a context at a network element, with the network element sending a signal any time it chooses.  Generating the signal requires the use of an AEAD.  In practice, this is likely a few packets for new flows, then a few any time the flow characteristics at the element change.

## Analysis

SCONE requires that network elements generate new packets, including spoofing the source IP address.  TRAIN only depends on packet modification.

SCONE requires that network elements remember an address tuple and client connection ID so that they can provide updates.  TRAIN can be processed without any state.

{:aside}
> Elements might benefit from recognizing a flow as QUIC and remembering it to manage [this issue](https://github.com/martinthomson/train-protocol/issues/32), that issue might be resolved by a small protocol change instead.

SCONE exposes network elements to weird attacks if they don’t maintain additional state.  They can be sent SCONE packets without an associated QUIC flow, which might cause the network element to spend effort on sending packets to a spoofed source address.  Given spoofed SCONE packets, those generated packets go to a destination of the attacker’s choice.  The time frame over which those packets are likely spread means that this is unlikely to be a significant amplification in terms of packet rate.  That might make DoS using this vulnerability less appealing, but it is worth noting as it has no upper bound in quantity.

# Miscellaneous SCONE Problems

There are a few things in SCONE that probably could be improved.  None of these really have a bearing on the overall solution.

## Packet Protection

SCONE defines packet protection for the packet that a client sends.  Network elements do not need to authenticate this – the information they need from that packet is not protected – but that’s not obvious from the specification.

Protection of signals from network elements provides no meaningful security value.  The value is in resilience to accidentally interpreting garbage packets as a signal and the AEAD is definitely better than the UDP checksum at detecting transmission errors.  However, those same benefits might be realized more cheaply by other means.

## Forwarding Bit

SCONE packets include a forward bit, which – if set – requests that network elements drop the packet. This requires exceptional processing by a network element, other than simply recognizing the packet.  That requirement might take all SCONE packets of the fast path in elements.

It seems like the point of this is to enable the targeting of a first-hop network element.  However, this assumes that the application of rate limits is done by something on that first network segment.  That’s true in a lot of cases, but this is a bad assumption.  The same might be more easily achieved by altering the IP TTL instead, with greater targeting precision.

Alternatively, this feature might be used to probe whether a path supports SCONE: if a forward=0 packet makes it to the other end, it might feed that information back.  Two things though:

* This feature on its own probably wouldn’t be enough to justify a server implementation.
* Network elements aren’t required to drop these packets.

TRAIN gets this capability for free.  An endpoint that receives an unmodified TRAIN packet might infer that the path doesn’t support TRAIN.

## Source Connection ID on Signals

SCONE insists on a random source connection ID from network elements.  To start with, this might cause things to drop the signals if they expect a particular source connection ID.  Also, as there is no ongoing communication, this could just swap source and destination connection ID from the original. That would strengthen protection against spoofing (many clients do not use connection IDs for QUIC packets they receive) at a small cost in maintained state.  However, any spoofing protection thus gained would not be assured, as there is no minimum entropy for connection IDs in either direction.

# Security Considerations

Requiring this section is perhaps no longer sensible, when due consideration is given to the topic of security throughout, as is the case with this document.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

Christian Huitema provided useful feedback, but does not necessarily endorse its message.
