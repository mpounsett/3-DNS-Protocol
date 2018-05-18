% title = "Third Party DNS operator to Registrars/Registries Protocol"
% abbrev = "3-DNS-RRR" 
% category = "std"
% ipr="trust200902"
% docName = "draft-ietf-regext-dnsoperator-to-rrr-protocol-05"
% workgroup = "regext"
% area = "Applications" 
% keyword = ["dnssec", "delegation maintenance", "trust anchors"]
%
% date = 2018-05-04T00:00:00Z
%
% [[author]]
% fullname = "Jacques Latour"
% initials = "J."
% surname = "Latour"
% organization="CIRA"
%   [author.address]
%   email="jacques.latour@cira.ca"
%   [author.address.postal]
%   city="Ottawa"
%   region="ON"
%
% [[author]]
% initials = "O."
% surname = "Gudmundsson"
% fullname = "Olafur Gudmundsson"
% organization = "Cloudflare, Inc."
%  [author.address]
%  email = "olafur+ietf@cloudflare.com"
%  [author.address.postal]
%  city = "San Francisco"
%  region = "CA"
%
% [[author]]
% fullname="Paul Wouters"
% initials = "P."
% surname = "Wouters"
% organization="Red Hat"
%  [author.address]
%  email="paul@nohats.ca"
%  [author.address.postal]
%  city="Toronto"
%  region="ON"
%
% [[author]]
% fullname="Matthew Pounsett"
% initials="M."
% surname="Pounsett"
% organization="Nimbus Operations Inc."
%  [author.address]
%  email="matt@conundrum.com"
%  [author.address.postal]
%  city="Toronto"
%  region="ON"
%

.# Abstract

This document describes a simple protocol that allows a third party DNS
operator to: establish the initial chain of trust (bootstrap DNSSEC) for a
delegation; update DS records for a delegation; and, remove DS records from a
secure delegation. The DNS operator may do these things in a trusted manner,
without involving the Registrant for each operation. This same protocol can be
used by Registrants to maintain their own domains if they wish.

There are several problems that arise in the standard
Registrant/Registrar/Registry model when the operator of a zone is neither the
Registrant nor the Registrar for the delegation. Historically the issues have
been minor, and limited to difficulty guiding the Registrant through the
initial changes to the NS records for the delegation. As this is usually a one
time activity when the operator first takes charge of the zone it has not been
treated as a serious issue.

When the domain uses DNSSEC it necessary to make regular (sometimes annual)
changes to the delegation, updating DS record(s) in order to track KSK
rollover. Under the current model this is prone to delays and errors, as the
Registrant must participate in updates to DS records. Implementation of this
protocol at the Registrar and/or Registry would eliminate the need for a
Registrant who is not normally involved in the technical operation of their
zone(s) from being required to become a participant in order to secure them
with DNSSEC.

{mainmatter}

# Introduction

After a domain has been registered, the DNS operator for the child zone on the
"primary" DNS servers might be the registrant, the registrar, or a third
party.  DNS registration systems were originally designed around making
registrations easy and fast, however after registration the complexity of
making changes to the delegation differs for each of these parties.  The
Registrar can make changes directly in the Registry systems through some API
(typically EPP [@RFC5730]).  The Registrant is typically limited to using a
web interface supplied by the Registrar or Reseller, but may be provided
access to an API by the Registrar or Registry.  Typically, a third party
DNS Operator must to go through the Registrant to update any delegation
information.

Unless the responsible Registration Entity is scanning child zones for CDS
records in order to bootstrap or update DNSSEC, a third-party operator must
contact and engage the Registrant in updating DS records for the delegation.

In this case, the existing process can result various failures that impact the
Registrant's delegation.  New information must be communicated to the
Registrant, who must submit that information to the Registrar.  Often this
involves cutting and pasting between email and a web interface, which is error
prone.  For example, a mistake with copy and paste may result in a truncated
DS record being submitted to the parent zone, which can result in validation
failures.  Or, a Registrant may miss or ignore email from their DNS operator
and take no action, halting a key roll.

Furthermore, involving Registrants in this way does not scale for even
moderately sized DNS operators. Tracking thousands (or millions) of change
requests sent to customers, and following up if those changes are not
submitted to the Registrar, or are submitted with errors, is itself expensive
and error prone.  The complexity results in either the inability of a DNS
operator to implement DNSSEC or in validation failures that cause the domain
to become unavailable to users behind validating resolvers.

