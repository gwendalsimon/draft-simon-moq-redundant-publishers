---
title: "Redundant Publishers for Media over QUIC"
abbrev: "MOQT Redundant Publishers"
category: std

docname: draft-simon-moq-redundant-publishers-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - redundancy
 - mandatory track property
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "gwendalsimon/draft-simon-moq-redundant-publishers"
  latest: "https://gwendalsimon.github.io/draft-simon-moq-redundant-publishers/draft-simon-moq-redundant-publishers.html"

author:
  - fullname: Gwendal Simon
    organization: Quortex
    email: gwendal.simon@quortex.io
  - fullname: Steven Riedl
    organization: Paramount
    email: steven.riedl@pluto.tv

normative:
  MOQT: I-D.draft-ietf-moq-transport

informative:
  REAP:
    title: "Information technology — Dynamic adaptive streaming over HTTP (DASH) — Part 9: Redundant encoding and packaging for segmented live media (REaP)"
    author:
      org: ISO/IEC
    date: 2024
  NETFLIX-MHV26:
    title: "Intelligent Live Origin and Resilient Streaming"
    author:
      - name: Xiaomei Liu
      - name: Rajasekhar Ummadisetty
      - name: Joseph Lynch
      - name: Chris Newton
    seriesinfo:
      "Mile-High Video Conference (MHV '26)": ""
    date: 2026

--- abstract

Live streaming services run redundant encoders. The copies of a Track from two
encoders are interchangeable, but they can differ in bytes. Media over QUIC
Transport (MOQT) treats two different Objects at the same Location as a
malformed Track. Therefore, a relay that receives both copies ends the Track.
This document defines REDUNDANT_PUBLISHERS, a Mandatory Track Property by which
the Original Publishers of a Track declare that their copies can differ. A relay
that supports the property delivers each Group from one Original Publisher and
continues the Track when one Original Publisher fails. Every Object carries
PUBLISHER_TAG, an Object Property that identifies its Original Publisher. A
relay uses the tag to keep each Group from one Original Publisher across several
hops. An optional Object Property, PUBLISHER_ROLE, lets a relay prefer a main
publisher.

--- middle

# Introduction

Live streaming services run redundant encoders, so that the failure of one
encoder does not interrupt the stream. For segmented media, REaP {{REAP}}
gives the same name to the segments that redundant encoders produce for the
same media. A packager or an origin can then take each segment from any
encoder without the player noticing. The segments are interchangeable, but
there is no guarantee that they are byte-identical.

Interchangeable segments still produce glitches when a player receives them
mixed. Netflix reports audio and video glitches when its origin mixes segments
from two pipelines {{NETFLIX-MHV26}}. Therefore, its origin prefers one
pipeline and switches to another pipeline only at a segment boundary, when the
preferred pipeline fails.

MOQT {{MOQT}} binds each Location of a Track to one Object. A relay that
receives two Objects at the same Location with different Payloads considers
the Track malformed (the Malformed Tracks section of {{MOQT}}). The relay then
terminates its downstream subscriptions. In a deployment with redundant
encoders, the two copies of a Track can break that rule wherever they differ.
Therefore, the redundancy that is meant to keep the Track alive can end it at
the first relay that receives both copies.

This document defines REDUNDANT_PUBLISHERS, a Mandatory Track Property by
which the Original Publishers of a Track declare that their copies can differ.
A relay that supports the property delivers each Group from one Original
Publisher. No Group mixes two copies. The Track continues when one Original
Publisher fails. An optional Object Property, PUBLISHER_ROLE, lets a relay
prefer a main publisher, as the Netflix origin prefers one pipeline. Under the
rules for Mandatory Track Properties in {{MOQT}}, a relay that does not
support the property does not carry the Track.

This document assumes that the Original Publishers of a Track use the same
Group ID for the same media, with aligned Group boundaries. How the encoders
align their output is out of scope. REaP {{REAP}} is one method.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms Track, Group, Subgroup, Object, Location, Original
Publisher, Track Property, Object Property, Mandatory Track Property, and Setup
Option as defined in {{MOQT}}.

This document also uses the following terms:

Tag:
: A number that an Original Publisher builds when it starts and includes in
  every Object it publishes ({{publisher-tag}}). Two Objects with the same tag
  come from the same Original Publisher, with no restart between them.

Role:
: The preference value that an Original Publisher carries in each Object
  ({{publisher-role}}). A relay prefers a lower role.

