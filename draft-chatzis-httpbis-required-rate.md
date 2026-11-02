---
title: "The Required-Rate HTTP Priority Parameter"
category: info

docname: draft-chatzis-httpbis-required-rate-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "HTTP"
keyword:
 - next generation
 - unicorn
 - AI-native
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "nikoschatzis/draft-chatzis-httpbis-required-rate"
  latest: "https://nikoschatzis.github.io/draft-chatzis-httpbis-required-rate/draft-chatzis-httpbis-required-rate.html"

author:
 -
    ins: N. Chatzis
    name: Nikolaos Chatzis
    organization: Meta
    email: nchatzis@meta.com
 -
    ins: S. Jaiswal
    name: Sharad Jaiswal
    organization: Meta
    email: sj77@meta.com
 -
    ins: A. Frindell
    name: Alan Frindell
    organization: Meta
    email: afrind@meta.com

normative:
  RFC9218:
  STRUCTURED-FIELDS: RFC9651

informative:
  QUIC: RFC9000

--- abstract

This document defines the required-rate (r) priority parameter for the
Extensible Prioritization Scheme for HTTP (RFC 9218).  The parameter allows a
client to indicate a delivery rate above which faster transfer of an HTTP
response is not expected to provide additional application benefit.  A server
can use this signal to rate-limit an individual response, allowing other eligible
responses to make progress without replacing the urgency and incremental
scheduling behavior defined by RFC 9218.

--- middle

# Introduction {#intro}

The Extensible Prioritization Scheme for HTTP {{RFC9218}} defines the
urgency (u) and incremental (i) parameters.  These parameters
allow a client to communicate the relative importance of concurrent
HTTP responses and whether their delivery can usefully be interleaved.

Urgency and incremental do not communicate the rate above
which faster delivery of a particular response ceases to provide
additional application benefit.  Consequently, a response selected by
the server's scheduler can consume all bandwidth currently available
to the connection, even when serving that response at a lower rate
would satisfy the application's delivery objective and allow other
responses to make progress.

This document defines the required-rate (r) parameter for the Priority HTTP
header field (Section 5 of {{RFC9218}}).  The parameter carries an
application-provided delivery rate in bits per second.  A server can use the
value as a per-response rate ceiling layered over its existing {{RFC9218}}
scheduling behavior.

The required-rate is an advisory application signal.  It is not a
reservation, a guaranteed minimum rate, or an estimate of the
bandwidth available to the connection.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Use Cases

## On-Screen Media and Prefetch

A media application can fetch content needed for current playback
while prefetching content that might be consumed later.  The
current-playback response generally has greater urgency, but
delivering it at every bit of available connection bandwidth might
provide little additional benefit once the application is comfortably
meeting its playback and buffering objectives.

The client can assign the current-playback response appropriate urgency,
incremental, and required-rate parameters. When the server rate-limits that
response, less urgent prefetch responses, that is, responses signaled with a
higher urgency value, can make progress using connection capacity that would
otherwise be unavailable to them.  Allowing prefetch to progress in this way
can reduce startup latency when the user navigates to the prefetched content.

The required-rate does not need to equal the encoded media bitrate.  A
client can select a higher rate to build a playback buffer or a lower
rate for speculative prefetch, according to its application policy.

## User-Visible Resources

Applications often have delivery objectives for resources that are not
consumed at a continuous rate.  For example, an application might need
an image to be available for rendering within a target interval or
to receive a feed response body quickly enough to satisfy a
loading-time objective.

A client can compute a required-rate for such a resource from the response
size, the desired availability time, and any data already received.  This
allows the parameter to be used for images, feed content, and other
user-visible resources without giving it a media-specific meaning.

## Other Applications

The mechanism is not limited to the examples above.  Any client that
can identify a rate above which faster delivery of a response provides
no additional application benefit can use the required-rate parameter.

# The Required-Rate Parameter {#required-rate}

The required-rate (r) parameter value is an Integer (Section 3.3.1 of
{{STRUCTURED-FIELDS}}) representing a delivery rate in bits per
second.

A positive value indicates the rate above which faster delivery of the response
is not expected to provide additional application benefit.  For a server that
chooses to act on the signal, the value is a ceiling while other responses are
eligible to be sent; it does not request or guarantee that the response will
receive that rate.

The default is 0.  An absent required-rate parameter and r=0 both mean
that no required-rate has been communicated.  Negative values and
values of unexpected types MUST be ignored.

The following example indicates urgency 2, incremental delivery, and a
required-rate of 5,000,000 bits per second:

~~~ http-message
:method = GET
:scheme = https
:authority = example.net
:path = /video/segment-42.m4s
priority = u=2, i, r=5000000
~~~

Unknown priority parameters and invalid parameter values are handled
as described by {{RFC9218}}.

# Client Behavior

A client SHOULD select a required-rate that reflects its application-level
delivery objective, that is, a rate above which faster delivery of the response
is not expected to provide additional application benefit.

Consumption rate is one way to determine the required-rate, but it is
not the only way.  An application can benefit from receiving a
response ahead of consumption, such as to build playback buffer,
prefetch content, or satisfy a resource-availability objective.

The client can derive this rate from one or more application-specific
inputs, including:

- the rate at which response data will be consumed;
- the amount of content already buffered;
- a desired response completion or availability time;
- the estimated amount of response data remaining;
- whether the response is needed immediately or is being prefetched; and
- the expected time at which a user-visible resource will be displayed.

For a response with a desired availability time, a client can derive
the required-rate from the estimated response data remaining and the
time remaining until that objective.  A client can include a margin
for uncertainty in response size, processing time, or network
delivery.