The goal of this document is to create a protocol for establishing a secure
chain of trust that involves parties not in the traditional
Registrant/Registrar/Registry (RRR) model, and to reduce the friction in
maintaining DNSSEC secured delegations in these cases.  It describes a
REST-based [@!RFC6690] protocol which can be used to establish DNSSEC initial
trust (to enable or bootstrap DNSSEC), and to trigger maintenance of DS
records.

Note that, while the protocol's intent is to enable a third party DNS operator
to update a Registrant's zone(s) on the Registrant's behalf, it has the
beneficial side-effect of enabling a Registrant who manages their own DNS to
automate their operations, should they wish to do so.

## Applicability

As this protocol is intended to enable third-party DNS operators to manage
their clients' zones, it may not be required in cases where this has already
been enabled by some scalable mechanism.

For example, some Registration Entities regularly scan their Registrants'
zones (both the secured and unsecured delegations) for CDS records, in order
to bootstrap or maintain the chain of trust.  In this case there is normally
no need for a DNS operator to signal the Registration Entity to scan for CDS,
since that is already being done, and so implementing this protocol would be
redundant.

[@RFC7344] section 6.1.2 discusses "other mechanisms" than regular polling
which will be used to trigger the parent (or Registration Entity) to check for
updated CDS in the child zone.  This protocol is one such mechanism.  Even a
Registration Entity that implements regular polling may wish to also implement
this signalling protocol in order to allow a DNS operator to indicate a need
to change DS records on a shorter time scale than the Registration Entity's
normal scanning period.

A key consideration for a Registration Entity in determining whether to
implement this protocol is the scalability, from the DNS operator's
perspective, of any existing notification mechanisms already implemented by
the Registration Entity.  If the Registration Entity does not implement an
update or signalling mechanism that is widely adoptable by other Registration
Entities, then implementing this protocol should be seriously considered.

For example, a proprietary API may not be considered scalable by a DNS
operator if it is only implemented by one Registration Entity, as that implies
that the DNS operator must maintain client software for the APIs of many
Registration Entiies, which is a significant development burden.  Similarly,
even an open API which requires a bilateral agreement and/or the maintenance
of unique authentication credentials may place an undue administrative burden
on third party DNS operators.

## Implementation Considerations

In some environments the role of the Registration Entity may be split between
parties.  For example, in some ccTLD environments Registrants may have a
Registrar and also deal directly with the Registry on occasion.  In such
environments it may be necessary for one Registration Entity to consider how
to notify another of changes made to a delegation as a result of implementing
this protocol.  This may be particularly important in cases where DNSSEC
information can be set by a Registrar and updated by the Registry.

As the specifics of the requirements in such an exchange of information are
dependent on the protocols in use (e.g. EPP) and the details of any agreements
between Registrar and Registry, the details are out of scope of this document.
The issue is mentioned here merely as an indication that the parties involved
may need to be aware of places that DNSSEC data might be stored outside of the
DNS, and that these data may need to be updated to prevent future conflicts or
confusion.

# Notional Conventions

## Definitions

For the purposes of this draft, a third-party DNS Operator is any DNS Operator
responsible for a zone, where the operator is neither the Registrant nor the
Registrar of record for the delegation.

Uses of "child" and "parent" refer to the relationship between DNS zone
operators (see [@RFC7719] and [@I-D.ietf-dnsop-terminology-bis]).  In
this document, unless otherwise noted, the child is the third-party DNS
operator and the parent is the Registry.

Use of the term "Registration Entity" in this document may refer to any party
that engages directly in registration activities with the Registrant.
Typically this will be a Reseller or Registrar, but in some cases, such as
when a Registry directly sells registrations to the public, may apply to the
Registry.  Even in cases where a Registrar is involved, this term may still
apply to a Registry if that Registry normally accepts DS/DNSKEY updates
directly from Registrants.

The CDS and CDNSKEY DNS resource records, having substantially the same
function but for different record types, are used interchangably in this
document.  Unless otherwise noted, any use of "CDS" or "CDNSKEY" can be
assumed to also refer to the other.

## RFC8174 Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 [@!RFC2119] [@!RFC8174]
when, and only when, they appear in all capitals, as shown here.

# Process Overview

## Identifying the Registration Entity

As of publication of this document, there has never been a standardized or
widely deployed method for easily and scalably identifying the Registration
Entity for a particular registration.

At this time, WHOIS [@RFC3912] is the only widely deployed protocol to carry
such information, but WHOIS responses are unstructured text, and each
implementor can lay out its text responses differently.  In addition,
Registries may include referrals in this unstructured text to the WHOIS
interfaces of their Registrars, and those Registrar WHOIS interface in turn
have their own layouts.  This presents a text parsing problem which is
infeasible to solve.