Main publisher:
: The Original Publisher to which the operator gives role 0.

Backup publisher:
: An Original Publisher to which the operator gives a role higher than 0.

Selection Timeout:
: The maximum time a relay waits, after the first Object of a Group arrives,
  before it chooses the source of the Group ({{source-selection}}). The
  REDUNDANT_PUBLISHERS property carries it ({{redundant-publishers}}).

Source:
: An upstream session over which a relay receives the Track, through a
  subscription, a PUBLISH, or a FETCH. Each session is a distinct source. A new
  session to the same peer after a reconnect is a new source.

Binding:
: The association of one Group of the Track with one source. A relay delivers
  a bound Group only from that source ({{sources-and-bindings}}).

Truncated Group:
: A Group whose source ended before delivering all of it
  ({{truncated-groups}}).

# Overview {#overview}

Two or more Original Publishers publish the same Track and set the
REDUNDANT_PUBLISHERS Track Property ({{redundant-publishers}}). The property
carries a Selection Timeout. Each Original Publisher includes its tag in every
Object ({{publisher-tag}}). An operator can also give each Original Publisher a
role ({{publisher-role}}): one main publisher and one or more backup publishers,
in order of preference. Each Object then carries the role of its Original
Publisher.

A relay delivers each Group of the Track from one source. The relay waits for
the Selection Timeout after the first Object of a Group arrives. It then chooses
the source that delivered the lowest role. Among sources with the same role, it
chooses the earliest. An Object without a role counts as the highest role. The
relay discards the copies of the Group received from other sources
({{relay-behavior}}).

A source can fail while it delivers a Group. The relay does not complete the
Group from another source, because the result would mix two copies. The relay
truncates the Group and delivers the next Group from another source.
Therefore, the subscribers lose at most the end of one Group. A relay that
receives an Object whose tag differs from the tag of the Group that it holds
discards the Object and truncates the Group.

A Setup Option ({{setup-option}}) declares support for the property. Under the
rules for Mandatory Track Properties in {{MOQT}}, an endpoint without support
for the property does not carry the Track. A player receiving the Track over
one session needs no new behavior beyond recognizing the property and
advertising the Setup Option ({{plain-subscribers}}).

# REDUNDANT_PUBLISHERS {#redundant-publishers}

REDUNDANT_PUBLISHERS (Property Type TBD1) is a Mandatory Track Property
({{MOQT}}). With this property, an Original Publisher declares that other
Original Publishers can publish the same Track. Their copies of a Group MAY
differ.

The Property Type is even. Therefore, the value is one variable-length integer:
the Selection Timeout, in milliseconds. A value of 0 means that a relay does not
wait. The relay then binds each Group to the first source to deliver an Object
of the Group. Original Publishers without a role SHOULD set the Selection
Timeout to 0.

Every Original Publisher of the Track MUST include REDUNDANT_PUBLISHERS in its
Track Properties. All Original Publishers of the Track MUST send the same
Track Properties, including the same Selection Timeout. A relay rejects a
source with different Track Properties ({{forwarding-and-caching}}).

{{MOQT}} defines how an endpoint without support for a Mandatory Track
Property handles the Track. A relay supporting REDUNDANT_PUBLISHERS follows
{{relay-behavior}} and forwards the property unchanged.

# PUBLISHER_TAG {#publisher-tag}

PUBLISHER_TAG (Property Type TBD2) is an Object Property ({{MOQT}}). The
Property Type is even. Therefore, the value is one variable-length integer:
the tag of the Original Publisher of the Object. The Property Type is outside
the range that {{MOQT}} reserves for Mandatory Track Properties.

The tag identifies an Original Publisher between two restarts. The tag has 21
bits. An Original Publisher builds its tag when it starts. The 16 high bits
carry the last 16 bits of its start time in milliseconds. The 5 low bits carry a
random nonce. An Original Publisher that restarts without the state of its
previous run MUST build a new tag. An Original Publisher that reconnects with
its state keeps its tag. An Original Publisher uses the same tag in every Track
that it publishes during one run.

An Original Publisher MUST include PUBLISHER_TAG in every Object of the Track
with status Normal, including the Objects that it sends as Datagrams. A relay
forwards and caches the property unchanged, as {{MOQT}} requires for Object
Properties. Therefore, every copy of an Object carries the tag of its Original
Publisher, including a copy in a FETCH response. A relay MUST treat a source
that delivers an Object with status Normal and without PUBLISHER_TAG as a
failed source ({{failed-sources}}). {{filling-and-fetch}} defines how a relay
uses the tag.

