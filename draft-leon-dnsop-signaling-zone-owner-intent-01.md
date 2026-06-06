---
title: "Signaling Zone Owner Intent"
abbrev: "Signaling Zone Owner Intent"
docname: draft-leon-dnsop-signaling-zone-owner-intent-01
date: {DATE}
category: std

ipr: trust200902
area: Internet
workgroup: DNSOP Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: L. Fernandez
    name: Leon Fernandez
    organization: The Swedish Internet Foundation
    country: Sweden
    email: leon.fernandez@internetstiftelsen.se
 -
    ins: E. Bergström
    name: Erik Bergström
    organization: The Swedish Internet Foundation
    country: Sweden
    email: erik.bergstrom@internetstiftelsen.se
 -
    ins: J. Stenstam
    name: Johan Stenstam
    organization: The Swedish Internet Foundation
    country: Sweden
    email: johan.stenstam@internetstiftelsen.se
 -
    ins: S. Crocker
    name: Steve Crocker
    organization: Edgemoor Research Institute
    country: United States
    email: steve@shinkuro.com

normative:

informative:

--- abstract

This document introduces a standardized mechanism for zone owners to
signal their intent regarding DNS provider responsibilities through
DNS itself. It defines two new DNS RRtypes — HSYNC3 (Horizontal
Synchronization, per-provider enrollment) and HSYNCPARAM
(zone-wide multi-provider policy) — that together enable zone
owners to designate which providers are authorized to serve and/or
sign their zones, control whether providers or the zone owner
manages the NS RRset, and specify zone transfer chain
configurations.

The HSYNC3 and HSYNCPARAM records allow DNS providers to discover
each other and establish secure communication channels, either via
DNS messages secured by JWS signatures (or legacy SIG(0)
signatures) or via a RESTful API secured by TLS. This
provider-to-provider communication via Agents enables automated
coordination for tasks such as NS RRset management, zone
transfers, and DNSSEC-related operations. This specification
covers the provider discovery and communication establishment
aspects.

The document defines two new roles to facilitate this synchronization:
the Agent responsible for provider-to-provider communication and the
Combiner which merges unsigned zone data from the zone owner with
managed data from providers.

While a distributed DNSSEC multi-signer architecture (similar to 
"model 2" in RFC8901) is an important application of this framework, 
the HSYNC-based signaling supports broader provider synchronization
needs. 

The specific synchronization algorithms for multi-signer operation are
described in {{?I-D.draft-ietf-dnsop-dnssec-automation}}. The DNS CHUNK
transport mechanism used for structured data exchange between agents is
defined in {{?I-D.draft-berra-dnsop-chunk-transport}}. Specification
for how to express the multi-signer synchronization algorithms over
provider-to-provider communication is left for a follow-up document.

TO BE REMOVED: This document is being collaborated on in Github at:
[https://github.com/johanix/draft-leon-dnsop-signaling-zone-owner-intent](https://github.com/johanix/draft-leon-dnsop-signaling-zone-owner-intent).
The most recent working version of the document, open issues, etc, should all be
available there.  The authors (gratefully) accept pull requests.

--- middle

# Introduction

DNS zone owners often need to work with multiple DNS providers to
serve their zones. These providers may have different
responsibilities - some may sign the zone, some may only serve it, and
some may do both. Traditionally, the configuration of these providers
and their responsibilities has been handled through manual processes
and provider-specific mechanisms.

This document presents a standardized mechanism for zone owners to
signal their intent regarding DNS provider responsibilities through
DNS itself. It defines two new DNS RRtypes, HSYNC3 and HSYNCPARAM,
that together allow zone owners to:

* Designate which providers should serve the zone (via the
  HSYNCPARAM `servers` key).

* Designate which providers should sign the zone (via the
  HSYNCPARAM `signers` key).

* Control whether providers or the zone owner manages the NS RRset
  (via the HSYNCPARAM `nsmgmt` key).

* Specify on a provider-level how the zone transfer chain should
  be set up (via the HSYNC3 Upstream field).

* Enable providers to locate each other and establish secure
  communication.

By publishing this information in the DNS, zone owners ensure that
all providers eventually converge on the same configuration,
modulo zone-transfer propagation delays and the integrity of the
zone-transfer path itself (see {{security-considerations}}). This
enables automated coordination between providers for tasks like:

* NS RRset management across multiple providers.

* Addition or removal of providers.

* Transition between different signing configurations.

* Management of DNSSEC-related records when multiple signers are used.

* Zone transfer chain configuration.

The intent of this document is to define a framework for secure
provider-to-provider communication, based directly on intent expressed
by the zone owner.

The mechanism by which agents exchange structured data (zone
contributions, key state signals, confirmations, etc.) is defined in
{{?I-D.draft-berra-dnsop-chunk-transport}}, which specifies a DNS
CHUNK transport with optional JWK-based authentication and encryption.

Lots of work remains with specification of the details for the various
synchronization processes that will be expressed over this
framework. That will take place in follow-up documents.

Knowledge of DNS NOTIFY {{!RFC1996}} and DNS Dynamic Updates
{{!RFC2136}} and {{!RFC3007}} is assumed. DNS SIG(0) transaction
signatures are documented in {{!RFC2931}}.

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**",
"**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",
"**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document
are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Terminology

Provider:
   In the context of this document the term "provider" always indicate
   a "DNS provider", i.e. an entity that provides DNS services,
   eg. DNSSEC-signing and/or authoritative nameservice.
...

# Requirements

The requirements for an architecture facilitating DNS provider
synchronization are defined as follows:

* Zone owners MUST be able to signal to their DNS providers
  information sufficient for the providers to identify each other and
  establish secure communication.

* All signaling from zone owner to DNS providers SHOULD be carried out
  via data in the served zone, ensuring that all providers receive the
  same configuration information at approximately the same time.

* Zone owners MUST be able to explicitly specify which DNS providers
  should serve and/or sign their zones.

* Zone owners MUST be able to signal the intent to onboard an
  additional DNS provider. This MUST automatically initiate the
  appropriate provider synchronization processes.

* Zone owners MUST be able to signal the intent to offboard an
  existing DNS provider. This MUST automatically initiate the
  appropriate provider synchronization processes.

* By engaging DNS providers for signing, the zone owner MUST give up
  control over the following records:
  * All DNSSEC-related records in the zone.
  * Any CDS and/or CSYNC RRsets.

* It SHOULD be possible but NOT MANDATORY for the zone owner to also
  delegate the management of the NS RRset to the set of DNS providers.

* DNS providers MUST be able to locate and establish secure
  communication with each other based on the information
  provided by the zone owner in the DNS via the HSYNC3 RRset
  and the HSYNCPARAM record.

* The architecture SHOULD support both DNS-based and API-based
  communication between providers.

* The architecture SHOULD allow for smooth transitions between
  different provider configurations without service interruption.

# DNS Provider Synchronization Scenarios

The HSYNC-based signaling supports a variety of scenarios where zone
owners need to coordinate multiple DNS providers. The following
scenarios illustrate the range of use cases this mechanism enables:

## Coordinated NS Record Management

A zone owner uses two DNS providers — one signs and serves the
zone while another only serves it. The zone owner publishes one
HSYNC3 record per provider and an HSYNCPARAM record whose
{{servers}} key lists both providers and whose {{signers}} key
lists only the signing one. The providers' Agents establish
secure communication channels, allowing them to coordinate NS
RRset management across all authoritative nameservers without
manual intervention. The zone owner can decide whether to retain
control of NS records or delegate this responsibility to the
providers via the {{nsmgmt}} key.

## Multi-Provider DNSSEC Redundancy

A zone owner needs to eliminate the "signing" single point of failure
in their DNSSEC setup. By contracting with multiple "multi-signer
capable" DNS providers and listing them in the HSYNCPARAM
{{signers}} key, the zone owner enables each provider to:

* Locate other designated providers via the HSYNC3 RRset and
  establish secure communications.

* Coordinate DNSKEY, CDS, CSYNC and NS RRset management.

* Sign the zone using its own DNSKEYs while publishing a DNSKEY RRset
  that includes keys from all authorized signers.
  
* Distribute the signed zone to its authoritative nameservers and
  possibly to non-signing downstream providers.

This creates a fully redundant DNSSEC infrastructure with no single
point of failure.

## Provider Transition Management

A zone owner wishes to replace their current DNSSEC-signing provider
with a new one. Using HSYNC3 + HSYNCPARAM, they are able to:

* Add a new HSYNC3 record with State="ON" for the incoming provider
  and add its Label to the HSYNCPARAM {{signers}} key, initiating
  the onboarding process.

* Allow the automated synchronization between providers to handle key
  exchange and transition.

* Once the new provider is fully operational, change the HSYNC3
  State for the outgoing provider to "OFF" (and remove its Label
  from {{signers}}) when convenient.

* The providers then automatically coordinate the safe removal of the
  outgoing provider's data.

This entire process maintains continuous service and valid signatures
while transitioning between DNS providers.

## Delegated NS Management

A zone owner wants DNS providers to handle NS RRset management while
retaining control of other zone data. By setting the {{nsmgmt}} key
in HSYNCPARAM to `"agent"`, the zone owner explicitly delegates NS
management responsibility to the DNS providers. The DNS providers
then coordinate to maintain a consistent NS RRset across all
authoritative servers, adding or removing nameservers as needed
based on the current set of authorized providers.

## Phased Migration to Multi-Provider Architecture

A zone owner currently using a single provider wants to implement a
more robust architecture but prefers a gradual transition. They can:

* First add a single HSYNC3 record designating their current
  provider, plus an HSYNCPARAM record listing that provider in
  {{servers}} (and, if signing, in {{signers}}), making no
  immediate operational changes.

* Later add a second HSYNC3 record for the additional provider and
  extend the HSYNCPARAM lists accordingly.

* Allow the providers to automatically coordinate the transition.

* Optionally delegate NS management to the providers by changing
  the {{nsmgmt}} key from `"owner"` to `"agent"`.

This approach enables a controlled, phased migration to a more
resilient multi-provider architecture.

# The Agent: Integrated Signer vs Separate Agent

In a distributed setup there must be a service located with each
DNS Provider that manages communication with other DNS
Providers. This is referred to as the Agent.

It is possible to implement support for the synchronization and
communication needs directly into an existing component of the
provider's provisioning infrastructure (which may be as simple as an
authoritative nameserver with or without the ability to do online
DNSSEC signing). In that case this component implements the Agent
functionality.