RDAP, the successor to WHOIS, described in [@RFC7480], solves the problems of
unstructured responses, and a consistently implemented referral system,
however at this time RDAP has yet to be deployed at most Registries. 

With no current mechanism in place to scalably discover the Registrar for a
particular registration, the problem of automatic discovery of the base URL 
of the API is considered out of scope of this document.  The authors recommend
standardization of an RDAP extension to obtain this information from the
Registry.

## Establishing a Chain of Trust

After signing the zone, the child DNS Operator needs to upload the DS
record(s) to the parent.  The child can signal its desire to have DNSSEC
validation enabled by publishing one of the special DNS records CDS and/or
CDNSKEY as defined in [@!RFC7344] and [@!RFC8078], and then by using this
protocol to trigger a CDS scan by the Registration Entity.

Once the Registration Entity finds a CDS record in an unsecured child zone it
is responsible for, or receives a signal via this API, it SHOULD start
acceptance processing as described below in (#acceptance) and (#bootstrap).

## Maintaining the Chain of Trust

Once the secure chain of trust is established, the Registration Entity SHOULD
regularly scan the child zone for CDS record changes.  If the Registration
Entity implements the protocol described in this document, then it SHOULD also
accept signals via this protocol to immediately check the child zone for CDS
records.

Server implementations of this protocol MAY include rate limiting to protect
their systems and the systems of child operators from abuse.

Each parent operator and Registration Entity is responsible for developing,
implementing, and communicating their DNSSEC maintenance policies.

## Acceptance Processing {#acceptance}

The Registration Entity, upon receiving a signal or detecting through polling
that the child desires to have its delegation updated, SHOULD run a series of
tests to ensure that updating the parent zone will not create or exacerbate
any problems with the child zone. The Registration Entity MUST implement the
processing rules defined in [@!RFC7344] section 4.1.  The Registration Entity
SHOULD also implement tests that include:

  - checks that the child zone is is properly signed as per the Registration
    Entity and parent DNSSEC policies
  - ensuring all name servers in the apex NS RRset of the child zone agree on
    the apex NS RRset and CDS RRset contents

The Registration Entity MAY require the child zone to implement zone
delegation best practices as described in
[@I-D.wallstrom-dnsop-dns-delegation-requirements].

The Registration Entity SHOULD NOT make any changes to the DS RRset if the
child name servers do not agree on the CDS content.

## Bootstrapping DNSSEC {#bootstrap}

A Registration Entity implementing this protocol SHOULD require compliance
with additional tests in the case of establishing a new chain of trust.

 - The Registration Entity SHOULD check that all child name servers respond
   with a consistent CDS RRset for a number of queries over an extended period
   of time ([@RFC8078] section 3.3).  Any change in DS response or
   inconsistency between child responses in that time might indicate an
   attempted Man in the Middle (MITM) attack, and SHOULD reset the test.  This
   minimizes the possibility of an attacker spoofing responses.  An example of
   such a policy might be to scan all child name servers in the delegation NS
   RRset every two hours for a week.

 - The Registration Entity SHOULD require all of the child name servers in the
   delegation NS RRset to send the same response to a CDS query whether sent
   over TCP or UDP.

 - The Registration Entity MAY require the child operator to prove they can
   add data to the zone, for example by publishing a particular token
   ([@RFC8078] section 3.4).  See (#token) below.

# Registration Entity Requirements

## Probing the DNS

When the Registration Entity is probing for child zone contents the
Registration Entity SHOULD query the child zone's authoritative servers
directly, and MUST NOT query a name server which is actively caching
responses.  A caching name server could have out of date child zone data
which could cause confusion or indeterminite behaviour as the child operator
expects the Registration Entity to see a set of data which they do not.

## Published Policy and Procedures {#policy}

A Registration Entity implementing this protocol SHOULD publish a policy and
procedures document which details steps taken, timings used, requirements
implemented, etc. as they apply to this protocol and to the Registration
Entity's implementation of CDS scanning.

# API Definition

## Authorization

Clients may be denied access to change the DS records for domains that are
subject to certain registration-related locks (HTTP Status code 401).  For
example, the Registry Lock is a mechanism provided by certain Registries or
Registrars that prevents domain hijacking by ensuring no attributes of the
domain are changeable, and no transfer or deletion transactions can be
processed against the domain name without manual intervention.

## Authentication

The API does not impose any unique server authentication requirements.  There
is a potential risk for a Man in the Middle style Denial of Service attack
against a client, where an attacker could impersonate the server and block a
client's signal from reaching the intended recipient.  Therefore, the API MUST
be provided over some transport which allows the client to reliably identify
whether they have connected to the correct server, such TLS-protected
transport (i.e. HTTPS) or VPN.

Client authentication is considered out of scope of this document.  The
publication of CDS records in the child zone is an indication that the
child operator intends to perform DS-record-updating activities (add/delete)
in the parent zone.  Since this protocol is simply a signal to the
Registration Entity that they should examine the child zone for such
intentions, additional authentication of the client making the request is
considered unnecessary.

A Registration Entity MAY implement their own policy to protect access to the
API, such as with IP white listing, client TLS certificates, etc..  The
Registration Entity SHOULD take steps to ensure that their implementation of
this protocol does not open up an avenue for a denial of service attack
against the systems of the Registration Entity, the Registry, or the child
operator.

## RESTful Resources

In the following text, "{domain}" is the child zone to be operated on.

### Policy Resource

Path: /policy

#### Obtain Policy URL

**Request**

Syntax: GET /policy

Request the URL for the Registration Entity's procedures and policy document
as described in (#policy).

**Response**

   - HTTP Status code 201 indicates a success.
   - HTTP Status code 404 indicates the resource does not exist.
   - HTTP Status code 429 indicates the client has been rate-limited.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.

The body of the response SHOULD contain the URL of the Registration Entity's
procedures and policy document.

### CDS resource

Path: /domains/{domain}/cds

#### Establishing Initial Trust (Enabling DNSSEC)

**Request**

Syntax: POST /domains/{domain}/cds

Request that an initial set of DS records based on the CDS record in the child
zone be inserted into the Registry and the parent zone upon the successful
completion of the request. If there are multiple CDS records in the CDS RRset,
multiple DS records will be added.

The body of the POST SHOULD be empty, however server implementations SHOULD
NOT reject nonempty requests.

**Response**

   - HTTP Status code 201 indicates a success.
   - HTTP Status code 400 indicates a failure due to validation.
   - HTTP Status code 401 indicates an unauthorized resource access.
   - HTTP Status code 403 indicates a failure due to an invalid challenge
     token.
   - HTTP Status code 404 indicates the domain does not exist.
   - HTTP Status code 412 indicates the delegation already has a DS RRset.
   - HTTP Status code 429 indicates the client has been rate-limited.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.

This request is for setting up initial trust in the delegation.  The
Registration Entity SHOULD return a status code 412 if it already has a DS
RRset for the child zone.

Upon receipt of a 403 response the child operator SHOULD issue a POST for the
"token" resource to fetch a challenge token to insert into the zone.

#### Removing DS Records

**Request**

Syntax: DELETE /domains/{domain}/cds

Request that the Registration Entity check for a null CDS or CDNSKEY record in
the child zone, indicating a request that the entire DS RRset be removed.
This will make the delegation insecure.

**Response**

   - HTTP Status code 200 indicates a success.
   - HTTP Status code 400 indicates a failure due to validation.
   - HTTP Status code 401 indicates an unauthorized resource access.
   - HTTP Status code 403 indicates a failure due to an invalid challenge
   - HTTP Status code 404 indicates the domain does not exist.
   - HTTP Status code 412 indicates the parent does not have a DS RRset
   - HTTP Status code 429 indicates the client has been rate-limited.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.

Upon receipt of a 403 response the child operator SHOULD issue a POST for the
"token" resource to fetch a challenge token to insert into the zone.

#### Modifying DS Records

**Request**

Syntax: PUT /domains/{domain}/cds

Request that the Registration Entity modify the DS RRset based on the CDS
available in the child zone.  As a result of this request the Registration
Entity SHOULD add or delete DS or DNSKEY records as indicated by the
CDS RRset, but MUST NOT delete the entire DS RRset.

**Response**

   - HTTP Status code 200 indicates a success.
   - HTTP Status code 400 indicates a failure due to validation.
   - HTTP Status code 401 indicates an unauthorized resource access.
   - HTTP Status code 403 indicates a failure due to an invalid challenge
   - HTTP Status code 404 indicates the domain does not exist.
   - HTTP Status code 412 indicates the parent does not have a DS RRset
   - HTTP Status code 429 indicates the client has been rate-limited.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.

Upon receipt of a 403 response the child operator SHOULD issue a POST for the
"token" resource to fetch a challenge token to insert into the zone.

### Token resource {#token}

Path: /domains/{domain}/token

#### Establish Initial Trust with Challenge

**Request**

Syntax: GET /domains/{domain}/token

The DNSSEC policy of the Registration Entity may require proof that the DNS
Operator is in control of the domain.  The token API call returns a random
token to be included as a TXT record for the _delegate.@ domain name (where @
is the apex of the child zone) prior establishing the DNSSEC initial trust.
This is an additional trust control mechanism to establish the initial chain
of trust.

Once the child operator has received a token, it SHOULD be inserted in the
zone and the operator SHOULD proceed with a POST of the cds resource.

The Registration Entity MAY expire the token after a reasonable period.  The
Registration Entity SHOULD document an explanation of whether and when tokens
are expired in their DNSSEC policy.

Note that the _delegate TXT record is publicly visible and therefore cannot be
treated as a secret token.

**Response**

   - HTTP Status code 200 indicates a success.  A token is included in the
	 body of the response, as a valid TXT record
   - HTTP Status code 404 indicates the domain does not exist.
   - HTTP Status code 429 indicates the client has been rate-limited.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.

## Customized Error Messages

The Registration Entity MAY provide a customized error message in the response
body in addition to the HTTP status code defined in the previous section.
This response MAY include an identifying number/string that can be used to
track the request.

# Security Considerations

Using CDS to bootstrap trust poses inherent risks.  The issues surrounding
this practice are well described in the Security Considerations section of
[@!RFC8078].  Since this protocol simply adds a layer of signalling on top of
the processes described in that document, there are no new Security
Considerations posed by using this protocol in combination with CDS.

# IANA Actions

This document has no actions for IANA

# Internationalization Considerations

This protocol is designed for machine to machine communications.  Clients and
servers SHOULD use punycode [@!RFC5891] when operating on internationalized
domain names.

{backmatter}

# Document History

## regext Version 06 (not yet published)

  - more grammar and spelling nits
  - clarify language indicating that a registrant may be a user of this
    protocol
  - try to make it clearer that the DNS operator role may be one of several
    different parties, without blurring the line between role and party
  - restructure the abstract to lead with the goals
  - better address potential issues with the status quo in the Introduction
  - removing plural Registration Entity references
  - clarify the intent of the authentication requirements
  - clarify the text about not having secrets in the _delegate TXT record.
  - since all security considerations were related to trust bootstrapping,
    offload security considerations to RFC8078
  - fix punycode reference to point to the correct (current) RFC
  - added Implementation Considerations subsection
  - change HTTP response 409 to 412 for consistency
  - remove synchronous vs. asynchronous text
  - rephrase registry lock text to make it an example rather than an
    exhaustive list of why a 401 may be returned
  - for above, add Authorization subsection
  - make HTTP 4xx response code use more consistent
  - moved non sequitur text from Establishing a Chain of Trust to a new
    Applicability subsection
  - cleaned up external references
  - added requirement for RE policy and procedure document
  - reduce the deep subheading mess
  - updated RFC2119 text to RFC8174

## regext Version 05

  - new version to keep the draft alive
  - updating author organization

## regext Version 04
  - changed uses of Registrar to Registration Entity and updated definitions
	to improve clarity
  - adding note about CDS/CDNSKEY interchangability in this document
  - added advice to scan all delegations (including insecure delegations) for
    CDS in order to bootstrap or update DNSSEC
  - removed "Other Delegation Maintenance" section, since we decided a while
    ago not to use this to update NS

## regext Version 03
  - simplify abstract
  - move all justification text to Intro
  - added HTTP response codes for rate limiting (429), missing DS RRsets
	(412)
  - expanded on Internationalization Considerations
  - corrected informative/normative document references
  - clarify parent/Registrar references in the draft
  - general spelling/grammar/style cleanup
  - removed references to NS and glue maintenance
  - clarify content of POST body for 'cds' resource
  - change verb for obtaining a 'token' to GET
  - Updated reference to RFC8078

## regext Version 02 
  - Clarified based on comments and questions from early implementors (JL)
  - Text edits and clarifications. 

## regext Version 01 
  - Rewrote Abstract and Into (MP) 
  - Introduced code 401 when changes are not allowed 
  - Text edits and clarifications. 

## regext Version 00 
  - Working group document same as 03, just track changed to standard

## Version 03
  - Clarified based on comments and questions from early implementors

## Version 02
  - Reflected comments on mailing lists

## Version 01
  - This version adds a full REST definition this is based on suggestions from
Jakob Schlyter.

## Version 00
  - First rough version