# PUBLISHER_ROLE {#publisher-role}

PUBLISHER_ROLE (Property Type TBD3) is an optional Object Property ({{MOQT}}).
The Property Type is even. Therefore, the value is one variable-length integer:
the role of the Original Publisher of the Object. The role ranges from 0 to 127.
A value of 0 identifies the main publisher. A higher value identifies a backup
publisher. The operator sets the role of each Original Publisher. An Original
Publisher MAY set a different role on a specific Group. For example, an Original
Publisher sets a higher role on a Group that it knows to contain encoding
artifacts. An Original Publisher MUST use the same role in every Object of a
Group.

An Original Publisher with a role SHOULD include PUBLISHER_ROLE in every Object
of the Track with status Normal. A relay considers an Object without
PUBLISHER_ROLE, or with a value higher than 127, as an Object with role 127. A
relay forwards and caches the property unchanged, as {{MOQT}} requires for
Object Properties. Therefore, a relay at any hop learns the role of the Original
Publisher of each Object it receives, including the Objects of a FETCH response.
{{source-selection}} describes how a relay uses the role.

# Setup Option {#setup-option}

REDUNDANT_PUBLISHERS_SUPPORTED (Setup Option Type TBD4) declares that an
endpoint supports this document. The Option Type is odd. The value has a length
of 0. The presence of the option declares support, as the Extension Negotiation
section of {{MOQT}} allows. The extension is negotiated on a session when both
endpoints include the option in SETUP.

An endpoint advertising the option supports the REDUNDANT_PUBLISHERS Track
Property ({{redundant-publishers}}). A relay advertising the option MUST follow
{{relay-behavior}}, including the use of PUBLISHER_TAG ({{publisher-tag}}) and
PUBLISHER_ROLE ({{publisher-role}}).

The Setup Option tells a publisher whether a subscriber supports
REDUNDANT_PUBLISHERS. On a session without the negotiated extension, a publisher
applies the rules of {{MOQT}} for a subscriber that does not support a Mandatory
Track Property. A subscriber that receives the Track without support refuses it
under the same rules.

This extension applies to the version of MOQT that {{MOQT}} defines.

# Relay Behavior {#relay-behavior}

This section defines how a relay handles a Track that carries the
REDUNDANT_PUBLISHERS Track Property ({{redundant-publishers}}). The rules apply
to every relay that has negotiated the Setup Option ({{setup-option}}). Under
the rules for Mandatory Track Properties in {{MOQT}}, no other relay processes
or forwards such a Track.

For such a Track, this section replaces two behaviors that {{MOQT}} defines
for a Track with multiple publishers. The first is forwarding Objects from
every publisher with best-effort deduplication (the Multiple Publishers section
of {{MOQT}}). The second is treating a duplicate Object with a different
Payload as a Malformed Track (the Caching Relays and Malformed Tracks sections
of {{MOQT}}). The rest of {{MOQT}} applies unchanged.

## Sources and Bindings {#sources-and-bindings}

A relay binds each Group of the Track to exactly one source. The relay
delivers that Group only from that source. A binding never changes while it
lasts. {{filling-and-fetch}} defines how long a binding lasts.

The binding rules hold end to end because every relay that processes the Track
follows them. The source of a relay delivers each Group from one Original
Publisher. The copy of that Group at the relay therefore also comes from one
Original Publisher. The same holds for every copy downstream of the relay.

## Source Selection {#source-selection}

A relay SHOULD bind a Group when the Selection Timeout expires. The timeout
starts when the relay receives the first Object of the Group from any source.
The role of a source for the Group is the role in the first Object of the Group
received from that source ({{publisher-role}}). The relay binds the Group to the
source with the lowest role. When several sources have the lowest role, the
relay binds the Group to the source whose first Object of the Group arrived
earliest. When the Selection Timeout is 0, the relay binds the Group to the
source of the first Object, without waiting.

A relay MAY bind a Group before the Selection Timeout expires, on a local
decision.

While it waits, the relay holds the Objects of the Group from every source and
forwards none of them. The wait adds at most the Selection Timeout to the
latency of a Group. A relay MAY bound the Objects that it holds during the wait.
A source whose held Objects the relay dropped is not eligible for that Group. A
relay that bounds the Objects it holds keeps the Selection Timeout unchanged.
{{relay-security}} describes a local cap on the timeout and its cost.