However, it is also possible to separate the synchronization and
communication needs into a separate agent. This Agent sits next to the
existing infrastructure, and is under the same administrative control
(the "DNS Provider"), but is a separate piece of software. Each Agent
is configured as a "secondary nameserver" and receives the (usually
signed) zone. In this document the functional separation using a
distinct Agent is used for clarity, not as a statement on preferred
implementation choice.

The "separate Agent" design has the major advantage of leaving the
DNSSEC-signer (if any) outside of the synchronization and
communication complexity. The requirements are only that the Agent is
treated as a normal secondary (it receives NOTIFY messages and is able
to request zone transfers).

# Source of Truth

A common design for DNSSEC signing (regardless of multi-signer) is to
use a separate, bump-on-the-wire Signer. This is a Signer that
receives the unsigned zone via an incoming zone transfer, signs the
zone, and publishes the signed zone via an outbound zone transfer. In
such a design the source of truth has been split up between the "zone
owner" (source of truth for all non-DNSSEC zone data), and the Signer
(source of truth for all DNSSEC data in the zone plus the DNSKEY RRset).

In the proposed architecture the source of truth is further split up
into three participants:

 * The zone owner is the source of truth for all unsigned zone data,
   except DNSSEC data and possibly the NS RRset.
   
 * The Signer is the source of truth for all data generated via DNSSEC
   signing: own DNSKEYs, NSEC/NSEC3 RRs, RRSIGs, etc.
   
 * The Agent is the source of truth for the RRsets that must be kept in
   sync across all the Signers for the zone. This includes the DNSKEYs
   from other providers, CDS and CSYNC RRsets. Possibly also the NS RRset.

The NS RRset is an interesting special case. Traditionally the NS
RRset is maintained by the zone owner, but based on data from the DNS
providers (as authoritative nameservers is a primary service for the
DNS provider). However, in the proposed architecture the NS RRset
should preferably be maintained by the Agents. For this reason the
proposed design makes control of the NS RRset explicit and the
responsibility of the zone owner to choose whether to retain control
or delegate to the Agents. Hence:

 * The Agent is the source of truth for the NS RRset, subject to the
   policy of the zone owner expressed in the {{nsmgmt}} key of the
   HSYNCPARAM record.

Making the control of the NS RRset explicit is useful regardless of
whether a zone uses multiple signers or single signer, as this makes
the zone owner intent explicit.

To be able to keep the Signer as simple as possible, the changes to the
NS, DNSKEY, CDS and CSYNC RRsets must be introduced into the unsigned
zone before the zone reaches the Signer. Likewise, to keep the zone
owner as simple as possible (i.e. not involved in the details of the
multi-signer automation) these changes must be introduced into the
unsigned zone after the zone leaves the zone owner.

## The Combiner

The consequence of these requirements is that the DNSKEY, CDS and
CSYNC RRsets (and possibly the NS RRset) are maintained via a separate
piece of software inserted between the zone owner and the Signer. This
is referred to as the Combiner.

The Combiner has the following features:

 * It supports inbound zone transfer of the unsigned zone from the
   zone owner.
   
 * It receives updates for the NS, DNSKEY, CDS and CSYNC
   RRsets from the Agent. Typically the mechanism used is DNS UPDATE
   with a TSIG signature, as this is easy to configure in a local
   context. However, other mechanisms, including APIs, are possible.
   
 * It stores all data received from the Agent separate from
   the zone data received from the zone owner.
   
 * Whenever it receives a new unsigned zone from the zone owner it
   COMBINES zone data from the zone owner (the majority of the zone)
   with specific zone data under control of the Agent: three specific
   RRsets, all in the apex of the zone: the DNSKEY, CDS and CSYNC
   RRsets. According to zone owner policy expressed in the
   HSYNCPARAM {{nsmgmt}} key it will also update the NS RRset.
   
 * It is policy free (apart from being limited to the four specified
   RRsets). I.e. the Combiner is not making any judgement about what
   data to include in the zone from the four defined RRsets. That
   judgement is the role of the Agent.
   
 * It does not sign the zone.
 
 * It provides outbound zone transfer of the combined zone to the
   Signer.

Example setup with two signers showing the logical flow of zone data
between the zone owner, the Combiner, the Signer and the Agent:

~~~
                            +--------------+
                            |     owner    |
               xfr          +-+---------+--+    xfr
            /----------------/           \----------------------\
           /                                                     \
    +-----+----+    DNS  +-------+ DNS/API +-------+  DNS    +----+-----+
    | combiner +<--------+ agent +---------+ agent +-------->+ combiner |
    +-----+----+  UPDATE +--+----+         +--+----+ UPDATE  +----+-----+
          |                 ^                 ^                 |
          v xfr             |                 |                 v xfr
    +-----+----+     xfr    |                 |   xfr      +----+-----+
    |  signer  +------------+                 +------------+  signer  |
    +-----+----+                                           +----+-----+
          |                                                     |
          v                                                     v
       +--+--+                                               +--+--+
       | NS  |--+                                            | NS  |+
       +-----+  |--+                                         +-----+|-+
          +-----+  |                                            +---+ |
             +-----+                                              +---+
~~~

## The DNS Provider

A "DNS Provider" is a term that is most commonly used to refer to an
entity that provides authoritative DNS service to one or more zone
owners. In the context of this document it is used to refer to an
entity that provides some subset of the following services:

 * Signing a zone received from the zone owner.
 * Serving the zone via a set of authoritative nameservers.
 * Distributing the signed zone to other downstream DNS Providers.

In addition to the above services a DNS Provider MUST also provide:

* An Agent for synchronization with other DNS Providers
* A Combiner for the management of changes to the zone via
  the synchronization among Agents (if it provides a signer)

