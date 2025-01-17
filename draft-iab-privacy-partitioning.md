---
title: "Partitioning as an Architecture for Privacy"
abbrev: "Partitioning for Privacy"
category: info
stream: IAB

docname: draft-iab-privacy-partitioning-latest
v: 3
keyword: Internet-Draft

author:
 -
    fullname: Mirja Kühlewind
    organization: Ericsson Research
    email: mirja.kuehlewind@ericsson.com
 -
    fullname: Tommy Pauly
    organization: Apple
    email: tpauly@apple.com
 -
    fullname: Christopher A. Wood
    organization: Cloudflare
    email: caw@heapingbits.net

normative:

informative:
  CensusReconstruction:
    title: The Census Bureau's Simulated Reconstruction-Abetted Re-identification Attack on the 2010 Census
    target: https://www.census.gov/data/academy/webinars/2021/disclosure-avoidance-series/simulated-reconstruction-abetted-re-identification-attack-on-the-2010-census.html


--- abstract

This document describes the principle of privacy partitioning, which selectively spreads data and communication across
multiple parties as a means to improve privacy by separating user identity from user data.
This document describes emerging patterns in protocols to partition what data and metadata is
revealed through protocol interactions, provides common terminology, and discusses how
to analyze such models.

--- middle

# Introduction

Protocols such as TLS and IPsec provide a secure (authenticated and encrypted) channel
between two endpoints over which endpoints transfer information. Encryption and authentication
of data in transit are necessary to protect information from being seen or modified by parties
other than the intended protocol participants. As such, this kind of security is necessary for ensuring that
information transferred over these channels remains private.

However, a secure channel between two endpoints is insufficient for the privacy of the endpoints
themselves. In recent years, privacy requirements have expanded beyond the need to protect data in transit
between two endpoints. Some examples of this expansion include:

- A user accessing a service on a website might not consent to reveal their location,
but if that service is able to observe the client's IP address, it can learn something
about the user's location. This is problematic for privacy since the service can link
user data to the user's location.

- A user might want to be able to access content for which they are authorized,
such as a news article, without needing to have which specific articles they
read on their account being recorded. This is problematic for privacy since the service
can link user activity to the user's account.

- A client device that needs to upload metrics to an aggregation service might want to be
able to contribute data to the system without having their specific contributions
attributed to them. This is problematic for privacy since the service can link client
contributions to the specific client.

The commonality in these examples is that clients want to interact with or use a service
without exposing too much user-specific or identifying information to that service. In particular,
separating the user-specific identity information from user-specific data is necessary for
privacy. Thus, in order to protect user privacy, it is important to keep identity (who) and data
(what) separate.

This document defines "privacy partitioning," sometimes also referred to as the "decoupling principle"
{{?DECOUPLING=DOI.10.1145/3563766.3564112}}, as the general technique used to separate the data
and metadata visible to various parties in network communication, with the aim of improving
user privacy. Partitioning is a spectrum and not a panacea. It is difficult to guarantee there
is no link between user-specific identity and user-specific data. However, when applied properly,
privacy partitioning helps ensure that user privacy violations become more technically difficult
to achieve over time.

Several IETF working groups are working on protocols or systems that adhere to the principle
of privacy partitioning, including Oblivious HTTP Application Intermediation (OHAI), Multiplexed Application Substrate over QUIC Encryption (MASQUE), Privacy Pass, and Privacy Preserving Measurement (PPM). This document summarizes
work in those groups and describes a framework for reasoning about the resulting privacy posture of different
endpoints in practice.

Privacy partitioning is particularly relevant as a tool for data minimization, which is described
in {{Section 6.1 of ?RFC6973}}. {{RFC6973}} provides guidance for privacy considerations in
Internet protocols, along with a set of questions on how to evaluate the data minimization
of a protocol in {{Section 7.1 of ?RFC6973}}. Protocols that employ privacy partitioning
ought to consider the questions in that section when evaluating their design, particularly
with regard to how identifiers and data can be correlated by protocol participants and
observers in each context that has been partitioned. Privacy partitioning can also be
used as a way to separate identity providers from relying parties
(see {{Section 6.1.4 of RFC6973}}), as in the case of Privacy Pass
(see Section {{privacypass}}).

# Privacy Partitioning

