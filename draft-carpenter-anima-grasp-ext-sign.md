---
title: GRASP Flood Signing Extension
abbrev: GRASP Flood Signing
docname: draft-carpenter-anima-grasp-ext-sign-latest

# stand_alone: true

ipr: trust200902
area: Ops
wg: ANIMA
kw: Internet-Draft
cat: std
updates: 8990


pi:
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:
      -
        ins: C. Bormann
        name: Carsten Bormann
        org: TZI
        abbrev: TZI
        postalLine: Germany
        email: cabo@tzi.org

      -
        ins: B. E. Carpenter
        name: Brian E. Carpenter
        org: The University of Auckland
        abbrev: Univ. of Auckland
        postalLine: School of Computer Science
        postalLine: The University of Auckland
        postalLine: PB 92019
        postalLine: Auckland 1142
        postalLine: New Zealand
        email: brian.e.carpenter@gmail.com

      -
        ins: T. Eckert
        name: Toerless Eckert
        org: Futurewei
        abbrev: Futurewei
        postalLine: USA
        email: tte@cs.fau.de

      -
        ins: M. Richardson
        name: Michael Richardson
        org: Sandelman
        abbrev: Sandelman
        postalLine: Canada
        email: mcr@sandelman.ca

normative:
  RFC8990:
  RFC8610:
  RFC8152:
  RFC8949:

informative:
  RFC8993:
  RFC9222:

--- abstract

This document clarifies how message formats for the GeneRic Autonomic Signaling Protocol (GRASP) defined by RFC 8990 may be updated by adding new options. It also describes one such option for cryptographically signing an M_FLOOD message.

--- middle

# Introduction        {#intro}

The GeneRic Autonomic Signaling Protocol (GRASP) is specified in {{RFC8990}}.
For the general model of an autonomic network, and for terminology not otherwise
defined here or in RFC 8990, see {{RFC8993}}.

One important feature of GRASP is its flooding mechanism, the M_FLOOD message,
which distributes GRASP objectives (as defined in RFC 8990)
to all GRASP nodes within range. Such messages are sent as multicasts,
which unlike GRASP unicast messages, cannot be protected
with TLS. To mitigate any risk of malicious M_FLOOD messages, this document
specifies a method of cryptographically signing such messages.

Preparing this specification revealed a weakness in the way RFC 8990
describes extensibility by the addition of new options to GRASP messages.
This document therefore starts by clarifying this aspect of RFC 8990.

# Terminology

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all capitals, as shown here.

# Extensibility of GRASP message formats

Although RFC 8990 defines a finite set of GRASP message types and their contents, it also implies that extensions are expected. In particular, the general definition of the message format (Section 2.8.2 of {{RFC8990}}) states that:

<blockquote>
The MESSAGE_TYPE indicates the type of the message and thus defines the expected options. Any options received that are not consistent with the MESSAGE_TYPE SHOULD be silently discarded.
</blockquote>

The general structure of a GRASP message is an integer MESSAGE_TYPE followed by various elements, some of which are options in the form \[OPTION_TYPE, ...]. The intention of the quoted statement was to allow new options to be defined without invalidating existing implementations. However, the formal CDDL definitions in {{RFC8610}}  of the various individual message types do not include explicit extension points where new options could be included. Thus, the implementer of a message decoder that adheres strictly to the CDDL might not implement the silent discard, because an unexpected option would violate the CDDL definition and trigger a decoding error. This creates a risk of interoperability failure when new options are introduced.

This document revises the CDDL definition of GRASP to clarify this point. Specifically, the CDDL now starts as follows:

~~~
grasp-message = (message .within message-structure) / noop-message
message-structure = [MESSAGE_TYPE, session-id, ?initiator,
                      *grasp-element]
grasp-element = (option .within option-structure) / objective / any
option-structure = [OPTION_TYPE, *any]
~~~

This allows future extensions to be made by specifying elements in the two "any" components shown. The complete revised CDDL definition is given in {{grasp-cddl}}. Consistent with RFC 8990, implementations **SHOULD** silently discard unknown options, with **OPTIONAL** logging for diagnostic purposes.