I.e. in the setup above there are two DNS Providers, both of which are
"complete" in the sense that they provide all three of the above
services.

# Identifying the Designated DNS Providers

It is the responsibility of the zone owner to choose a set of "DNS
Providers", either internal or external to the zone owner's
organization. These DNS Providers MUST be clearly and uniquely
designated via the HSYNC3 RRset (one record per Provider) and the
HSYNCPARAM record (zone-wide policy referencing the Providers by
Label), both located at the apex of the zone.

The HSYNC3 RRset and HSYNCPARAM record MUST be added, by the zone
owner, to the typically unsigned zone that the zone owner
maintains so that they are visible to the downstream DNS Providers
and their Agents.


TO BE REMOVED BEFORE PUBLICATION:

# Rationale: Two Records, Not One

The signaling described in this document is split across two RR
types: HSYNC3 carries the per-provider enrollment (one record per
provider), and HSYNCPARAM carries zone-wide policy (one record per
zone, structured as a list of key-value pairs). An earlier design
used a single HSYNC record. The two-record model replaces it for
three reasons:

* The single HSYNC record was intended to be a static-and-fixed
  RDATA, but the set of things zone owners need to signal kept
  growing. Each addition meant another fixed field. A
  fixed-schema record does not extend gracefully.

* As fields multiplied, each per-provider record had to carry a
  value for every field, even for fields that did not apply to
  that provider. Most records ended up saying "no" or "0" or
  "off" for most fields.

