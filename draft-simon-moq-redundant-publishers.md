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

Truncated Group:
: A Group whose source ended before delivering all of it
  ({{truncated-groups}}).

# REDUNDANT_PUBLISHERS {#redundant-publishers}

TODO

# PUBLISHER_ROLE {#publisher-role}

TODO

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
relay MAY cap the Selection Timeout that it applies at a local maximum, to
bound the memory that it spends holding Objects ({{relay-security}}).

A relay MAY receive the Track from a subset of the available sources, for
example from the main publisher alone until that publisher fails. The relay
then trades failover time for upstream bandwidth. The binding rules do not
change.

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

All sources of the Track MUST send the same Track Properties. A relay SHOULD
NOT bind a Group to a source whose Track Properties differ from those of the
sources that it already uses.

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
of the Group. The relay also keeps the binding for at least MAX_CACHE_DURATION
({{MOQT}}) after it last forwarded or served any Object of the Group. The
binding therefore outlives the Objects themselves. After that period, a later
request MAY bind the Group again, possibly to a different source.

Keeping the binding after the Objects expire closes a gap. Suppose a
downstream relay still holds part of a Group and requests the rest. Without
the kept binding, the upstream relay could serve the rest from another
Original Publisher and mix two copies in one Group. With the kept binding, the
upstream relay serves the rest from the same source. If that source is gone,
the upstream relay marks the rest Gone ({{truncated-groups}}). Each relay
restarts the period each time it serves the Group. The binding therefore
outlasts every downstream copy.

## Truncated Groups {#truncated-groups}

A relay does not complete a truncated Group from another source. The relay
binds the next Group afresh. The loss is therefore bounded by the remainder of
one Group.

A Group becomes truncated at a relay when its source ends before delivering
all of it. The source ends when the session closes or the subscription ends.
The source also ends when a Subgroup stream is reset and a request to the same
source cannot recover the missing Objects.

For a truncated Group, the relay takes three actions:

* The relay ends each Subgroup stream of the Group that it was still
  delivering downstream with a reset. It SHOULD use RESET_STREAM_AT with a
  reliable_size that covers the Objects that it has sent (the Closing Subgroup
  Streams section of {{MOQT}}).

* The relay records as Gone the Objects of the Group that it neither received
  from the source nor knows not to exist. It MAY cache that state.

* In FETCH responses, the relay serves the Objects that it holds and marks the
  Objects that it recorded as Gone with End of Gone Range indicators. The last
  indicator ends at Location `{Group ID, 2^64-1}`.

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
Group as it handles any Group whose Subgroup streams were reset.

These two requirements let player implementations add support before any
Track uses the property, at no cost.

An endpoint that receives the Track over more than one session, for example a
player connected to two relays, binds each Group to one session, as a relay
does ({{sources-and-bindings}} through {{truncated-groups}}).

## Failed and Malformed Sources {#failed-sources}

A relay MAY treat a source whose Objects make the Track malformed as a failed
source, instead of terminating downstream subscriptions. The relay cancels its
requests to that source. It truncates any Group bound to that source
({{truncated-groups}}). It continues with the remaining sources. If no source
remains, the Malformed Track rules in {{MOQT}} apply.

# Security Considerations

TODO

## Relay Considerations {#relay-security}

Holding Objects for the Selection Timeout costs relay memory for every Group
and every source. A publisher that sets a long timeout raises that cost. A
relay can therefore cap the timeout that it applies ({{source-selection}}).

A relay cannot tell a divergent copy of a Group from a corrupted or forged
one. Under this property, the relay does not need to. Authorization is
unchanged. Every source has to be authorized to publish the Track. Groups from
any authorized source can reach subscribers, as Objects from a single
authorized publisher can today.

Copies of a Group from different Original Publishers differ. Per-Object
signatures, for example in Immutable Properties ({{MOQT}}), therefore differ
between Groups. A verifier has to accept the key of each publisher.
Verification within a Group still works, because no Group is ever mixed.

# IANA Considerations

TODO

--- back

# Acknowledgments
{:numbered="false"}

TODO