It is not expected that the basic GRASP message formats and options will require frequent extensions. In general, extended capabilities **SHOULD** be created by designing suitable GRASP objectives, as defined in Section 2.10 of {{RFC8990}} and discussed in {{RFC9222}}.

# Format of the GRASP M_FLOOD Signature Option

The formal definition of the GRASP M_FLOOD message is extended as follows, defined
in fragmentary CDDL {{RFC8610}}.

The statement given in {{RFC8990}}:

~~~~
    flood-message = [M_FLOOD, session-id, initiator, ttl,
                 +[objective, (locator-option / [])]]
~~~~

is replaced by:

~~~~
   flood-message = [M_FLOOD, session-id, initiator, ttl,
                 +[objective, (locator-option / [])], ?sign-option]

   sign-option = [O_COSE_SIGN, bytes] ; see text description
   O_COSE_SIGN = 107
~~~~

Note that signed M_FLOOD messages are relayed exactly as desccribed in
Section 2.5.6.2 of {{RFC8990}}, including decrementing the loop count,
which is therefore excluded from the following signature process.

The bytestring content of the sign-option is a COSE signature with a detached
payload, as described in {{RFC8152}}. The payload that is signed is a copy
of the entire flood-message contents encoded as CBOR {{RFC8949}}, but
without the sign-option component, and with the loop-count component of the first
GRASP objective replaced by zero.

In more detail, a node that sends a signed M_FLOOD proceeds as follows:

1. Build the M_FLOOD message with no signature.

2. Make a copy of this structure.

3. Set the loop-count of the first objective to zero.

5. Encode this copy in CBOR to be used as the COSE payload, as described in {{RFC8152}}.

4. Create a COSE signature expressed in CBOR with this payload, as described in {{RFC8152}}, using this node's key.

5. Decode this signature from CBOR, typically using a loads() function.

6. Replace the payload component by nil.

7. Re-encode this signature in CBOR, typically using a dumps() function.

The result is the bytestring to be included in the sign-option.

The COSE signature uses the following choices according to RFC 8152:

TBD TBD

The process by which the relevant COSE keys are generated and distributed is out of scope for the present document.

# Verification

Upon receipt of a signed M_FLOOD message, each GRASP node **SHOULD**
verify it with a key related to the node identified by the 'initiator' element.
If verification succeeds, the message is processed as normal. If
verification fails, the message **MUST** be ignored and the event **MAY** be logged.

Verification proceeds as follows:

1. Extract the signature bytes from the sign-option and decoding them
from CBOR, typically using a loads() function. The resulting object is
the COSE signature with a detached payload.

2. Make a copy of the received M_FLOOD message

3. Remove the sign-option from that copy.

4. Set the loop-count component of the first GRASP objective in that copy to zero.

5. Re-encode this copy as CBOR, typically using a dumps() function.

6. Insert the result as the 'payload' component of the COSE signature.

7. Verify the signature as defined in {{RFC8152}}.

A node that does not support verification of  a signed M_FLOOD message
**MAY** process the message as normal, ignoring the sign-option, and
**MAY** log the presence of the extra option.

# Open Issues \[RFC Editor: please remove]

1. The above describes a "voluntary-to-verify" signature, i.e. nodes that do not support COSE signing can simply ignore the signature. Is this OK, or do we also need a "mandatory-to-verify" version?

2. Is the current arbitrary limitation to 256 option types necessary?

# Implementation Status \[RFC Editor: please remove]

