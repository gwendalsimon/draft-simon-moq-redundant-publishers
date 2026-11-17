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

TODO: Abstract, written once the plan is agreed.

--- middle

# Introduction

TODO

# Conventions and Definitions

{::boilerplate bcp14-tagged}

TODO

The relay behavior in {{relay-behavior}} uses the following terms:

Source:
: An upstream session over which a relay receives the Track, through a
  subscription, a PUBLISH, or a FETCH. Each session is a distinct source. A new
  session to the same peer after a reconnect is a new source.

Binding:
: The association of one Group of the Track with one source. A relay delivers
  a bound Group only from that source ({{sources-and-bindings}}).

Tag:
: The value of PUBLISHER_TAG in an Object ({{publisher-tag}}). The tag
  identifies one run of the Original Publisher of the Object.

Truncated Group:
: A Group whose source ended before delivering all of it
  ({{truncated-groups}}).

# REDUNDANT_PUBLISHERS {#redundant-publishers}

TODO

# PUBLISHER_ROLE {#publisher-role}

TODO

# PUBLISHER_TAG {#publisher-tag}

PUBLISHER_TAG (Property Type TBD) is an Object Property ({{MOQT}}). The
Property Type is even. Therefore, the value is one variable-length integer:
the tag of the Original Publisher of the Object. The Property Type is outside
the range that {{MOQT}} reserves for Mandatory Track Properties.

The tag identifies one run of an Original Publisher. An Original Publisher
draws its tag uniformly at random between 0 and 2^32-1 when it starts. An
Original Publisher that restarts without the state of its previous run MUST
draw a new tag. An Original Publisher that reconnects with its state keeps its
tag. Two runs draw the same tag with a probability of 2^-32.

An Original Publisher MUST NOT take its tag from its configuration. A
configured tag survives a restart. The restarted Original Publisher would then
send a different copy of a Group under the same tag.

An Original Publisher MUST include PUBLISHER_TAG in every Object of the Track
with status Normal, including the Objects that it sends as Datagrams. A relay
forwards and caches the property unchanged, as {{MOQT}} requires for Object
Properties. Therefore, every copy of an Object carries the tag of its Original
Publisher, including a copy in a FETCH response. A relay MUST treat a source
that delivers an Object with status Normal and without PUBLISHER_TAG as a
failed source ({{failed-sources}}). {{filling-and-fetch}} defines how a relay
uses the tag.

# Setup Option {#setup-option}

TODO

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

A relay binds a Group when it first receives an Object of that Group from any
source. The relay selects the source as follows:

* When no source carries PUBLISHER_ROLE ({{publisher-role}}), or when the
  Selection Timeout is zero, the relay binds the Group to the first source
  that delivers an Object of the Group.

* Otherwise, the relay waits up to the Selection Timeout, measured from the
  arrival of that first Object, for the source with the lowest PUBLISHER_ROLE
  among the sources from which it currently receives the Track. The relay
  binds the Group to that source if that source delivers an Object of the
  Group in time. Otherwise, the relay binds the Group to the lowest-role
  source that has delivered an Object of the Group.

While it waits, the relay holds the Objects of the Group from every source and
forwards none of them. The wait adds at most the Selection Timeout to the
latency of a Group. The wait occurs only when the preferred source is late. A
relay MAY bound the Objects that it holds during the wait. A source whose held
Objects the relay dropped is not eligible for that Group. A relay that bounds
the Objects it holds keeps the Selection Timeout unchanged.
{{relay-security}} describes a local cap on the timeout and its cost.

A relay subscribes to every publisher of the Track that it discovers, as
{{MOQT}} requires of a relay. A relay MAY limit the number of sources that it
holds, with the means that {{MOQT}} gives a relay for additional publishers.
The relay SHOULD keep the sources with the lowest PUBLISHER_ROLE. A relay
cannot select a source that it does not hold. It can fail over to that source
only after it subscribes to the source and waits for a Group boundary.

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
chooses one source for the Group and requests the Group from that source.
When roles are present, the relay prefers the source with the lowest
PUBLISHER_ROLE. A FETCH that spans more than one Group can bind different
Groups to different sources. It binds each Group to one source only. For
example, suppose the source of an upstream FETCH for Groups 1 through 5 fails
while delivering Group 3. Groups 1 and 2 stay bound to that source. Group 3 is
truncated. The relay can request Groups 4 and 5 from another source.

A relay keeps the binding of a Group while it holds any Object or Object state
of the Group. Once the relay holds none, the binding ends. A later request MAY
then bind the Group again, possibly to a different source. A relay MAY
remember the source of a Group after the binding ends, so that it binds the
Group to the same source again.

The tag ({{publisher-tag}}) protects a Group after a binding ends upstream.
All Objects of a Group that a relay holds carry the same tag, because they
come from one Original Publisher. A relay MUST compare the tag of each Object
that it receives from the source of a Group with the tag of the Objects of
that Group that it holds. When the tags differ, the relay MUST discard the
received Object. It neither forwards nor caches that Object. The relay then
handles the Group as a truncated Group ({{truncated-groups}}).

For example, suppose a downstream relay holds Objects 0 through 20 of a Group
and requests the rest from an upstream relay. The upstream relay no longer
holds the Group, and its binding has ended. The upstream relay binds the Group
again, to a source that delivers the copy of another Original Publisher. The
Objects that it returns carry the tag of that Original Publisher. The
downstream relay discards them and records the rest of the Group as Gone.
Without the tag, the downstream relay would serve the head of one copy and the
tail of another copy as one Group.

{{MOQT}} limits a cached Gone state to MAX_CACHE_DURATION when the Track
carries that property. When the Track does not carry it, a relay MUST NOT keep
a Gone state of the Track longer than it keeps the Objects of the Track. Once
the Objects and the Gone state of a Group expire, a later request can bind the
Group again and obtain a complete copy.

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

A relay MUST NOT forward an End of Track status whose Location is at or below
an Object of the Track that it has forwarded or cached from any source.
Forwarding it would mark as not existing an Object that the relay delivered,
which is a Malformed Track condition in {{MOQT}}. When this rule forbids the
largest Location, the relay sends no End of Track status. It ends its
downstream subscriptions with PUBLISH_DONE and status TRACK_ENDED.

A publisher SHOULD end the Track at a Group boundary. A backup publisher that
stops cleanly therefore never ends the Track for the subscribers of the main
publisher.

# Security Considerations

TODO

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
authorized publisher can today.

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

TODO

--- back

# Acknowledgments
{:numbered="false"}

TODO