A relay subscribes to every publisher of the Track that it discovers, as
{{MOQT}} requires of a relay. A relay MAY limit the number of sources that it
holds, with the means that {{MOQT}} gives a relay for additional publishers.
The relay SHOULD keep the sources with the lowest role. A relay
cannot select a source that it does not hold. It can fail over to that source
only after it subscribes to the source and waits for a Group boundary.

Open issue: a relay selects a source per Track. Therefore, a relay can deliver
Tracks of one broadcast from different Original Publishers. See
https://github.com/gwendalsimon/draft-simon-moq-redundant-publishers/issues/14.

## Forwarding and Caching {#forwarding-and-caching}

Once a Group is bound, the relay handles the Objects of that Group as follows:

1. The relay forwards and caches the Objects and Object states of the Group
   that it receives from the source of the Group.

2. The relay discards the Objects and Object states of the Group that it
   receives from any other source. It neither forwards nor caches them.

3. The relay does not compare a discarded Object with the Object that it
   holds at the same Location. A discarded Object never makes the Track
   malformed.

The Malformed Track conditions in {{MOQT}} apply unchanged among the Objects
and Object states that a relay receives from one source.

All Subgroups of a Group share the binding of the Group. Objects of the Group
sent as Datagrams share it too. A relay MUST NOT deliver one Subgroup of a
Group from one source and another Subgroup of the same Group from another
source.

A relay MUST NOT switch the source of a Group while the Group is in progress.
The rule holds even when the new source could continue from the same
Location. A relay that resumes other Tracks mid-Group from an equivalent
upstream when the current upstream fails MUST NOT do so for this Track. The
relay truncates the Group instead ({{truncated-groups}}).

A relay compares the Track Properties of a new source with those of the sources
that it already uses. The relay makes the comparison when it receives the
PUBLISH, SUBSCRIBE_OK, or FETCH_OK that establishes the source. A relay MUST
reject a source whose Track Properties differ. The relay rejects a PUBLISH with
PUBLISH_ERROR and the error code TRACK_PROPERTY_MISMATCH. It cancels a
subscription or a fetch. A relay MUST NOT set INCLUDE_PROPERTIES to 0 in a
request to a source of the Track.

The first source that the relay accepts sets the reference Track Properties.
The relay keeps that reference while it holds any Object or Object state of
the Track. Once the relay holds none, the next source that it accepts sets a
new reference.

A rejected source removes redundancy without any effect that subscribers can
see. A relay SHOULD report each rejected source to its operator.

## Cache Fill and FETCH {#filling-and-fetch}

The binding also governs cache fill. A relay that needs Objects of a bound
Group that it does not hold requests them only from the source of the Group.
The need arises when the relay serves a FETCH or fills a gap in a
subscription. The relay MUST NOT request those Objects from another source,
even when the source of the Group is no longer available.

A FETCH for a Group that the relay has not bound binds that Group. The relay
chooses one source for the Group and requests the Group from that source. The
relay prefers the source whose most recent Object carried the lowest role. A
FETCH that spans more than one Group can bind different Groups to different
sources. It binds each Group to one source only. For example, suppose the source
of an upstream FETCH for Groups 1 through 5 fails while delivering Group 3.
Groups 1 and 2 stay bound to that source. Group 3 is truncated. The relay can
request Groups 4 and 5 from another source.

A relay keeps the binding of a Group while it holds any Object or Object state
of the Group. Once the relay holds none, the binding ends. A later request MAY
then bind the Group again, possibly to a different source. A relay MAY
remember the source of a Group after the binding ends, so that it binds the
Group to the same source again.

The tag ({{publisher-tag}}) protects a Group after a binding ends upstream.
All Objects of a Group that a relay holds carry the same tag, because they
come from one Original Publisher. A relay SHOULD compare the tag of each Object
that it receives from the source of a Group with the tag of the Objects of
that Group that it holds. When the tags differ, the relay MUST discard the
received Object. It neither forwards nor caches that Object. The relay then
handles the Group as a truncated Group ({{truncated-groups}}).

{{MOQT}} limits a cached Gone state to MAX_CACHE_DURATION when the Track
carries that property. When the Track does not carry it, a relay MUST NOT keep
a Gone state of the Track longer than it keeps the Objects of the Track. Once
the Objects and the Gone state of a Group expire, a later request can bind the
Group again and obtain a complete copy.