For the purposes of user privacy, this document focuses on user-specific information. This
might include any identifying information that is specific to a user, such as their email
address or IP address, or data about the user, such as their date of birth. Informally,
the goal of privacy partitioning is to ensure that each party in a system beyond the user
themselves only have access to one type of user-specific information.

This is a simple application of the principle of least privilege, wherein every party in
a system only has access to the minimum amount of information needed to fulfill their
function. Privacy partitioning advocates for this minimization by ensuring that protocols,
applications, and systems only reveal user-specific information to parties that need access
to the information for their intended purpose.

Put simply, privacy partitioning aims to separate *who* someone is from *what* they do. In the
rest of this section, we describe how privacy partitioning can be used to achieve this goal.

## Privacy Contexts

Each piece of user-specific information exists within some context, where a context
is abstractly defined as a set of data, metadata, and the entities that share access
to that information. In order to prevent the correlation of user-specific information across
contexts, partitions need to ensure that any single entity (other than the client itself)
does not participate in more than one context where the information is visible.

{{?RFC6973}} discusses the importance of identifiers in reducing correlation as a way
of improving privacy:

{:quote}
> Correlation is the combination of various pieces of information related to an individual
> or that obtain that characteristic when combined... Correlation is closely related to
> identification.  Internet protocols can facilitate correlation by allowing individuals'
> activities to be tracked and combined over time.
>
> Pseudonymity is strengthened when less personal data can be linked to the pseudonym; when
> the same pseudonym is used less often and across fewer contexts; and when independently
> chosen pseudonyms are more frequently used for new actions (making them, from an observer's or
> attacker's perspective, unlinkable).

Context separation is foundational to privacy partitioning and reducing correlation.
As an example, consider an unencrypted HTTP session over TCP, wherein the context includes both the
content of the transaction as well as any metadata from the transport and IP headers; and the
participants include the client, routers, other network middleboxes, intermediaries, and server.

~~~ aasvg
+-------------------------------------------------------------------+
| Context A                                                         |
|  +--------+                +-----------+              +--------+  |
|  |        +------HTTP------+           +--------------+        |  |
|  | Client |                | Middlebox |              | Server |  |
|  |        +------TCP-------+           +--------------+        |  |
|  +--------+      flow      +-----------+              +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
~~~
{: #diagram-middlebox title="Diagram of a basic unencrypted client-to-server connection with middleboxes"}

Adding TLS encryption to the HTTP session is a simple partitioning technique that splits the
previous context into two separate contexts: the content of the transaction is now only visible
to the client, TLS-terminating intermediaries, and server; while the metadata in transport and
IP headers remain in the original context. In this scenario, without any further partitioning,
the entities that participate in both contexts can allow the data in both contexts to be correlated.

~~~ aasvg
+-------------------------------------------------------------------+
| Context A                                                         |
|  +--------+                                           +--------+  |
|  |        |                                           |        |  |
|  | Client +-------------------HTTPS-------------------+ Server |  |
|  |        |                                           |        |  |
|  +--------+                                           +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
| Context B                                                         |
|  +--------+                +-----------+              +--------+  |
|  |        |                |           |              |        |  |
|  | Client +-------TCP------+ Middlebox +--------------+ Server |  |
|  |        |       flow     |           |              |        |  |
|  +--------+                +-----------+              +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
~~~
{: #diagram-https title="Diagram of how adding encryption splits the context into two"}

Another way to create a partition is to simply use separate connections. For example, to
split two separate HTTP requests from one another, a client could issue the requests on
separate TCP connections, each on a different network, and at different times; and avoid
including obvious identifiers like HTTP cookies across the requests.

~~~ aasvg
+-------------------------------------------------------------------+
| Context A                                                         |
|  +--------+                +-----------+              +--------+  |
|  |        | IP A           |           |              |        |  |
|  | Client +-------TCP------+ Middlebox +--------------+ Server |  |
|  |        |      flow A    |     A     |              |        |  |
|  +--------+                +-----------+              +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
| Context B                                                         |
|  +--------+                +-----------+              +--------+  |
|  |        | IP B           |           |              |        |  |
|  | Client +-------TCP------+ Middlebox +--------------+ Server |  |
|  |        |      flow B    |     B     |              |        |  |
|  +--------+                +-----------+              +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
~~~
{: #diagram-dualconnect title="Diagram of making separate connections to generate separate contexts"}

Using separate connections to create separate contexts can reduce or eliminate
the ability of specific parties to correlate activity across contexts. However,
any identifier at any layer that is common across different contexts can be
used as a way to correlate activity. Beyond IP addresses, many other factors
can be used together to create a fingerprint of a specific device (such as
MAC addresses, device properties, software properties and behavior, application state, etc).

## Context Separation

In order to define and analyze how various partitioning techniques work, the boundaries of what is
being partitioned need to be established. This is the role of context separation. In particular,
in order to prevent the correlation of user-specific information across contexts, partitions need
to ensure that any single entity (other than the client itself) does not participate in contexts
where both identities are visible.

Context separation can be achieved in different ways, for example, over time, across network paths, based
on (en)coding, etc. The privacy-oriented protocols described in this document generally involve
more complex partitioning, but the techniques to partition communication contexts still employ the
same techniques:

1. Encryption allows partitioning of contexts within a given network path.
1. Using separate connections across time or space allows partitioning of contexts for different
application transactions.

These techniques are frequently used in conjunction for context separation. For example,
encrypting an HTTP exchange might prevent a network middlebox that sees a client IP address
from seeing the user account identity, but it doesn't prevent the TLS-terminating server
from observing both identities and correlating them. As such, preventing correlation
requires separating contexts, such as by using proxying to conceal a client's IP address
that would otherwise be used as an identifier.

## Approaches to Partitioning

While all of the partitioning protocols described in this document create
separate contexts using encryption and/or connection separation, each one has a
unique approach that results in different sets of contexts. Since many of
these protocols are new, it is yet to be seen how each approach will be
used at scale across the Internet, and what new models will emerge in the
future.

There are multiple factors that lead to a diversity in approaches to
partitioning, including:

- Adding privacy partitioning to existing protocol ecosystems places
requirements and constraints on how contexts are constructed. CONNECT-style
proxying is intended to work with servers that are unaware of privacy contexts,
requiring more intermediaries to provide strong separation guarantees.
Oblivious HTTP, on the other hand, assumes servers that cooperate with context
separation, and thus reduces the overall number of elements in the solution.

- Whether or not information exchange needs to happen bidirectionally in an
interactive fashion determines how contexts can be separated. Some use cases,
like metrics collection for PPM, can occur with information flowing only from
clients to servers, and can function even when clients are no longer connected.
Privacy Pass is an example of a case that can be either interactive or not,
depending on whether tokens can be cached and reused. CONNECT-style proxying and
Oblivious HTTP often requires bidirectional and interactive communication.

- The degree to which contexts need to be partitioned depends in part
on the client's threat models and level of trust in various protocol participants. For example,
in Oblivious HTTP, clients allow relays to learn that clients are accessing a particular
application-specific gateway. If clients do not trust relays with this information, they can
instead use a multi-hop CONNECT-style proxy approach wherein no single party learns
whether specific clients are accessing a specific application. This is the default trust model
for systems like Tor, where multiple hops are used to drive down the probability of privacy
violations due to collusion or related attacks.

# A Survey of Protocols using Partitioning

The following section discusses current on-going work in the IETF
that is applying privacy partitioning.

## CONNECT Proxying and MASQUE {#masque}

HTTP forward proxies, when using encryption on the connection between the client and the
proxy, provide privacy partitioning by separating a connection into multiple segments.
When connections to targets via the proxy themselves are encrypted,
the proxy cannot see the end-to-end content. HTTP has historically supported forward proxying
for TCP-like streams via the CONNECT method. More recently, the Multiplexed Application
Substrate over QUIC Encryption (MASQUE) working group has developed
protocols to similarly proxy UDP {{?CONNECT-UDP=RFC9297}} and IP packets
{{?CONNECT-IP=I-D.ietf-masque-connect-ip}} based on tunneling.

In a single-proxy setup, there is a tunnel connection between the client and proxy and
an end-to-end connection that is tunnelled between the client and target. This setup,
as shown in the figure below, partitions communication into:

- a Client-to-Proxy context, which contains the transport metadata between the client
and the target, and the request to the proxy to open a connection to the target;

- a Client-to-Target proxied context, which is the end-to-end data to the target that is
also visible to the proxy, such as a TLS session;

- a Client-to-Target encrypted context, which contains the end-to-end content
with the TLS session to the target, such as HTTP content;

- and a Proxy-to-Target context, which for TCP and UDP proxying
contains any packet header information that is added or modified by the proxy,
e.g., the IP and TCP/UDP headers.

~~~ aasvg
+-------------------------------------------------------------------+
| Client-to-Target Encrypted Context                                |
|  +--------+                                           +--------+  |
|  |        |                                           |        |  |
|  | Client +------------------HTTPS--------------------+ Target |  |
|  |        |                 content                   |        |  |
|  +--------+                                           +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
| Client-to-Target Proxied Context                                  |
|  +--------+                +-----------+              +--------+  |
|  |        |                |           |              |        |  |
|  | Client +----Proxied-----+   Proxy   +--------------+ Target |  |
|  |        |    TLS flow    |           |              |        |  |
|  +--------+                +-----------+              +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
| Client-to-Proxy Context                                           |
|  +--------+                +-----------+                          |
|  |        |                |           |                          |
|  | Client +---Transport----+   Proxy   |                          |
|  |        |     flow       |           |                          |
|  +--------+                +-----------+                          |
|                                                                   |
+-------------------------------------------------------------------+
| Proxy-to-Target Context                                           |
|                            +-----------+              +--------+  |
|                            |           |              |        |  |
|                            |   Proxy   +--Transport---+ Target |  |
|                            |           |    flow      |        |  |
|                            +-----------+              +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
~~~
{: #diagram-1hop title="Diagram of one-hop proxy contexts"}


Using two (or more) proxies provides better privacy partitioning. In particular, with two proxies,
each proxy sees the Client metadata, but not the Target; the Target, but not the Client
metadata; or neither.

~~~ aasvg
+-------------------------------------------------------------------+
| Client-to-Target Encrypted Context                                |
|  +--------+                                           +--------+  |
|  |        |                                           |        |  |
|  | Client +------------------HTTPS--------------------+ Target |  |
|  |        |                 content                   |        |  |
|  +--------+                                           +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
| Client-to-Target Proxied Context                                  |
|  +--------+                           +-------+       +--------+  |
|  |        |                           |       |       |        |  |
|  | Client +----------Proxied----------+ Proxy +-------+ Target |  |
|  |        |          TLS flow         |   B   |       |        |  |
|  +--------+                           +-------+       +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
| Client-to-Proxy B Context                                         |
|  +--------+         +-------+         +-------+                   |
|  |        |         |       |         |       |                   |
|  | Client +---------+ Proxy +---------+ Proxy |                   |
|  |        |         |   A   |         |   B   |                   |
|  +--------+         +-------+         +-------+                   |
|                                                                   |
+-------------------------------------------------------------------+
| Client-to-Proxy A Context                                         |
|  +--------+         +-------+                                     |
|  |        |         |       |                                     |
|  | Client +---------+ Proxy |                                     |
|  |        |         |   A   |                                     |
|  +--------+         +-------+                                     |
|                                                                   |
+-------------------------------------------------------------------+
| Proxy A-to-Proxy B Context                                        |
|                     +-------+         +-------+                   |
|                     |       |         |       |                   |
|                     | Proxy +---------+ Proxy |                   |
|                     |   A   |         |   B   |                   |
|                     +-------+         +-------+                   |
|                                                                   |
+-------------------------------------------------------------------+
| Proxy B-to-Target Context                                         |
|                                       +-------+       +--------+  |
|                                       |       |       |        |  |
|                                       | Proxy +-------+ Target |  |
|                                       |   B   |       |        |  |
|                                       +-------+       +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
~~~
{: #diagram-2hop title="Diagram of two-hop proxy contexts"}

Forward proxying, such as the protocols developed in MASQUE, uses both encryption (via TLS) and
separation of connections (via proxy hops that see only the next hop) to achieve privacy partitioning.

## Oblivious HTTP and DNS

Oblivious HTTP {{?OHTTP=I-D.ietf-ohai-ohttp}}, developed in the Oblivious HTTP Application
Intermediation (OHAI) working group, adds per-message
encryption to HTTP exchanges through a relay system. Clients send requests through an Oblivious Relay,
which cannot read message contents, to an Oblivious Gateway, which can decrypt the messages but
cannot communicate directly with the client or observe client metadata like IP address.
Oblivious HTTP relies on Hybrid Public Key Encryption {{?HPKE=RFC9180}} to perform encryption.

Oblivious HTTP uses both encryption and separation of connections to achieve privacy partitioning.
The end-to-end messages are encrypted between the Client and Gateway (forming a Client-to-Gateway context),
and the connections are separated into a Client-to-Relay context and a Relay-to-Gateway context. It is also important
to note that the Relay-to-Gateway connection can be a single connection, even if the Relay has many
separate Clients. This provides better anonymity by making the pseudonym presented by the Relay to
be shared across many Clients.

~~~ aasvg
+-------------------------------------------------------------------+
| Client-to-Target Context                                          |
|  +--------+                           +---------+     +--------+  |
|  |        |                           |         |     |        |  |
|  | Client +---------------------------+ Gateway +-----+ Target |  |
|  |        |                           |         |     |        |  |
|  +--------+                           +---------+     +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
| Client-to-Gateway Context                                         |
|  +--------+         +-------+         +---------+                 |
|  |        |         |       |         |         |                 |
|  | Client +---------+ Relay +---------+ Gateway |                 |
|  |        |         |       |         |         |                 |
|  +--------+         +-------+         +---------+                 |
|                                                                   |
+-------------------------------------------------------------------+
| Client-to-Relay Context                                           |
|  +--------+         +-------+                                     |
|  |        |         |       |                                     |
|  | Client +---------+ Relay |                                     |
|  |        |         |       |                                     |
|  +--------+         +-------+                                     |
|                                                                   |
+-------------------------------------------------------------------+
~~~
{: #diagram-ohttp title="Diagram of Oblivious HTTP contexts"}

Oblivious DNS over HTTPS {{?ODOH=RFC9230}} applies the same principle as Oblivious HTTP, but operates on
DNS messages only. As a precursor to the more generalized Oblivious HTTP, it relies on the same
HPKE cryptographic primitives, and can be analyzed in the same way.

## Privacy Pass {#privacypass}

Privacy Pass is an architecture {{?PRIVACYPASS=I-D.ietf-privacypass-architecture}} and a set of protocols
being developed in the Privacy Pass working group that allows clients to present proof of verification in
an anonymous and unlinkable fashion, via tokens. These tokens originally were designed as a way to prove
that a client had solved a CAPTCHA, but can be applied to other types of user or device attestation checks
as well. In Privacy Pass, clients interact with an attester and issuer for the purposes of issuing a token,
and clients then interact with an origin server to redeem said token.

In Privacy Pass, privacy partitioning is achieved with cryptographic protection (in the form of blind
signature protocols or similar) and separation of connections across two contexts:
a "redemption context" between clients and origins (servers that request and receive tokens), and an
"issuance context" between clients, attestation servers, and token issuance servers. The cryptographic
protection ensures that information revealed during the issuance context is separated from information
revealed during the redemption context.

~~~ aasvg
+-------------------------------------------------------------------+
| Redemption Context                                                |
|  +--------+         +--------+                                    |
|  |        |         |        |                                    |
|  | Origin +---------+ Client |                                    |
|  |        |         |        |                                    |
|  +--------+         +--------+                                    |
|                                                                   |
+-------------------------------------------------------------------+
| Issuance Context                                                  |
|                     +--------+      +----------+      +--------+  |
|                     |        |      |          |      |        |  |
|                     | Client +------+ Attester +------+ Issuer |  |
|                     |        |      |          |      |        |  |
|                     +--------+      +----------+      +--------+  |
|                                                                   |
+-------------------------------------------------------------------+
~~~
{: #diagram-privacypass title="Diagram of contexts in Privacy Pass"}

Since the redemption context and issuance context are separate connections
that involve separate entities, they can also be further decoupled by
running those parts of the protocols at different times. Clients can
fetch tokens through the issuance context early, and cache the tokens
to later use in redemption contexts. This can aid in partitioning identifiers
and data.

{{PRIVACYPASS}} describes different deployment models for which entities operate
origins, attesters, and issuers; in some models, they are all separate
entities, but in others, they can be operated by the same entity. The
model impacts the effectiveness of partitioning, and some models
(such as when all three are operated by the same entity) only provide
effective partitioning when the timing of connections on the two
contexts are not correlated, and when the client uses different
identifiers (such as different IP addresses) on each context.

## Privacy Preserving Measurement

The Privacy Preserving Measurement (PPM) working group is chartered to develop protocols and systems
that help a data aggregation or collection server (or multiple, non-colluding servers) compute aggregate
values without learning the value of any one client's individual measurement. Distributed Aggregation
Protocol (DAP) is the primary working item of the group.

At a high level, DAP uses a combination of cryptographic protection (in the form of secret sharing amongst
non-colluding servers) to establish two contexts: an "upload context" between clients and non-colluding
aggregation servers wherein aggregation servers possibly learn client identity but nothing about their individual
measurement reports, and a "collect context" wherein a collector learns aggregate measurement results and nothing
about individual client data.

~~~ aasvg
+-------------------------------------+--------------------+
| Upload Context                      | Collect Context    |
|                     +------------+  |                    |
|              +----->|   Helper   |  |                    |
| +--------+   |      +------------+  |                    |
| |        +---+             ^        |   +-----------+    |
| | Client |                 |        |   | Collector |    |
| |        +---+             v        |   +-----+-----+    |
| +--------+   |      +------------+  |         |          |
|              +----->|   Leader   |<-----------+          |
|                     +------------+  |                    |
+-------------------------------------+--------------------+
~~~
{: #pa-topology title="Diagram of contexts in DAP"}

# Applying Privacy Partitioning

Applying privacy partitioning to an existing or new system or protocol requires the following steps:

1. Identify the types of information used or exposed in a system or protocol, some
of which can be used to identify a user or correlate to other contexts.
1.  Partition data to minimize the amount of user-identifying or correlatable
information in any given context to only include what is necessary for that
context, and prevent the sharing of data across contexts wherever possible.

The most impactful types of information to partition are (a) user-identifying information,
such as user identity or identities (including account names or IP addresses) that can be
linked and (b) non-user-identifying information (including content a user
generates or accesses), which can be often sensitive when combined with a user identity.

In this section, we discuss considerations for partitioning these types of information.

## User-Identifying Information

User data can itself be user-identifying, in which case it should be treated as an identifier.
For example, Oblivious DoH and Oblivious HTTP partition the client IP address and client request data into
separate contexts, thereby ensuring that no entity beyond the client can observe both. Collusion across contexts
could reverse this partitioning, but can also promote non-user-identifying information to user-identifying.
For example, in CONNECT proxy systems that use QUIC, the QUIC connection ID is inherently non-user-identifying
since it is generated randomly ({{?QUIC=RFC9000, Section 5.1}}). However, if combined with another context that has user-identifying
information such as the client IP address, the QUIC connection ID can become user-identifying information.

Some information is innate to client user-agents, including details of implementation of
protocols in hardware and software, and network location. This information can be used to construct
user-identifying information, which is a process sometimes referred to as fingerprinting.
Depending on the application and system constraints, users may not be able to prevent fingerprinting
in privacy contexts. As a result, fingerprinting information, when combined with non-user-identifying
user data, could promote user data to user-identifying information.

## Incorrect or Incomplete Partitioning

Privacy partitioning can be applied incorrectly or incompletely. Contexts may contain
more user-identifying information than desired, or some information in a context may be more user-identifying
than intended. Moreover, splitting user-identifying information over multiple contexts has to be done
with care, as creating more contexts can increase the number of entities that need to be trusted to not collude.
Nevertheless, partitions can help improve the client's privacy posture when applied carefully.


Evaluating and qualifying the resulting privacy of a system or protocol that applies privacy partitioning depends
on the contexts that exist and the types of user-identifying information in each context. Such evaluation is
helpful for identifying ways in which systems or protocols can improve their privacy posture. For example,
consider DNS-over-HTTPS {{?DOH=RFC8484}}, which produces a single context which contains both the client IP
address and client query. One application of privacy partitioning results in ODoH, which produces two contexts,
one with the client IP address and the other with the client query.

## Identifying Information for Partitioning

Recognizing potential applications of privacy partitioning requires identifying the contexts in use, the information
exposed in a context, and the intent of information exposed in a context. Unfortunately, determining what
information to include in a given context is a nontrivial task. In principle, the information contained
in a context should be fit for purpose. As such, new systems or protocols developed should aim to
ensure that all information exposed in a context serves as few purposes as possible. Designing with this
principle from the start helps mitigate issues that arise if users of the system or protocol inadvertently
ossify on the information available in contexts. Legacy systems that have ossified on information available
in contexts may be difficult to change in practice. As an example, many existing anti-abuse systems
depend on some notion of client identity such as client IP address, coupled with client data, to provide
value. Partitioning contexts in these systems such that they no longer see the client's identity requires new
solutions to the anti-abuse problem.

# Limits of Privacy Partitioning {#limits}

Privacy Partitioning aims to increase user privacy, though as stated, it is not a panacea.
The privacy properties depend on numerous factors, including, though not limited to:

- Non-collusion across contexts; and
- The type of information exposed in each context.

We elaborate on each below.

## Violations by Collusion

Privacy partitions ensure that only the client, i.e., the entity which is responsible for partitioning,
can link all user-specific information together up to collusion. No other entity individually
knows how to link all the user-specific information as long as they do not collude with each other
across contexts. This is why non-collusion is a fundamental requirement for privacy partitioning
to offer meaningful privacy for end-users. In particular, the trust relationships that users have
with different parties affect the resulting impact on the user's privacy.

As an example, consider OHTTP, wherein the Oblivious Relay knows the Client identity but not
the Client data, and the Oblivious Gateway knows the Client data but not the Client identity.
If the Oblivious Relay and Gateway collude, they can link Client identity and data together
for each request and response transaction by simply observing requests in transit.

It is not currently possible to guarantee with technical protocol measures that two
entities are not colluding. Even if two entities do not collude directly, if both entities reveal
information to other parties, it will not be possible to guarantee that the information won't
be combined. However, there are some mitigations that can be applied
to reduce the risk of collusion happening in practice:

- Policy and contractual agreements between entities involved in partitioning to disallow
logging or sharing of data, along with auditing to validate that the policies are being followed.
For cases where logging is required (such as for service operation), such logged data should
be minimized and anonymized to prevent it from being useful for collusion.
- Protocol requirements to make collusion or data sharing more difficult.
- Adding more partitions and contexts, to make it increasingly difficult to collude with
enough parties to recover identities.

## Violations by Insufficient Partitioning

It is possible to define contexts that contain more than one type of user-specific information,
despite efforts to do otherwise. As an example, consider OHTTP used for the purposes of hiding
client-identifying information for a browser telemetry system. It is entirely possible for reports
in such a telemetry system to contain both client-specific telemetry data, such as information
about their specific browser instance, as well as client-identifying inforamtion, such as the client's
location or IP address. Even though OHTTP separates the client IP address from the server via
a relay, the server still learns this directly from the client.

Other relevant examples of insufficient partitioning include TLS Encrypted Client Hello (ECH) {{?I-D.ietf-tls-esni}}
and VPNs. ECH use cryptographic protection (encryption) to hide information from unauthorized parties,
but both clients and servers (two entities) can link user-specific data to user-specific identity (IP address).
Similarly, while VPNs hide identity from end servers, the VPN server has still can see the identity of both the
client and server. Applying privacy partitioning would advocate for at least two additional entities to avoid
revealing both (identity (who) and user actions (what)) from each involved party.

While straightforward violations of user privacy like this may seem straightforward to mitigate, it
remains an open problem to determine whether a certain set of information reveals "too much" about a
specific user. There is ample evidence of data being assumed "private" or "anonymous" but, in hindsight,
winds up revealing too much information such that it allows one to link back to individual
clients; see {{?DataSetReconstruction=DOI.10.1109/SP.2008.33}} and {{CensusReconstruction}}
for more examples of this in the real world.

Beyond the information that is intentionally revealed by applying privacy partitioning, it is also possible
for the information to be unintentionally revealed through side channels. For example, in the two-hop
proxy arrangement described in {{masque}}, Proxy A sees and proxies TLS data between the client and
Proxy B. While it does not directly learn information that Proxy B sees, it does learn information through
metadata, such as the timing and size of encrypted data being proxied. Traffic analysis could be exploited
to learn more information from such metadata, including, in some cases, application data that Proxy A was
never meant to see. Although privacy partitioning does not obviate such attacks, it does increase the cost
necessary to carry them out in practice. See {{security-considerations}} for more discussion on this topic.

# Partitioning Impacts

Applying privacy partitioning to communication protocols leads to a substantial change in communication patterns.
For example, instead of sending traffic directly to a service, essentially all user traffic is routed through
a set of intermediaries, possibly adding more end-to-end round trips in the process (depending on the system
and protocol). This has a number of practical implications, described below.

1. Service operational or management challenges. Information that is traditionally passively observed in the
   network or metadata that has been unintentionally revealed to the service provider cannot be used anymore
   for e.g., existing security procedures such as application rate limiting or DDoS mitigation.
   However, network management techniques deployed at present often rely on information that is exposed by
   most traffic but without any guarantees that the information is accurate.

   Privacy partitioning provides an opportunity for improvements in these management techniques with
   opportunities to actively exchange information with each entity in a privacy-preserving way and requesting
   exactly the information needed for a specific task or function rather than relying on the assumption that
   are derived from a limited set of unintentionally revealed information which cannot be guaranteed to be
   present and may disappear at any time in future.

1. Varying performance effects and costs. Depending on how context separation is done, privacy partitioning may
   affect application performance. As an example, Privacy Pass introduces an entire end-to-end round
   trip to issue a token before it can be redeemed, thereby decreasing performance. In contrast, while
   systems like CONNECT proxying may seem like they would regress performance, oftentimes the highly
   optimized nature of proxy-to-proxy paths leads to improved performance.

   Performance may also push back against the desire to apply privacy partitioning. For example, HTTPS
   connection reuse {{?HTTP2=RFC9113, Section 9.1.1}} allows clients to use an existing HTTPS session created
   for one origin to interact with different origins (provided the original origin is authoritative for
   these alternative origins). Reusing connections saves the cost of connection establishment, but means that
   the server can now link the client's activity with these two or more origins together. Applying privacy
   partitioning would prevent this, while typically at the cost of less performance.

   In general, while performance and privacy tradeoffs are often cast as a zero-sum game, in practice this
   is often not the case. The relationship between privacy and performance varies depending on a number
   of related factors, such as application characteristics, network path properties, and so on.

1. Increased attack surface. Even in the event that information is adequately partitioned across
   non-colluding parties, the resulting effects on the end-user may not always be positive. For example,
   using OHTTP as a basis for illustration, consider a hypothetical scenario where the Oblivious
   Gateway has an implementation flaw that causes all of its decrypt requests to be
   inappropriately logged to a public or otherwise compromised location. Moreover, assume
   that the Target Resource for which these requests are destined does not have such an
   implementation flaw. Applications which use OHTTP with this flawed Oblivious Gateway to
   interact with the Target Resource risk their user request information being made public,
   albeit in a way that is decoupled from user identifying information, whereas applications
   that do not use OHTTP to interact with the Target Resource do not risk this type of disclosure.

1. Centralization. Depending on the protocol and system, as well as the desired privacy properties, the
   use of partitioning may inherently force centralization to a select set of trusted participants.
   As an example, the impact of OHTTP on end-user privacy generally increases proportionally to the
   number of users that exist behind a given Oblivious Relay. That is, the probability of an Oblivious
   Gateway determining the client associated with a request forwarded through an Oblivious Relay decreases
   as the number of possible clients behind the Oblivious Relay increases. This tradeoff encourages
   the centralization of the Oblivious Relays.

# Security Considerations

{{limits}} discusses some of the limitations of privacy partitioning in practice. In general,
privacy is best viewed as a spectrum and not a binary state (private or not). Applied correctly,
partitioning helps improve an end-user's privacy posture, thereby making violations harder to
do via technical, social, or policy means. For example, side channels such as traffic analysis
{{?I-D.irtf-pearg-website-fingerprinting}} or timing analysis are still possible and can allow
an unauthorized entity to learn information about a context they are not a participant of.
Proposed mitigations for these types of attacks, e.g., padding application traffic or generating
fake traffic, can be very expensive and are therefore not typically applied in practice.
Nevertheless, privacy partitioning moves the threat vector from one that has direct access to
user-specific information to one which requires more effort, e.g., computational resources,
to violate end-user privacy.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