The required-rate expresses application demand rather than predicted
network capacity.  It can therefore be greater than or less than the
client's current bandwidth estimate.  Clients SHOULD NOT change the
value solely in response to short-term bandwidth-estimation
fluctuations unless those fluctuations materially change the
application's delivery objective.

A client MAY update the required-rate when its delivery objective
changes.  For example, a media client might update it after a material
change in playback-buffer occupancy, and an application can lower or
clear it when a response becomes speculative or is no longer
time-sensitive.

A client that cannot determine a meaningful required-rate SHOULD omit
the required-rate parameter.

# Interaction with PRIORITY_UPDATE Frames {#priority-update}

The required-rate parameter MAY appear in an initial Priority HTTP header field
and in a PRIORITY_UPDATE frame (Section 7 of {{RFC9218}}).

A valid required-rate value in a PRIORITY_UPDATE frame replaces the
previously signaled required-rate for the response.  Sending r=0 clears
the previous value.

As specified by {{RFC9218}}, a PRIORITY_UPDATE frame communicates all priority
parameters in the Priority Field Value field.  Omitting the required-rate
parameter is a signal to use its default value of 0.

A server that receives a new valid required-rate MAY apply it to data
transmitted after processing the update.  The update does not alter the rate at
which data was delivered before it was processed.

PRIORITY_UPDATE frames can be received out of order relative to request
data.  The ordering and processing requirements of {{RFC9218}} apply
unchanged.

# Server Scheduling Behavior {#scheduling}

A server that receives a required-rate value MAY use it as an input to its
scheduling decisions.  As with the other priority parameters defined by
{{RFC9218}}, the required-rate is advisory and a server is not obligated to act
on it. A server that acts on a required-rate SHOULD otherwise schedule the
response according to {{RFC9218}}.

When transmitting additional data would cause a response to exceed its
required-rate and other responses are eligible to be sent, temporarily
demoting the response is RECOMMENDED: the server schedules it at a lower
priority than signaled, e.g., as if it had urgency 7 and the incremental
parameter set, and counts data sent while demoted against its
required-rate.  A demoted response uses only capacity that higher-priority
responses do not use, so connection capacity is not left unused.  A server
MAY instead temporarily treat the response as ineligible for scheduling.

Once the response is again within its required-rate, it is scheduled
according to its signaled urgency and incremental parameters.  Other
responses continue to be selected according to the server's {{RFC9218}}
scheduling policy.

When no other response is eligible to be sent, e.g., because other responses
are application-limited or blocked by flow control, a server SHOULD continue to
send the response above its required-rate, without pausing it or repeatedly
changing how it is scheduled.  The required-rate therefore affects delivery
only when other responses compete for capacity.

A server MUST NOT treat the required-rate as a minimum allocation or a
bandwidth reservation.  Actual delivery remains subject to congestion
control ({{QUIC}}), connection-level and stream-level flow control,
peer behavior, and other transport constraints.  A required-rate does
not permit a server to transmit more data than the transport allows.

## Rate Measurement and Bursts

The required-rate describes an average delivery rate.  It does not require
uniform packet pacing or a particular rate-control algorithm.

A server MAY enforce the required-rate using pacing, a token bucket,
cumulative byte accounting, or another mechanism.  The accounting
interval is implementation-defined.  A token bucket refilled at the
required-rate, with a depth of one maximum-sized packet plus 10
milliseconds of data at the required-rate, is RECOMMENDED.  A server MAY
instead account for all data sent since the start of the response, at the
cost of larger bursts after periods in which the response was not sent.
A server MAY permit bounded bursts, including bursts needed for efficient
packetization and transport operation.

When a server chooses to enforce the required-rate, its accounting mechanism
SHOULD prevent the response's average delivery rate from materially exceeding
the signaled required-rate, while other responses are eligible to be sent,
over periods longer than the implementation's burst allowance and scheduling
quantum.  An implementation SHOULD avoid an accounting interval or burst
allowance so large that it defeats the purpose of the signaled required-rate.

On receiving a required-rate update ({{priority-update}}), a server MAY reset
or adjust its rate accounting state.  It SHOULD apply the new value without
continuing to withhold data solely because of the previous value.

# Security Considerations {#security}

The required-rate is an advisory signal.  A server retains discretion
over whether and how to apply it.

A malicious or malfunctioning client can send an artificially low
value and cause its own response to be delivered slowly if the server
honors the signal.  A server can ignore values it considers
pathological or inconsistent with local policy.

A high required-rate does not reserve bandwidth and does not override
congestion control, flow control, or the server's scheduling policy
({{scheduling}}).  A client therefore cannot use the parameter to
require the server to exceed the rate permitted by the transport.

Frequent updates can create additional scheduling and timer work.
Servers MAY rate-limit the processing of required-rate updates or
ignore updates that do not materially change the effective value.

The security considerations of {{RFC9218}} apply in their entirety.

# IANA Considerations {#iana}

This specification registers the following entry in the "Hypertext
Transfer Protocol (HTTP) Priority" registry established by
{{RFC9218}}:

| Name | Description | Reference |
|------|-------------|-----------|
| r    | Application-indicated delivery-rate ceiling for an HTTP response, in bits per second. | [](#required-rate) |
{: title="Additional Priority Parameter"}

This registration uses a key length of one and is therefore subject to
the Specification Required policy specified in Section 4.3.1 of
{{RFC9218}}.  This document is the specification for that registration.


--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank Brandon Schlinker and Max Repaci for
discussions and feedback that helped shape the overall design.