Open issue: a relay that holds part of a Group can request the rest from an
upstream relay whose binding of the Group has ended. The tag check then
truncates the Group. A FETCH parameter that carries the tag would let the
upstream relay request the rest from the same Original Publisher. The same
parameter would apply to a fill (FILL_PARAMETERS in {{MOQT}}). See
https://github.com/gwendalsimon/draft-simon-moq-redundant-publishers/issues/16.

## Truncated Groups {#truncated-groups}

A relay does not complete a truncated Group from another source. The relay
binds the next Group afresh. The loss is therefore bounded by the remainder of
one Group.

A Group becomes truncated at a relay when its source ends before delivering
all of it. The source ends when the session closes, when the subscription or
fetch ends, when the relay cancels its requests to the source
({{failed-sources}}), or when the source signals End of Track
({{end-of-track}}). A reset of one Subgroup stream by a source that remains
available does not end the source. The relay MAY request the missing Objects
from that source. The relay caches the Object states that the source returns.

For a truncated Group, the relay takes three actions:

* The relay ends each Subgroup stream of the Group that it was still
  delivering downstream with a reset. It SHOULD use RESET_STREAM_AT with a
  reliable_size that covers the Objects that it has sent (the Closing Subgroup
  Streams section of {{MOQT}}).

* The relay records as Gone the Objects of the Group that it neither received
  from the source nor knows not to exist. It MAY cache that state.

* The relay records the Gone range up to Location `{Group ID, 2^64-1}`. In
  FETCH responses, the relay serves the Objects that it holds and marks the
  Objects that it recorded as Gone with End of Gone Range indicators. The last
  indicator ends at the End Location of the request when that Location lies
  in the Group. Otherwise it ends at Location `{Group ID, 2^64-1}`.

The second action is an exception to the rule in {{MOQT}} that only the
Original Publisher originates Gone (the Object States and Relay Fetch Handling
sections of {{MOQT}}). The exception matches what Gone means there. The relay
cannot deliver those Objects and never will. Another copy of the Group, from
another Original Publisher, might still hold them. A downstream relay treats
this Gone state as it treats any Gone state received from upstream. It caches
and forwards the state and does not request those Objects again.

A relay that has recorded part of a Group as Gone under this section still
follows {{forwarding-and-caching}}. It accepts later Objects of the Group only
from the source of the Group.

## Endpoints That Neither Cache Nor Forward {#plain-subscribers}

A subscriber that receives the Track over one session and neither caches nor
forwards it, such as a player, needs no new data-plane behavior. Every Group
that it receives already comes from one Original Publisher. To receive the
Track at all, the subscriber MUST understand REDUNDANT_PUBLISHERS, so that it
does not reject the Track under the rules for Mandatory Track Properties in
{{MOQT}}. The subscriber MUST advertise the Setup Option ({{setup-option}}).
It ignores the Selection Timeout and PUBLISHER_ROLE. It handles a truncated
Group as it handles any Group whose Subgroup streams were reset. It MAY
compare the tags of the Objects of a Group ({{publisher-tag}}) to detect a
Group that mixes two copies.

These two requirements let player implementations add support before any
Track uses the property, at no cost.

An endpoint that receives the Track over more than one session, for example a
player connected to two relays, binds each Group to one session, as a relay
does ({{sources-and-bindings}} through {{truncated-groups}}).

## Failed and Malformed Sources {#failed-sources}

A relay MUST treat a source whose Objects make the Track malformed as a failed
source. It MUST NOT terminate downstream subscriptions on that account while
another source remains. The relay cancels its requests to that source. It
truncates any Group bound to that source ({{truncated-groups}}). It continues
with the remaining sources. If no source remains, the Malformed Track rules in
{{MOQT}} apply.

## End of Track {#end-of-track}

One source can signal End of Track while another source still delivers later
Groups, for example when one encoder stops before the other. End of Track
means that no Object at or after the given Location exists. Does Not Exist is
permanent and authoritative in {{MOQT}} (the Object States section of
{{MOQT}}).

End of Track from a source ends that source. The Track stays established
downstream while another source remains. The relay binds later Groups to the
remaining sources as usual. The relay forwards End of Track only when every
source of the Track has ended. It forwards the largest End of Track Location
that it received. When every source has ended and no source signaled End of
Track, the relay truncates a Group in progress ({{truncated-groups}}) and ends
its downstream subscriptions with PUBLISH_DONE and status TRACK_ENDED.

