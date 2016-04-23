% title = "Third Party DNS operator to Registrars/Registries Protocol"
% abbrev = "3-DNS-RRR"
% category = "info"
% ipr="trust200902"
% docName = "draft-latour-dnsoperator-to-rrr-protocol-03.txt"
% area = "Applications"
% workgroup = ""
% keyword = ["dnssec", "delegation maintenance", "trust anchors"]
%
% date = 2016-03-21T00:00:00Z
%
% [[author]]
% fullname = "Jacques Latour"
% initials = "J."
% surname = "Latour"
% organization="CIRA"
%   [author.address]
%   street="Ottawa, ON"
%   email="jacques.latour@cira.ca"
%
% [[author]]
% initials = "O."
% surname = "Gudmundsson"
% fullname = "Olafur Gudmundsson"
% organization = "Cloudflare, Inc."
%  [author.address]
%  email = "olafur+ietf@cloudflare.com"
%  street = "San Francisco, CA"
%
% [[author]]
% fullname="Paul Wouters"
% initials = "P."
% surname = "Wouters"
% organization="Red Hat"
%  [author.address]
%  street="Toronto, ON"
%  email="paul@nohats.ca"
%
% [[author]]
% fullname="Matthew Pounsett"
% initials="M."
% surname="Pounsett"
% organization="Rightside Group, Ltd."
%  [author.address]
%  street="Toronto, ON"
%  email="matt@conundrum.com"
%

.# Abstract
There are problems that arise in the standard Registrant/Registrar/Registry
model when the operator of a zone is neither the Registrant nor the Registrar
for a delegation.  Originally this was limited to NS record changes which were
most often done once at delegation time, or rarely when the operator
reorganized their infrastructure, or the registrant moved operators.  With
DNSSEC it becomes necessary for delegation information to be changed
frequently, which introduces errors or critical delays as information must be
channeled through a Registrant who is likely not technically knowledgeable. 

This draft describes a REST-based protocol that will allow authorized
third-party DNS operators to make necessary operational changes to zones
delegated to them without needing to involve the Registrant.


{mainmatter}

# Introduction

DNS registration systems are designed around making registrations easy and
fast. When it comes to setting up the master (or primary) DNS service, the
level of ease varies greatly depending on whether the DNS operator is the
Registrar, Registrant, or a third party.

When the Registrar is the DNS operator, it is able to directly and
automatically make changes at the Registry as necessary. If the Registrant
is the DNS operator, they can make whatever changes they need using whatever
interface the Registrar provides. A third party DNS operator, on the other
hand, must go through the Registrant (who may or may not be technically
capable) in order to have changes submitted through the Registrar to the
Registry.

There are many examples of common failure modes with a third-party DNS
operator:
- submission of incorrect data to the Registrar due to copy and paste errors,
  or unintentionally omitting important data
- Registrants failing to submit data to the Registrar in a timely manner
- failure to remove DS records when moving from a DNS operator that supports
  DNSSEC to one that does not

These sorts of human errors can result in partial or complete failure of a
zone for anyone using a DNSSEC validating resolver. The protocol described by
this draft is intended to simplify the process of updating delegation
information, for both the Registrant and third party DNS operators, by
enabling automation and eliminating obvious and common sources of human
error.

In addition to allowing a third party operator to properly maintain DNSSEC
data, the protocol mitigates a much older, but less serious problem of being
able to update delegation NS and glue records as necessary without needing to
get the attention of the Registrant.

# Notational Conventions

## RFC2119 Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [@RFC2119].

## Definitions

For the purposes of this document, a third-party DNS Operator is any DNS
Operator responsible for a zone where the operator is neither the Registrant
nor the Registrar of record for the delegation.

Uses of the word 'Registrar' in this document may also be applied to
Resellers: an entity that sells delegations through a Registrar with whom the
entity has a Reseller agreement.

## Other Document Conventions

Whenever this document refers to a parent organization making changes to its
zone, the organization may be the Registry directly modifying its zone, or a
Registrar or Reseller directing the Registry to make changes whether through
EPP or some other registration protocol.


# Protocol Overview

The primary goal of the described protocol is to securely pass updated
delegation information from a child zone operator to its parent.

The first step in this process for a DNS operator is parent discovery.
Although the parent name servers may be easily found, the correct organization
to pass updates to may not be obvious, and the URI for an API might still be
more difficult to find. In some cases the Registry may directly accept
updates of delegation information, or the correct recipient of updates may be
one of hundreds of possible Registrars. WHOIS [@RFC3912] might seem like the
natural mechanism to discover the entry point for the API at the parent
organization, but parent discovery must be automated, and the lack of a formal
standard for the data structure of a WHOIS response makes that infeasible.
RDAP [@RFC7480], the successor to WHOIS, provides this standardized response
however it may be some time before RDAP achieves wide deployment. A simple
discovery API that can be easily deployed while Registries and Registrars deal
with the larger problem of rolling out RDAP may be advisable.

[Editors Note (to be removed before publication): Do we want to define a
discovery API here?]

Once the zone operator has identified the URI for the API at the parent, this
REST-based [@RFC6690] protocol may be used to submit requests to the parent to
make changes to the child records (NS, DS, etc.) contained in the parent zone.
The parent will use the contents of the child zone to verify requests made via
this API, and to discover information required to complete these requests.
Validation of requests made through this protocol require that both the child
and parent zones MUST be DNSSEC signed.

## Requiring DNSSEC

Since the parent and child organizations do not necessarily have a secure
channel arranged prior to the first use of this API, it is necessary that the
parent organization be able to validate requests being made. DNSSEC
[@!RFC4035] provides data validation for DNS answers; having both zones DNSSEC
signed makes it possible for the parent to validate and trust the records in
the child zone.