* The single-record model permitted inconsistent configurations
  on the wire. For example, two providers could each publish an
  HSYNC record claiming a different value for the same zone-wide
  policy field (one saying "the agents handle parent
  synchronization", the other saying "the owner does"). Both
  cannot be right, but the wire format allowed both.

The two-record model prevents the third problem by construction:
zone-wide policy lives in HSYNCPARAM (a single record per zone),
and each provider appears in HSYNCPARAM key values only for the
keys where that provider participates. It avoids the second
problem by listing providers only where relevant rather than
flagging every provider on every field. And it avoids the first
problem by structuring HSYNCPARAM as a registry of keys (analogous
to the SvcParamKey registry for SVCB), so new signaling concerns
can be added by registering a new key rather than by changing the
record's RDATA.

# The HSYNC3 RRset

The HSYNC3 RRset is published at the apex of the zone and consists
of one HSYNC3 record per designated DNS Provider. Each record
identifies one Provider and locates that Provider's Agent. HSYNC3
carries no role information; roles (signer, server, auditor,
etc.) are expressed in the HSYNCPARAM record described in the next
section.

An HSYNC3 record has four fields:

zone.example.    IN HSYNC3  State  Label  Identity  Upstream

State:
    Unsigned 8-bit. Defined values are 1=ON and 2=OFF. The value 0
    is an error. Values 3-127 are presently undefined. Values
    128-255 are reserved for private use. The presentation format
    MUST use the tokens "ON" and "OFF". State semantics are
    described in the section "Semantics of the HSYNC3 State Field"
    below.

Label:
    An unqualified token (NOT a fully qualified domain name) that
    serves as a short handle for this Provider within HSYNCPARAM
    key values. Two HSYNC3 records in the same zone MUST NOT use
    the same Label.

Identity:
    Domain name. Used to uniquely identify the Agent for the DNS
    Provider that this record represents. This is the name under
    which the Agent's URI, SVCB, JWK/KEY discovery records are
    published.

Upstream:
    Either an unqualified Label referring to another HSYNC3 record
    in the same zone, or "." if this Provider has no upstream
    Provider (or the upstream is to be configured manually).

Example:

zone.example.   IN HSYNC3  ON  fox  agent.fox.example.    .
zone.example.   IN HSYNC3  ON  hare agent.hare.example.   fox

In this example the zone has two designated Providers, "fox" and
"hare". "fox" has no upstream; "hare" has "fox" as its upstream.
The unqualified token (Label) used in the Upstream field MUST
match the Label of an HSYNC3 record in the same zone.

## Semantics of the HSYNC3 State Field

The State field signals to all Agents what the status of each DNS
Provider is from the point-of-view of the zone owner. The two
defined values are "ON" and "OFF":

* "ON" means that the DNS Provider is currently a designated DNS
  Provider for the zone (or in the process of being onboarded).

* "OFF" means that the DNS Provider was previously a designated DNS
  Provider for the zone and is in the process of being offboarded.

The "OFF" state matters because the offboarding process typically
involves the remaining DNS Providers, and they need to know which
DNS Provider is being offboarded so that the correct data may be
removed in the correct order (either during the multi-signer
"remove signer" process of {{!RFC8901}} or a simpler "remove auth
nameserver" process).

Once the offboarding process is complete, the HSYNC3 record for the
offboarded DNS Provider may be removed from the zone at the zone
owner's discretion.

State=OFF is also useful during initial setup of a new DNS
Provider. As long as State=OFF, no data from the Provider must be
used by other Providers. However, it is possible to verify that
communication and the discovery records all work as intended.

# The HSYNCPARAM Record

The HSYNCPARAM record is published at the apex of the zone. There
is exactly one HSYNCPARAM record per zone. It carries zone-wide
policy as a list of key-value pairs, structurally similar to the
SVCB record's SvcParamKey list.

The RDATA is a sequence of key-value pairs. Each key has a 16-bit
key number registered in the "HSYNCPARAM Keys" registry (see
{{hsyncparam-keys-registry}}). Three key types are defined:

* Flag keys carry no value. The presence of the key signals "true".
* Value keys carry a single value (typically a token).
* List keys carry a comma-separated list of values (typically
  Labels referring to HSYNC3 records in the same zone).

Presentation format places the keys in any order, separated by
whitespace. Flag keys are written as the key name alone. Value
keys are written as `key="value"`. List keys are written as
`key="v1,v2,v3"`.

Example:

zone.example.   IN HSYNCPARAM  nsmgmt="agent" signers="fox,hare" pubkey

The specific keys defined by this document are listed in
{{hsyncparam-keys-registry}}.

## Unknown Keys and Private Use

A receiver that encounters an HSYNCPARAM key number it does not
recognize MUST preserve the key on read-back but MUST NOT take any
action based on it. In presentation format, unknown numeric keys
MUST be written as `keyN` (where N is the decimal key number) so
that they can be parsed unambiguously by tools that have been
updated with the key definition.

The key number range 0-32767 is allocated for IANA-registered keys.
The range 32768-65534 is reserved for Private Use; receivers within
a single administrative domain may assign meaning to keys in that
range without registration. Key number 65535 is reserved and MUST
NOT be assigned.


# HSYNCPARAM Keys

This section defines the eight HSYNCPARAM keys assigned by this
document. The order reflects the typical sequence of decisions a
zone owner makes when configuring a multi-provider deployment:
first the role assignments (who serves the zone, who signs it, who
audits it), then auxiliary policy (NS management, parent
synchronization, in-bailiwick naming, and publication intent for
provider-managed records).

## servers {#servers}

Key number: 0

Type: list of Labels.

The `servers` key signals which Providers are designated to serve
the zone authoritatively. Each value in the list is a Label
matching the Label field of an HSYNC3 record in the same zone.
Providers whose Label appears in `servers` SHOULD configure
themselves as authoritative for the zone.

The Labels used in HSYNCPARAM list keys are unqualified tokens, not
fully qualified domain names. They are short handles defined by
the HSYNC3 records in the same zone; the FQDN of each Provider's
Agent is given by the HSYNC3 Identity field. See {{the-hsync3-rrset}}
for the HSYNC3 record format.

Example:

zone.example. IN HSYNCPARAM servers="fox,hare"

## signers {#signers}

Key number: 1

Type: list of Labels.

The `signers` key signals which Providers are designated to sign
the zone. Each value is an HSYNC3 Label. A Provider whose Label is
in `signers` is typically also in {{servers}}, but this is not
required.

Example:

zone.example. IN HSYNCPARAM servers="fox,hare" signers="fox,hare"

## auditors {#auditors}

Key number: 2

Type: list of Labels.

The `auditors` key signals which Providers (or Provider-like
entities) act as Auditors for the zone. An Auditor observes the
synchronization between other Providers and may flag
inconsistencies.

Example:

zone.example. IN HSYNCPARAM servers="fox,hare" signers="fox,hare" auditors="auditor1"

## nsmgmt {#nsmgmt}

Key number: 3

Type: value, one of "owner" or "agent".

The `nsmgmt` key signals who is responsible for the contents of the
NS RRset for the zone. Two values are defined:

* "owner" — the zone owner is responsible for the NS RRset.
  Agents MUST NOT instruct their local Combiner to update the NS
  RRset.

* "agent" — the Providers' Agents collectively are responsible for
  the NS RRset. Agents whose Provider is listed in {{signers}}
  MUST instruct their local Combiner to update the NS RRset based
  on the union of NS records contributed by Providers via
  Agent-to-Agent communication.

If `nsmgmt` is absent, the default is "owner".

In-bailiwick address records (A/AAAA records for nameservers whose
name lies within the zone) are deliberately not covered by
`nsmgmt`. The reasons are to limit the possibility of DNS
Providers polluting the zone's namespace, and to keep the
specification simpler — the concept of delegated NS management is
already new. See {{suffix}} for the separate signaling that lets
Providers add their own nameserver names and addresses, scoped
under a label designated by the zone owner.

Example:

zone.example. IN HSYNCPARAM nsmgmt="agent" signers="fox,hare"

## parentsync {#parentsync}

Key number: 4

Type: value, one of "owner" or "agent".

The `parentsync` key signals who is responsible for synchronizing
delegation information (NS, glue, DS) with the parent zone. Two
values are defined:

* "owner" — the zone owner is responsible for sending updates to
  the parent (via whichever mechanism the parent announces in its
  DSYNC record).

* "agent" — the Providers' Agents collectively are responsible
  for parent synchronization; this is typically coordinated via
  leader election among the Agents.

If `parentsync` is absent, the default is "owner". The specific
mechanism by which the parent receives the update (NOTIFY,
DDNS UPDATE, etc.) is announced by the parent via the DSYNC record
defined in {{!RFC9859}}.

Example:

zone.example. IN HSYNCPARAM nsmgmt="agent" parentsync="agent" signers="fox,hare"

## suffix {#suffix}

Key number: 5

Type: value, a single valid DNS label.

When the `suffix` key is present, DNS Providers MAY add
in-bailiwick address records to the zone for nameservers they
contribute — but only for names below `{suffix}.{zone}`. The value
of the key MUST be a single valid DNS label (not a fully qualified
domain name).

If `suffix` is absent, Providers MUST NOT add in-bailiwick
nameserver records (NS or address records) to the zone. The purpose
of this restriction is to prevent unintended namespace collisions
between owner-controlled names and Provider-added names.

If `suffix="ns"` is present in HSYNCPARAM, then a Provider with
Label "fox" MAY add:

zone.example.         IN NS    fox1.ns.zone.example.
fox1.ns.zone.example. IN A     1.2.3.4
fox1.ns.zone.example. IN AAAA  2001::53

and similarly for other Providers. Providers MUST coordinate
amongst themselves (via Agent-to-Agent communication) to avoid
name collisions below `{suffix}.{zone}`.

Example:

zone.example. IN HSYNCPARAM nsmgmt="agent" signers="fox,hare" suffix="ns"

## pubkey {#pubkey}

Key number: 6

Type: flag.

Keys `pubkey` and `pubcds` (see {{pubcds}}) instruct DNS Providers
to publish KEY and CDS/CDNSKEY records on behalf of the zone
owner at well-known names. Without these signals, providers would
have to scan customer zones for various conventional content (per
{{!RFC9615}} §3.1 for CDS, and similar conventions for other RR
types). The HSYNCPARAM record provides a single,
designed-for-purpose place where the zone owner expresses such
intent, making the signaling explicit rather than implicit in zone
content.

The `pubkey` flag signals the zone owner's intent that each
Provider SHOULD publish the child's SIG(0) KEY at the special name
`_sig0key.{child}._signal.{their-ns-name}.` in their own
zone. The use case for this key is the SIG(0) bootstrap mechanism
for the cross-zone-cut DNS UPDATE messages defined in
{{?I-D.ietf-dnsop-delegation-mgmt-via-ddns}}.

The `_signal` label in the name pattern is registered in the
"Underscored and Globally Scoped DNS Node Names" registry
{{!RFC8552}} by {{!RFC9615}}.

Example:

zone.example. IN HSYNCPARAM signers="fox,hare" pubkey

## pubcds {#pubcds}

Key number: 7

Type: flag.

The `pubcds` flag signals the zone owner's intent that each
Provider SHOULD publish the zone's CDS and/or CDNSKEY records at
the special name `_dsboot.{child}._signal.{their-ns-name}.` in
their own zone, per the DNSSEC bootstrap mechanism defined in
{{!RFC9615}}.

This signal replaces the implicit RFC 9615 §3.1 convention by
which Providers would otherwise scan customer zones for CDS or
CDNSKEY content. Under `pubcds` the zone owner's intent is
explicit; under absence of `pubcds`, Providers MUST NOT publish
CDS or CDNSKEY records on behalf of the zone.

Example:

zone.example. IN HSYNCPARAM signers="fox,hare" pubkey pubcds


# Linking HSYNC3 and HSYNCPARAM {#linking-hsync3-and-hsyncparam}

HSYNC3 and HSYNCPARAM are designed to work together: HSYNC3 names
the Providers (one record each), and HSYNCPARAM expresses
zone-wide policy that references those Providers by Label.
Resolving a policy decision such as "is provider X a signer?"
therefore requires consulting both records.

The signaling appears in the zone the Agent receives via zone
transfer. An Agent does not perform DNS lookups to resolve these
links; it analyzes the HSYNC3 RRset and the HSYNCPARAM record
already present in the zone. This is local zone analysis, not
recursive resolution.

The procedure is straightforward. To determine whether the
Provider identified by HSYNC3 Label "fox" is a signer for the
zone, an Agent:

1. Examines the HSYNCPARAM record at the zone apex.

2. Reads the value of the `signers` key (a list of Labels).

3. Checks whether "fox" appears in that list.

The same pattern applies to all HSYNCPARAM list keys (`servers`,
`signers`, `auditors`): the value is a list of Labels, each
referring to an HSYNC3 record in the same zone. Conversely, an
HSYNC3 Label that does not appear in any HSYNCPARAM list key is
simply not assigned that role.

A Label referenced by an HSYNCPARAM list key MUST match the Label
field of an HSYNC3 record in the same zone. An HSYNCPARAM list
value that does not match any HSYNC3 Label SHOULD be logged by the
Agent and treated as if absent.

Example:

zone.example.    IN HSYNC3      ON  fox   agent.fox.example.    .
zone.example.    IN HSYNC3      ON  hare  agent.hare.example.   fox
zone.example.    IN HSYNCPARAM  servers="fox,hare" signers="fox" nsmgmt="agent"

In this example, both "fox" and "hare" serve the zone (both are in
`servers`), but only "fox" signs the zone (only "fox" is in
`signers`). NS management is delegated to the Agents
(`nsmgmt="agent"`).


# Communication Between Agents

For the communication between Agents there are two choices that need to
be made among the designated Agents for a zone. The first is what
"transport" to use for the communication. The second is what
"synchronization" model to use when executing future synchronization
processes.

The two defined transport alternatives are:

* DNS-based communication (mandatory to support)
* REST API-based communication

Each has pros and cons and at this point in time it is not clear that
one always is better than the other. To simplify the choice of
transport DNS-based communication is mandatory to support and the REST
API-based communication may only be used if all Agents support
it. Supported transports are signaled in the Provider-Synchronization
EDNS(0) Option (see {{provsync-option}}).

The two defined synchronization alternatives are:

* Leader/Follower synchronization (mandatory to support)
* Peer-to-Peer synchronization

Just as for transport, supported synchronization models are signaled
in the Provider-Synchronization EDNS(0) Option (see {{provsync-option}}).

Regardless of the synchronization model and communication method used,
the Agents SHOULD exchange all needed information about the zone and
the DNS Provider they represent to enable the synchronization
processes to execute correctly. This includes notifications about
changes to DNSKEYs, changes to the NS RRset, etc. Depending on
synchronization model it may also include instructions for changes to
the zone.

In all cases the information published by a DNS provider to allow
other providers to locate its Agent MUST be DNSSEC-signed.

## Agent Communication via DNS

This transport alternative is based on the observation that all the
communication needs between Agents can be expressed via DNS
messages. Structured data (zone contributions, key state signals,
confirmations) is carried in DNS NOTIFY messages using the CHUNK
EDNS(0) option defined in {{I-D.draft-berra-dnsop-chunk-transport}}.
Synchronization state is communicated via the Provider-Synchronization
EDNS(0) option defined in this document.

The CHUNK transport optionally provides payload authentication via
JWS signatures and confidentiality via HPKE encryption, with
cryptographic keys discovered from JWK records published in the DNS.

This model builds on the approach used by
{{?I-D.draft-berra-dnsop-keystate}} for delegation synchronization
between child and parent, which has already been implemented and
shown to work.

## Agent Communication via REST API

REST APIs are well-known and a natural fit for many distributed
systems. The challenge is mostly in the initial setup of secure
communication. The certificates need to be validated, preferably
without a requirement on trusting a third party CA. The API endpoints
for each Agent need to be located. Once secure communication has been
established, using a REST API for Agent communication is
straight-forward.

## Locating Remote Agents

When an Agent receives a zone via zone transfer from the Signer it
analyzes the zone to see whether it contains an HSYNC3 RRset. If
there is no HSYNC3 RRset the zone MUST be ignored by the Agent
from the point-of-view of provider synchronization.

If the zone contains an HSYNC3 RRset, the Agent MUST analyze it to
identify the other Agents for the zone via the Identity field in
each HSYNC3 record. If any of the other Agents identified by the
HSYNC3 RRset is previously unknown to this Agent then secure
communication with this other Agent MUST be established.

Secure communication can be achieved via various transports and it
is up to the Agents named by the zone's HSYNC3 RRset to determine
amongst themselves. This document proposes two transports: "DNS"
and "API". "DNS" is designated as a baseline that Agents MUST
support to be compliant.

The following two subsections describe the mechanism by which an Agent
SHOULD locate a remote Agent and establish secure DNS-based and
API-based communications, respectively.

### Locating a Remote DNS Transport Agent

Locating a remote Agent using the DNS mechanism consists of the
following steps:

 * Lookup and DNSSEC-validate a URI record for the DNS protocol for
   the HSYNC3 Identity. This provides the domain name and port to
   which DNS messages should be sent.

 * Lookup and DNSSEC-validate the SVCB record of the URI record target
   to get the IP addresses to use for communication with the remote
   Agent.

 * If both the URI record and the SVCB record both include information
   about the target port then the port information in the SVCB MUST
   take precedence.

 * Lookup and DNSSEC-validate the JWK record(s) of the URI record
   target name. This provides the cryptographic keys needed for
   authenticating and optionally encrypting communication with the
   remote Agent. The JWK record type is defined in
   {{I-D.draft-berra-dnsop-chunk-transport}}.

 * If no JWK record is found, fall back to looking up and
   DNSSEC-validating the KEY record of the URI record target name.
   This enables verification of the SIG(0) public key of the remote
   Agent once communication starts. The KEY record is the legacy
   mechanism; new implementations SHOULD publish JWK records.

Example: given the following HSYNC3 record for a remote Agent:

zone.example. IN HSYNC3  ON  remote  agent.provider.com. .

The local Agent will look up the URI record for agent.provider.com:

_dns._tcp.agent.provider.com.  IN  URI  10 10 "dns://dns.agent.provider.com:5399/"
_dns._tcp.agent.provider.com.  IN  RRSIG URI …

which triggers a lookup for dns.agent.provider.com. SVCB to get the IPv4
and IPv6 addresses as ipv4hints and ipv6hints in the response to the
SVCB query:

dns.agent.provider.com.   IN  SVCB  1 . ipv4hint=5.6.7.8 ipv6hint=2001::53
dns.agent.provider.com.   IN  RRSIG SVCB …

and also a lookup for the JWK record(s) for dns.agent.provider.com:

dns.agent.provider.com.  IN  JWK  {"kty":"EC","crv":"P-256",...,"use":"sig"}
dns.agent.provider.com.  IN  JWK  {"kty":"OKP","crv":"X25519",...,"use":"enc"}
dns.agent.provider.com.  IN  RRSIG JWK …

The signing key (use="sig") enables verification of JWS-signed
payloads from the remote Agent. The encryption key (use="enc")
enables HPKE-encrypted communication with the remote Agent. Both
key types and their use are defined in
{{I-D.draft-berra-dnsop-chunk-transport}}.

If no JWK record is available, the Agent falls back to the KEY record:

dns.agent.provider.com.  IN  KEY …
dns.agent.provider.com.  IN  RRSIG KEY …

Once all the DNS lookups and DNSSEC-validation of the returned data
has been done, the local Agent is able to initiate communication with
the remote Agent and verify the identity of the responding party via the
validated JWK or KEY record.


### Locating a Remote API Transport Agent

Locating a remote Agent using the API mechanism consists of the
following steps:

* Lookup and DNSSEC-validate the URI record for the HTTPS protocol
  for the HSYNC3 Identity. This provides the base URL that will be used
  to construct the individual API endpoints for the REST API. It also
  provides the port to use.
  
* Lookup and DNSSEC-validate the SVCB record for the URI record
  target.  This provides the IP-addresses to use for communication
  with the Agent. 
  
* If both the URI record and the SVCB record both include information
  about the target port then the port information in the SVCB MUST
  take precedence.
  
* Lookup and DNSSEC-validate the TLSA record for the port and protocol
  specified in the URI record. This will enable verification of the
  certificate of the remote Agent once communication starts.

Example: given the following HSYNC3 record for a remote Agent:

zone.example.     IN HSYNC3  ON  remote  agent.provider.com. .

the local Agent will look up the URI record for agent.provider.com:

_https._tcp.agent.provider.com.  IN  URI  10 10 “https://api.provider.com:443/api/v2/”
_https._tcp.agent.provider.com.  IN  RRSIG URI …

which triggers a lookup for api.provider.com IPv4 and IPv6
addresses as hints in an SVCB RR:

api.provider.com.   IN  SVCB 1 ipv4hint=1.2.3.4 ipv6hint=2001::bad:cafe:443
api.provider.com.   IN  RRSIG SVCB …

Now we know the IP-address and the port as well as the base URL to
use. Finally the TLSA record for _443._tcp.api.provider.com is
looked up, with a response that may look like this:

  _443._tcp.api.provider.com.  IN  TLSA 3 1 1 ….
  _443._tcp.api.provider.com.  IN  RRSIG TLSA …

Once all the DNS lookups and DNSSEC-validation of the returned data
has been done, the local Agent is able to initiate communication with
the remote Agent and verify the identity of the responding party via the
TLSA record for the remote Agent's certificate.

#### Fallback to DNS-based Communication

If the API-based communication fails, either because needed DNS
records are missing, the TLSA record fails to validate the remote Agents
certificate or the remote Agent simply doesn't respond, the local Agent
MUST fall back to DNS-based communication.

## The Initial HELLO Phase

When two Agents need to communicate with each other for the first time
(because they are both designated DNS providers for the same zone), they
need to establish secure communication. This is done in a "HELLO"
phase where the Agents exchange information about their capabilities.

If all the information needed for API-based transport for the remote
party was available, the Agent SHOULD attempt an API-based HELLO. If,
however, this fails for some reason, it should fall back to DNS-based
HELLO.

### DNS-based HELLO Phase

When using DNS-based communication the HELLO phase is done by sending
a NOTIFY(SOA) for the zone that triggered the need for
communication. The NOTIFY message MUST contain both a
Provider-Synchronization EDNS(0) Option (see {{provsync-option}}) and
a CHUNK EDNS(0) option (see {{I-D.draft-berra-dnsop-chunk-transport}})
carrying the HELLO payload.

In the Provider-Synchronization EDNS(0) Option the OPERATION field
MUST have the value "HELLO" (1). Furthermore, the Agent signals its
transport and synchronization capabilities in the TRANSPORT and
SYNCHRONIZATION fields. The CHUNK payload contains the sender's
identity and the zone that triggered the communication. The payload
is optionally signed using JWS with the Agent's signing key published
as a JWK record, or signed using the SIG(0) key published as a KEY
record for backward compatibility.

In the response to the NOTIFY, the remote Agent does the same and the
two Agents can now verify each other's identity and are also aware of
the other Agent's transport and synchronization capabilities.

### API-based HELLO Phase

When using API-based communication the HELLO phase is done by sending
a REST API POST request to the remote Agent at the "/hello"
endpoint. The request MUST contain a JSON encoded object with the
following fields:

* "transport": The transport capabilities of the local Agent.
* "synchronization": The synchronization capabilities of the local Agent.

The response MUST contain a JSON object with the following fields:

* "transport": The transport capabilities of the remote Agent.
* "synchronization": The synchronization capabilities of the remote Agent.

### Interpretation of the HELLO Responses

Once an Agent has received HELLO responses from all other Agents that
are designated signers for the zone, it knows the capabilities of the
Agents as a group. It can then use this information to determine which
transport to use:

* If all Agents support API-based communication, the Agents will use
  API-based communication for this zone.

* If one or more Agents only support DNS-based communication, the
  Agents will use DNS-based communication for this zone.

Likewise, each Agent now knows the provider synchronization
capabilities of the other Agents and can determine which
synchronization model to use:

* If all Agents support the Peer-to-Peer synchronization model, the
  Agents will use the Peer-to-Peer synchronization model for this
  zone.

* If one or more Agents only support the Leader/Follower
  synchronization model, the Agents will use the Leader/Follower
  synchronization model for this zone.

## Provider-Synchronization EDNS(0) Option Format {#provsync-option}

This document uses an Extended Mechanism for DNS (EDNS0) {{!RFC6891}}
option to include DNS Provider synchronization information in DNS
messages.

The Provider-Synchronization EDNS(0) option is structured as follows:

~~~
                                               1   1   1   1   1   1
       0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 0:  |                            OPTION-CODE                        |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 2:  |                           OPTION-LENGTH                       |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 4:  |           OPERATION           |           TRANSPORT           |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 6:  |    SYNCHRONIZATION-MODEL      |                               /
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 7:  / OPERATION-BODY                                                /
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
~~~

Field definition details:

OPTION-CODE:
    2 octets / 16 bits (defined in {{!RFC6891}}) contains the value TBD
    for Provider-Synchronization.

OPTION-LENGTH:
    2 octets / 16 bits (defined in {{!RFC6891}}) contains
    the length of the payload (everything after OPTION-LENGTH) in
    octets and should be 3 plus the length of the OPERATION-BODY field
    (which may be zero octets long).

OPERATION:
    8 bits. Signals the type of operation the message performs.
    Defined values are listed in {{defined-operations}}.

TRANSPORT:
    8 bits. Encodes the transport capabilities of the Agent. With
    8 bits it is possible to define up to 8 different transports of
    which this document defines two: DNS and API.

SYNCHRONIZATION-MODEL:
    8 bits. Encodes the synchronization capabilities of the Agent.
    With 8 bits it is possible to define up to 8 different
    synchronization models of which this document identifies two:
    Leader/Follower and Peer-to-Peer.

OPERATION-BODY:
    Variable-length. Used to carry operation-specific parameters.

### Encoding Transport Capabilities in the Provider-Synchronization EDNS(0) Option

An Agent signals the union of its transport capabilities by setting the
corresponding bits to 1.

0: DNS transport supported (baseline, MUST be supported by all Agents)

1: API transport supported

2-7: unused

### Encoding Synchronization Capabilities in the Provider-Synchronization EDNS(0) Option

An Agent signals its synchronization capabilities by setting the
corresponding bits to 1.

0: Leader/Follower multi-signer synchronization supported

1: Peer-to-Peer multi-signer synchronization supported

2-7: unused

## Defined Operations {#defined-operations}

The OPERATION field in the Provider-Synchronization EDNS(0) option
identifies the type of message being exchanged. This document defines
the following operations. The structured data payload for each
operation is carried in the CHUNK EDNS(0) option
({{I-D.draft-berra-dnsop-chunk-transport}}) included in the same
DNS message.

### HELLO (1)

The HELLO operation is used during the initial handshake between two
Agents that need to communicate for the first time. The HELLO message
is sent as a DNS NOTIFY with both the Provider-Synchronization option
(OPERATION=HELLO) and a CHUNK option containing a JSON payload with
the sender's identity and the zone that triggered the communication.

The response includes a CHUNK option with the responder's identity
and capabilities. Once both sides have exchanged HELLO messages
successfully, they transition to the operational state.

### BEAT (2)

The BEAT operation (heartbeat) is used for periodic keep-alive
signaling between Agents that have established communication. The
BEAT message carries the sender's identity, the list of zones shared
between the two Agents, and the sender's intended heartbeat interval.

An Agent that does not receive a BEAT from a peer within a
configurable timeout SHOULD consider the peer unreachable and MAY
attempt to re-establish communication via the HELLO phase.

### PING (3)

The PING operation is used to test connectivity with a remote Agent.
The PING carries a random nonce that the responder echoes back in the
response. This enables round-trip verification of the communication
path.

### SYNC (4)

The SYNC operation is used for Agent-to-Agent zone data
synchronization. The CHUNK payload contains the zone data that the
sending Agent contributes (DNSKEY, CDS, CSYNC, and optionally NS
records). The receiving Agent processes the data according to its
local policy and returns a confirmation indicating which records were
accepted, removed, or rejected.

### UPDATE (5)

The UPDATE operation is used for Agent-to-Combiner zone data
contributions. It carries the same payload format as SYNC, but the
recipient is the local Combiner rather than a remote Agent. The
Combiner returns an immediate "pending" acknowledgment and processes
the update asynchronously, sending a detailed CONFIRM message once
processing is complete.

### CONFIRM (6)

The CONFIRM operation is used to send an explicit confirmation
message, typically as an asynchronous response to a previously
received SYNC or UPDATE. The CHUNK payload contains the distribution
ID of the original message, the processing status (success, partial,
or error), and per-record detail of which records were accepted,
removed, or rejected.

### RFI (7)

The RFI (Request For Information) operation is used to request
specific data from a remote Agent or Signer. The RFI message includes
an RFI subtype indicating what information is being requested:

* SYNC: Request all zone data from the peer.
* UPSTREAM: Request zone data from an upstream Agent.
* DOWNSTREAM: Request zone data from a downstream Agent.
* KEYSTATE: Request the key inventory from a Signer.

The response contains the requested data in the CHUNK payload.

### KEYSTATE (8)

The KEYSTATE operation is used for DNSSEC key lifecycle signaling
between an Agent and its Signer. The CHUNK payload includes the zone
name, key tag, algorithm, and a signal indicating the key state
transition:

* Signals from Agent to Signer: "propagated" (key has been
  distributed to all providers), "rejected" (key was rejected),
  "removed" (key has been removed from zones).

* Signals from Signer to Agent: "published" (new key created),
  "retired" (key retired), "inventory" (complete key inventory).

The KEYSTATE operation enables coordinated key rollovers across
multiple providers.

# Sequence Diagram Example of Establishing Secure Comms - "The Hello Phase"

The procedure of locating another Agent and establishing a secure
communication, referred to as "The Hello Phase" is exemplified in the
sequence diagram below.

The procedure is as follows:

1. The Agents receive a zone via zone transfer. By
   analyzing the HSYNC3 RRset each Agent becomes aware of the
   identities of the other Agents for the zone. I.e. each Agent
   knows which other Agents it needs to communicate with.
   Communication with each of these, previously unknown, remote
   Agents is referred to as "NEEDED".

2. Each Agent starts acquiring the information needed to establish secure
   communications with any previously unknown Agents. Here we only
   illustrate the baseline case where DNS-based communications is to
   be used in the following phase. Once all needed information has
   been collected the communication with this remote Agent is considered
   to be "KNOWN".

3. Once an Agent has received the required information (URI, SVCB and
   KEY records in the baseline case) it sends a NOTIFY message with a
   dedicated Provider-Synchronization OPT code with OPERATION="HELLO".
   The sender uses this OPT field to signal its transport and synchronization
   capabilities. Similarly, the responder signals its capabilities
   using the same field.

4. When an Agent either gets a NOERROR response to its NOTIFY OPT(hello)
   message or responds with a NOERROR, it transitions out of "The
   Hello Phase" with the exchanging party and they transition to the
   next phase where they start sending NOTIFY OPT(beat) signals
   instead. The communication with the remote Agent is now considered to
   be in the "OPERATIONAL" state.

In the case where one Agent is aware of the need to communicate with
another Agent, but the other is not (eg. the zone transfer was delayed
for one of them), the slower one SHOULD respond with a RCODE=REFUSED
to any NOTIFY OPT(hello) it receives. Once it is ready, it will send
its own NOTIFY OPT(hello) which should be responded to with a
RCODE=NOERROR.

~~~
+----------+                 +----------+                        +----------+
|  Owner   |                 | Agent A  |                        | Agent B  |
+----------+                 +----------+                        +----------+
     |                            |                                    |
     |      AXFR(sign-me.se.)     |                                    |
     |--------------------------->|                                    |
     |      AXFR(sign-me.se.)     |                                    |
     |---------------------------------------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |  QUERY _dns._tcp.agent-b.se. URI?  |
     |                            |----------------------------------->|
     |                            |  QUERY ns.agent-b.se. SVCB?        |
     |                            |----------------------------------->|
     |                            |  QUERY ns.agent-b.se. KEY?         |
     |                            |----------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |  NOTIFY sign-me.se. OPT(hello)     |
     |                            |----------------------------------->|
     |                            |  NOERROR sign-me.se. OPT(hello)    |
     |                            |<-----------------------------------|
     |                            |                                    |
     |                            |                                    |
     |                            |  NOTIFY sign-me.se. OPT(beat) |
     |                            |----------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |  NOTIFY sign-me.se. OPT(beat) |
     |                            |<-----------------------------------|
     |                            |                                    |
     |                            |                                    |
     |                            |                                    |

~~~

# Responsibilities of an Agent

Each Agent has certain responsibilities, depending on supported
transports methods.

## Enabling Remote Agents to Locate This Agent

For a group of Agents to be able to communicate securely and synchronize
data for a zone, each Agent must ensure that the DNS records needed for
secure communication with other Agents are published:

  * URI, SVCB and JWK records required for DNS-based communication
    with CHUNK transport (see {{I-D.draft-berra-dnsop-chunk-transport}}).
    For backward compatibility, KEY records for SIG(0) MAY also be
    published.

  * URI, SVCB and TLSA records required for API-based communication
    secured by TLS (if supported).

  * All of the above MUST be published in a DNSSEC-signed zone under
    the domain name that is the identity of the Agent.

## Exchanging Zone Data Between Agents

When using DNS transport between Agents, the following types of
information need to be conveyed between parties:

1. Notifications and structured data (sent as DNS NOTIFY carrying a
   CHUNK EDNS(0) option, as defined in
   {{I-D.draft-berra-dnsop-chunk-transport}}).

2. Provider synchronization state (sent via the
   Provider-Synchronization EDNS(0) Option).

3. Confirmations and responses (sent as DNS NOTIFY carrying a CHUNK
   EDNS(0) option in the response or as a separate NOTIFY).

The data that each Agent contributes for synchronization with other
Agents includes:

  * The DNSKEY RRset for the zone consisting of the DNSKEYs that the
    local Signer for this DNS Provider uses to sign the zone.

  * The CDS RRset for the zone, representing the KSK that the local
    Signer uses to sign the zone (when needed).

  * The CSYNC RRset for the zone (when needed).

  * The NS RRs for the zone, consisting of the NS records of the
    authoritative nameservers that this DNS Provider is responsible
    for (when NS management is delegated to the Agents).

This data is exchanged between Agents via CHUNK NOTIFY messages
(see {{defined-operations}}). Each Agent sends its zone data
contributions to all other Agents for the zone. The receiving Agent
is responsible for instructing its local Combiner to incorporate
the received data.

# Migration from Single-Signer to Multi-Signer

The migration from a single-signer to a multi-signer architecture
is done by adding the second Provider to the {{signers}} list of
the HSYNCPARAM record. This may be done in several steps.

## Adding HSYNC3 and HSYNCPARAM Records To an Already Signed Zone

Adding HSYNC3 and HSYNCPARAM records to a zone that is already
signed by a single DNS Provider, while keeping that Provider as
the sole signer and the zone owner in charge of NS management, is
a no-op that does not change anything in the zone:

zone.example. IN HSYNC3      ON  provider  agent.provider.com.  .
zone.example. IN HSYNCPARAM  servers="provider" signers="provider"

The zone was already signed by the DNS Provider "provider.com" and
the Provider added any needed DNSSEC records, including DNSKEYs.
The zone NS RRset was managed by the zone owner (the default in
the absence of {{nsmgmt}}). All of this is unchanged by the
addition of the two records.

What does change is the possibility of further migration steps
that build on the now-published signaling.

## Promoting a Provider from Server-Only to Signer

A zone owner may want to start having a Provider sign the zone
without changing which Providers serve it. With the HSYNC3
records already in place, this is signaled by adding the
Provider's Label to the {{signers}} list of HSYNCPARAM. For
example, starting from a zone where "fox" serves but does not
sign:

zone.example. IN HSYNC3      ON  fox   agent.fox.example.   .
zone.example. IN HSYNC3      ON  hare  agent.hare.example.  fox
zone.example. IN HSYNCPARAM  servers="fox,hare" signers="hare"

the zone owner adds "fox" to the signers list:

zone.example. IN HSYNCPARAM  servers="fox,hare" signers="fox,hare"

The HSYNC3 records are unchanged. From this point onward, both
Providers are designated signers, and the multi-signer "add signer"
process (see {{?I-D.draft-ietf-dnsop-dnssec-automation}}) is
initiated by the Agents to bring the new signer's keys into the
joint DNSKEY RRset.

## Delegating NS Management to the Agents

To migrate from owner-maintained NS RRset to Agent-maintained, the
zone owner must first verify that the NS RRset as it would be
computed by the Agents (from the union of their NS contributions)
is in sync with the NS RRset currently published by the zone
owner. After this verification the zone owner adds (or changes)
the `nsmgmt` key in the HSYNCPARAM record to `nsmgmt="agent"`. The
HSYNC3 records are unchanged.

## Migrating from a Multi-Signer Architecture Back to Single-Signer.

If, for some reason, a zone owner wants to migrate back to a
single-signer architecture (i.e. offboarding the second DNS Provider),
the process is essentially the reverse of the migration from
single-signer to multi-signer:

1. The zone owner offboards the second signing DNS Provider (only keeping
   one signing DNS Provider).

Offboarding the second signing DNS Provider is signalled by
removing its Label from the HSYNCPARAM {{signers}} key and
typically setting the HSYNC3 State for that Provider to "OFF". This
initiates the multi-step "remove signer" process (as defined in
{{?I-D.draft-ietf-dnsop-dnssec-automation}}), which removes the
second DNS Provider's data from the zone in a series of steps.

The zone is now essentially back to a single-signer architecture.
Once the offboarding is complete, the zone owner may remove the
HSYNC3 record for the offboarded DNS Provider from the zone.

TO BE REMOVED BEFORE PUBLICATION:
# Rationale

## Choice of the HSYNC Mnemonic

Initially the mnemonic "MSIGNER" was used for the HSYNC RRset. However,
as work progressed it became clear that we want also non-signing DNS
Providers to be able to participate. So the RRset is a signalling
mechanism from zone owner to DNS Providers, some of which may or may
not be instructed to sign the zone. Therefore we suggest the mnemonic
"HSYNC" to indicate that this is a mechanism for "horizontal
synchronization" inside a zone.

But the mnemonic chosen is a very minor point and should a better
suggestion come up it would be great.

## Separation of Agent and Combiner

It is possible to integrate all three multi-signer components (Signer,
Agent and Combiner) into a single piece of software (or two pieces,
depending on the preferred way of slicing the functionality). However,
such a composite module would be a fairly complex piece of software.

This document aims to describe the functional separation of the
different components rather than make a judgement on software design
alternatives.  Hence possible implementation choices are left to the
implementer.

# Security Considerations {#security-considerations}

An architecture for automated multi-provider zone management is a
complex system with a number of components.  The authors believe that
the only way to make such an architecture useful in practice is via
automation. However, automation is a double-edged sword. It can both
make the system more robust and more vulnerable.

While all communication between Agents is authenticated (either via
SIG(0) signatures or TLS), the signalling from the zone owner to the
Agents is via the HSYNC3 RRset and the HSYNCPARAM record in an
unsigned zone. This is a potential attack vector. However, securing
zone transfers from zone owner to DNS providers is a well-known
issue with lots of existing solutions (TSIG, zone transfer via a
secure channel, zone transfer-over-TLS, etc). Employing some of
these solutions is strongly recommended.

From a vulnerability point-of-view this architecture introduces
several new components into the zone signing and publication
process. In particular the Combiner and the Agents are new components
that need to be secure. The Combiner has the advantage of not having
to announce its location to the outside world, as it only needs to
communicate with internal components (the zone owner, the Signer and
the Agent).

The Agent is more vulnerable. It needs to be discoverable by other
Agents and hence it is also discoverable by an adversary. On the
other hand, the Agents are not needed for a new zone to be signed and
published, they are only needed when there are changes that require
the Agents to synchronize, which is an infrequent event. 

Furthermore, should an Agent be unable to fulfill its role during the
execution of a change requiring synchronization, whether it is a
complex multi-signer process or perhaps only a change to the NS RRset,
the synchronization process will simply stop where it is. Regardless
of where the stop (or rather pause) occurs, the zone will be fully
functional (as in available and properly signed). Once the Agent is
able to resume its role, the synchronization process will continue
from where it left off.

# IANA Considerations

**Note to the RFC Editor**: In this section, please replace
occurrences of "(This document)" with a proper reference.

## HSYNC3 RR Type

IANA is requested to add the following entry to the "Resource Record
(RR) TYPEs" registry under the "Domain Name System (DNS) Parameters"
registry group:

Type
: HSYNC3

Value
: TBD

Meaning
: Per-provider enrollment for zone-owner-designated DNS providers

Reference
: (This document)

## HSYNCPARAM RR Type

IANA is requested to add the following entry to the "Resource Record
(RR) TYPEs" registry under the "Domain Name System (DNS) Parameters"
registry group:

Type
: HSYNCPARAM

Value
: TBD

Meaning
: Zone-wide multi-provider policy expressed as key-value pairs

Reference
: (This document)

## A New Registry for HSYNCPARAM Keys {#hsyncparam-keys-registry}

The HSYNCPARAM record carries policy as a list of key-value pairs.
IANA is requested to create and maintain a new registry entitled
"HSYNCPARAM Keys", used by the HSYNCPARAM RR type. The keys defined
by this document are listed below; future assignments in the 8-32767
range are to be made through Specification Required review
{{?BCP26}}.

| KEY         | Mnemonic    | Description                                 | Reference         |
|-------------|-------------|---------------------------------------------|-------------------|
| 0           | servers     | Designated Providers serving the zone       | (This document)   |
| 1           | signers     | Designated Providers signing the zone       | (This document)   |
| 2           | auditors    | Designated Auditors for the zone            | (This document)   |
| 3           | nsmgmt      | Who manages the NS RRset                    | (This document)   |
| 4           | parentsync  | Who synchronizes with the parent zone       | (This document)   |
| 5           | suffix      | DNS label below which Providers may add NS/glue | (This document)   |
| 6           | pubkey      | Publish SIG(0) KEY record under _signal.{nsname} | (This document) |
| 7           | pubcds      | Publish CDS/CDNSKEY records under _signal.{nsname} | (This document) |
| 8-32767     | Unassigned  |                                             | (This document)   |
| 32768-65534 | Private Use |                                             | (This document)   |
| 65535       | Reserved    | MUST NOT be assigned                        | (This document)   |

## New Provider-Synchronization EDNS Option

This document defines a new EDNS(0) option, entitled "Provider-Synchronization",
assigned a value of TBD in the "DNS EDNS0 Option Codes (OPT)" registry

TO BE REMOVED UPON PUBLICATION:
[https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-11](foo)

~~~
   +-------+--------------------------+----------+----------------------+
   | Value | Name                     | Status   | Reference            |
   +-------+--------------------------+----------+----------------------+
   | TBD   | Provider-Synchronization | Standard | ( This document )    |
   +-------+--------------------------+----------+----------------------+
~~~

## A New Registry for EDNS Option Provider-Synchronization Operation Codes {#provsync-registry}

The Provider-Synchronization option also defines an 8-bit operation field, for
which IANA is requested to create and maintain a new registry entitled
"Provider-Synchronization Operations", used by the Provider-Synchronization
option. Initial values for the "Provider-Synchronization Operations" registry
are given below; future assignments in the 9-127 range are to be made through
Specification Required review {{?BCP26}}.

| OPERATION | Mnemonic   | Description                        | Reference         |
|-----------|------------|------------------------------------|-------------------|
| 0         | FORBIDDEN  | Reserved, MUST NOT be used         | (This document)   |
| 1         | HELLO      | Initial handshake                  | (This document)   |
| 2         | BEAT       | Heartbeat / keep-alive             | (This document)   |
| 3         | PING       | Connectivity test with nonce       | (This document)   |
| 4         | SYNC       | Agent-to-Agent zone data sync      | (This document)   |
| 5         | UPDATE     | Agent-to-Combiner zone contribution| (This document)   |
| 6         | CONFIRM    | Asynchronous confirmation          | (This document)   |
| 7         | RFI        | Request For Information            | (This document)   |
| 8         | KEYSTATE   | DNSSEC key lifecycle signaling     | (This document)   |
| 9-127     | Unassigned |                                    | (This document)   |
| 128-255   | Private Use|                                    | (This document)   |

--- back

# Change History (to be removed before publication)

* draft-leon-dnsop-signaling-zone-owner-intent-01

> Agent-to-agent data exchange now uses the CHUNK transport mechanism
> defined in {{I-D.draft-berra-dnsop-chunk-transport}} instead of the
> per-provider subdomain publication scheme ({zone}.{identity}).
> Removed the "Enabling Remote Agents to Lookup Zone Data Added By This
> Agent" section and replaced it with "Exchanging Zone Data Between
> Agents" describing the CHUNK-based approach.

> Agent discovery for DNS transport now looks up JWK records (preferred)
> with fallback to KEY records (legacy). JWK records enable both JWS
> signature verification and HPKE encryption. Updated the discovery
> examples accordingly.

> Added six new Provider-Synchronization operation codes: PING (3),
> SYNC (4), UPDATE (5), CONFIRM (6), RFI (7), and KEYSTATE (8).
> Renamed HEARTBEAT (2) to BEAT (2). Added a "Defined Operations"
> section describing each operation.

> Fixed wire format diagram for Provider-Synchronization EDNS(0) option:
> corrected byte offsets (3 octets of fixed fields, not 4).

> Updated "Agent Communication via DNS" to describe CHUNK transport with
> optional JWS/HPKE security layer.

> Replaced hardcoded section references with kramdown anchors.

> Fixed the HSYNC Sign field example from "YES" to "SIGN" for
> consistency with the field definition.

> Editorial fixes: typos (examplified, aquiring, phaste, eith,
> responsibilites), duplicate words (a a, as as, for for, the these),
> missing possessives, missing parenthesis.

* draft-leon-dnsop-signaling-zone-owner-intent-00

> This document is derived from the earlier document
> draft-leon-dnsop-distributed-multi-signer-00.
> Initial public draft.