A source can stay connected without delivering any Object or Object state of the
Track. Under the rule above, such a source keeps the Track open with no time
limit. Therefore, once one source has signaled End of Track, a relay MAY treat
another source as ended. The condition is that the relay has received no Object
or Object state of the Track from the other source during an idle interval set
by the operator of the relay. The relay then cancels its subscriptions and
fetches to the ended source.

A relay MUST NOT forward an End of Track status whose Location is at or below
an Object of the Track that it has forwarded or cached from any source.
Forwarding it would mark as not existing an Object that the relay delivered,
which is a Malformed Track condition in {{MOQT}}. When this rule forbids the
largest Location, the relay sends no End of Track status. It ends its
downstream subscriptions with PUBLISH_DONE and status TRACK_ENDED.

A publisher SHOULD end the Track at a Group boundary.

# Security Considerations

The security considerations of {{MOQT}} apply. This document adds four
considerations for relays. A relay holds Objects during the Selection Timeout,
which costs memory. A relay also no longer treats two different Objects at the
same Location as a Malformed Track. Therefore, the relay cannot detect a
corrupted or forged copy of a Group by comparison with another copy. The tag
identifies an Original Publisher without authenticating it. A relay originates
Gone for the rest of a truncated Group. {{relay-security}} describes the four
considerations, with the effect of roles and of per-Object signatures.

## Relay Considerations {#relay-security}

Holding Objects for the Selection Timeout costs relay memory for every Group
and every source. A publisher that sets a long timeout raises that cost. A
relay bounds the Objects that it holds before it changes the timeout
({{source-selection}}). A relay under memory pressure MAY cap the Selection
Timeout that it applies at a local maximum. Such a cap changes which source
wins at that relay. Relays with different caps can bind the same Group to
different Original Publishers. The cap is an overload measure outside Source
Selection.

A relay cannot tell a divergent copy of a Group from a corrupted or forged
one. Under this property, the relay does not need to. Authorization is
unchanged. Every source has to be authorized to publish the Track. Groups from
any authorized source can reach subscribers, as Objects from a single
authorized publisher can today. An authorized Original Publisher can set role 0
and win every selection. The role expresses a preference, not a credential.

Copies of a Group from different Original Publishers differ. Per-Object
signatures, for example in Immutable Properties ({{MOQT}}), therefore differ
between Groups. A verifier has to accept the key of each publisher.
Verification within a Group still works, because no Group is ever mixed.

The tag ({{publisher-tag}}) is not a credential. An authorized Original
Publisher can send the tag of another Original Publisher. A relay cannot
detect the copied tag. A Group can then mix two copies. The tag protects a
Group against the loss of a binding ({{filling-and-fetch}}). It does not
authenticate an Original Publisher.

A relay originates Gone under {{truncated-groups}} and {{filling-and-fetch}}.
A subscriber cannot tell that state from Gone signaled by an Original
Publisher. A faulty relay can therefore hide Objects that another Original
Publisher still holds. The Gone state of a relay that follows this document
covers at most the rest of one Group. Every Gone state of the Track expires as
{{filling-and-fetch}} requires.

# IANA Considerations

This document registers five codepoints in the registries of {{MOQT}}.

This document registers three entries in the "MOQ Properties" registry:

| Type | Name                 | Scope  | Specification            |
|-----:|:---------------------|:-------|:-------------------------|
| TBD1 | REDUNDANT_PUBLISHERS | Track  | {{redundant-publishers}} |
| TBD2 | PUBLISHER_TAG        | Object | {{publisher-tag}}        |
| TBD3 | PUBLISHER_ROLE       | Object | {{publisher-role}}       |

TBD1 is an even value in the range of Mandatory Track Properties, 0x4000 to
0x7FFF. TBD2 and TBD3 are even values outside that range.

This document registers one entry in the "Setup Options" registry:

| Type | Name                           | Specification    |
|-----:|:-------------------------------|:-----------------|
| TBD4 | REDUNDANT_PUBLISHERS_SUPPORTED | {{setup-option}} |

TBD4 is an odd value.

This document registers one entry in the "REQUEST_ERROR Codes" registry:

| Name                    | Code | Specification              |
|:------------------------|:----:|:---------------------------|
| TRACK_PROPERTY_MISMATCH | TBD5 | {{forwarding-and-caching}} |

--- back

# Acknowledgments
{:numbered="false"}

TODO