TBD. Code will be [on Github](https://github.com/becarpenter/graspy).

# Security Considerations

The security considerations of {{RFC8990}} apply. This document enhances protection against malicious nodes by defining a verifiable COSE signature for flooded objectives. It does not describe the keying process.

In a network where only some nodes are capable of verifying the signature of flooded GRASP objectives, the operator must consider which objectives require to be signed, and what is the impact of some nodes using the flooded information without verification. An operator might choose to disallow nodes that cannot verify such messages.

# Revised CDDL Specification of GRASP
{: #grasp-cddl}
This section replaces Section 4 of {{RFC8990}}.

~~~~
<CODE BEGINS>
grasp-message = (message .within message-structure) / noop-message

message-structure = [MESSAGE_TYPE, session-id, ?initiator,
                     *grasp-element]

grasp-element = (option .within option-structure) / objective / any
option-structure = [OPTION_TYPE, *any]

MESSAGE_TYPE = 0..255
session-id = 0..4294967295 ; up to 32 bits
OPTION_TYPE = 0..255

message /= discovery-message
discovery-message = [M_DISCOVERY, session-id, initiator, objective]

message /= response-message ; response to Discovery
response-message = [M_RESPONSE, session-id, initiator, ttl,
                    (+locator-option // divert-option), ?objective]

message /= synch-message ; response to Synchronization request
synch-message = [M_SYNCH, session-id, objective]

message /= flood-message
flood-message = [M_FLOOD, session-id, initiator, ttl,
                 +[objective, (locator-option / [])], ?sign-option]

message /= request-negotiation-message
request-negotiation-message = [M_REQ_NEG, session-id, objective]

message /= request-synchronization-message
request-synchronization-message = [M_REQ_SYN, session-id, objective]

message /= negotiation-message
negotiation-message = [M_NEGOTIATE, session-id, objective]

message /= end-message
end-message = [M_END, session-id, accept-option / decline-option]

message /= wait-message
wait-message = [M_WAIT, session-id, waiting-time]

message /= invalid-message
invalid-message = [M_INVALID, session-id, ?any]

noop-message = [M_NOOP]

option /= divert-option
divert-option = [O_DIVERT, +locator-option]

option /= accept-option
accept-option = [O_ACCEPT]

option /= decline-option
decline-option = [O_DECLINE, ?reason]
reason = text  ; optional UTF-8 error message

waiting-time = 0..4294967295 ; in milliseconds
ttl = 0..4294967295 ; in milliseconds

option /= locator-option
locator-option /= [O_IPv4_LOCATOR, ipv4-address,
                   transport-proto, port-number]
ipv4-address = bytes .size 4

locator-option /= [O_IPv6_LOCATOR, ipv6-address,
                   transport-proto, port-number]
ipv6-address = bytes .size 16

locator-option /= [O_FQDN_LOCATOR, text, transport-proto,
                   port-number]

locator-option /= [O_URI_LOCATOR, text,
                   transport-proto / null, port-number / null]

transport-proto = IPPROTO_TCP / IPPROTO_UDP
IPPROTO_TCP = 6
IPPROTO_UDP = 17
port-number = 0..65535

option /= sign-option
sign-option = [O_COSE_SIGN, bytes] ; see "COSE Signature for GRASP"

initiator = ipv4-address / ipv6-address

objective-flags = uint .bits objective-flag

objective-flag = &(
  F_DISC: 0    ; valid for discovery
  F_NEG: 1     ; valid for negotiation
  F_SYNCH: 2   ; valid for synchronization
  F_NEG_DRY: 3 ; negotiation is a dry run
)

objective = [objective-name, objective-flags,
             loop-count, ?objective-value]

objective-name = text ; see section "Format of Objective Options"

objective-value = any

loop-count = 0..255

; Constants for message types and option types

M_NOOP = 0
M_DISCOVERY = 1
M_RESPONSE = 2
M_REQ_NEG = 3
M_REQ_SYN = 4
M_NEGOTIATE = 5
M_END = 6
M_WAIT = 7
M_SYNCH = 8
M_FLOOD = 9
M_INVALID = 99

O_DIVERT = 100
O_ACCEPT = 101
O_DECLINE = 102
O_IPv6_LOCATOR = 103
O_IPv4_LOCATOR = 104
O_FQDN_LOCATOR = 105
O_URI_LOCATOR = 106
O_COSE_SIGN = 107
<CODE ENDS>
~~~~

# IANA Considerations

IANA is requested to add one entry to the GRASP Messages and Options registry, and update
the unassigned values accordingly:

~~~~
107       O_COSE_SIGN   [RFCxxxx]
108-255   Unassigned
~~~~

--- back

# Change Log

## Draft-00

- Original version

# Acknowledgements

TBD