The most difficult aspect of this is the initial setting of the DS record in
the parent zone, since prior to this the parent organization has no way of
validating the CDS records in the child zone.

## Signalling the Parent to Add DS

Before contacting the parent organization, the child organization must first
sign the zone. Once signed, the child organization can signal its desire to
have DNSSEC validation enabled by publishing one of the special DNS records
CDS and/or CDNSKEY [@!RFC7344]. The parent and child SHOULD support and use
the CDS/CDNSKEY extensions proposed in [@!I-D.ietf-dnsop-maintain-ds#01]

Once these records are published, the child organization can use this API to
indicate to the parent that CDS processing should begin.

## Validating DS Add Requests

The parent, upon receiving a signal via this API to check the child zone for
CDS publication, should proceed to validate the request with a series of
tests:
1. the child zone MUST be signed
2. the child zone MUST have one or more CDS or CDNSKEY records
3. all of the name-servers in the NS RRSET for the child zone which are in the
   parent zone must agree on the CDS/CDNSKEY RRSET
4. the CDS/CDNSKEY records MUST have an RRSIG which can be validated from the
   DS record in the parent zone if it exists, or from a CDS/CDNSKEY record if
   it does not

If the parent and child organization do not have a previously arranged secure
communication channel, the parent SHOULD have additional tests or conditions
which must pass before adding DS records to the parent zone. Parents MAY
require additional tests or conditions in any case. Some example additional
tests and conditions are:
- monitoring over an extended period before trusting new CDS records
- additional queries over TCP
- tokens added to the child zone to prove control

The API is partially synchronous. The server can elect to allow a connection
to remain open until the operation has completed, or it can immediately return
acknowledgment of the request. In the latter case, it is up to the child
operator to monitor the parent zone for completion of the operation, and issue
possible followup requests as necessary.


# OP-3-DNS-RR RESTful API

The specification of this API is minimalist, but a realistic one. Question:
How to respond if the party contacted is not ALLOWED to make the requested
change?

## Authentication
The API does not impose any unique server authentication requirements. The
server authentication provided by TLS fully addresses the needs. In general,
for the API SHOULD be provided over TLS-protected transport (e.g., HTTPS) or
VPN.

## Authorization
Authorization is out of scope of this document. The CDS records present in the
zone file are indications of intention to sign/unsign/update the DS records of
the domain in the parent zone. This means the proceeding of the action is not
determined by who issued the request. Therefore, authorization is out of the
scope. Registries and Registrars who plan to provide this service can,
however, implement their own policy such as IP white listing, API key, etc.

## Base URL Locator

The base URL for Registries or Registrars who want to provide this service to
DNS Operators can be made auto-discoverable as an RDAP extension.

## CDS resource
Path: /domains/{domain}/cds
{domain}: is the domain name to be operated on

### Initial Trust Establishment (Enable DNSSEC validation)
#### Request
Syntax: POST /domains/{domain}/cds

A DS record based on the CDS record in the child zone file will be inserted
into the Registry and the parent zone file upon the successful completion of
such request. If there are multiple CDS records in the CDS RRSET, multiple DS
records will be added.

Either the CDS/CDNSKEY or the DNSKEY can be used to create the DS record.
Note: entity expecting CDNSKEY is still expected accept the /cds command.

#### Response
   - HTTP Status code 201 indicates a success.
   - HTTP Status code 400 indicates a failure due to validation.
   - HTTP Status code 403 indicates a failure due to an invalid challenge token.
   - HTTP Status code 404 indicates the domain does not exist.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.


### Removing a DS (turn off DNSSEC)
#### Request
    Syntax: DELETE /domains/{domain}/cds

#### Response
   - HTTP Status code 200 indicates a success.
   - HTTP Status code 400 indicates a failure due to validation.
   - HTTP Status code 404 indicates the domain does not exist.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.

### DS Maintenance (Key roll over)
#### Request
    Syntax: PUT /domains/{domain}/cds

#### Response
   - HTTP Status code 200 indicates a success.
   - HTTP Status code 400 indicates a failure due to validation.
   - HTTP Status code 404 indicates the domain does not exist.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.

## Tokens resource
   Path: /domains/{domain}/tokens
   {domain}: is the domain name to be operated on

### Setup Initial Trust Establishment with Challenge
#### Request
    Syntax: POST /domains/{domain}/tokens

A random token to be included as a _delegate TXT record prior establishing the
DNSSEC initial trust.

#### Response
   - HTTP Status code 200 indicates a success. Token included in the body of
     the response, as a valid TXT record
   - HTTP Status code 404 indicates the domain does not exist.
   - HTTP Status code 500 indicates a failure due to unforeseeable reasons.


## Customized Error Messages
Service providers can provide a customized error message in the response body
in addition to the HTTP status code defined in the previous section.

This can include an identifying number/string that can be used to track the
requests.

#Using the definitions
This section at the moment contains comments from early implementers

## How to react to 403 on POST cds
The basic reaction to a 403 on POST /domains/{domain}/cds is to issue POST /domains/{domain}/tokens
command to fetch the challenge to insert into the zone.

# Security considerations

TBD This will hopefully get more zones to become validated thus overall the
security gain out weights the possible drawbacks.

risk of takeover ?
risk of validation errors < declines
transfer issues

# IANA Actions
URI ??? TBD


# Internationalization Considerations
This protocol is designed for machine to machine communications

{backmatter}

# Document History

## Version 03
Clarified based on comments and questions from early implementors

## Version 02
Reflected comments on mailing lists

## Version 01
This version adds a full REST definition this is based on suggestions from
Jakob Schlyter.


## Version 00
First rough version


