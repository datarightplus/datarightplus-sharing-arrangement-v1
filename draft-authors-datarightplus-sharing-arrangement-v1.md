%%%
title = "DataRight+: Sharing Arrangement V1"
area = "Internet"
workgroup = "datarightplus"
submissionType = "independent"

[seriesInfo]
name = "Internet-Draft"
value = "draft-authors-datarightplus-sharing-arrangement-v1-latest"
stream = "independent"
status = "experimental"

date = 2024-04-02T00:00:00Z

[[author]]
initials="S."
surname="Low"
fullname="Stuart Low"
organization="Biza.io"
  [author.address]
  email = "stuart@biza.io"

[[author]]
initials="B."
surname="Kolera"
fullname="Ben Kolera"
organization="Biza.io"
  [author.address]
  email = "bkolera@biza.io"
%%%

.# Abstract

This specification outlines the technical requirements related to the delivery of Sharing Arrangement V1.

.# Notational Conventions

The keywords  "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",  "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in [@!RFC2119].

{mainmatter}

# Scope

The scope of this specification is focused on describing the authorisation related components of establishing a CDR Sharing Arrangement. This specification unavoidably references ecosystem specific concepts (Australian CDR) due to the very nature of localised claim names. Sharing Arrangement V1 therefore is aligned with the definition of a CDR Arrangement as originally outlined within [@!CDS].

# Terms & Definitions

This specification uses the terms "Consumer", "Provider", "Initiator", "Personally Identifiable Information (PII)",
"Pairwise Pseudonymous Identifier (PPID)", "Initiator", "CDR Sharing Arrangement" and "CDR Sharing Arrangement Identifier"  as defined by [@!DATARIGHTPLUS-ROSETTA].

# Provider

The following provisions apply to services delivered by Providers.

## Authorisation Server

In addition to the provisions outlined in [@!DATARIGHTPLUS-INFOSEC-BASELINE] the authorisation server:

1. **SHALL NOT** issue a refresh token unless the duration of the CDR Arrangement is greater than `0` (see [Request Object])
4. **SHALL**, where the supplied `cdr_arrangement_id`, if provided, does not match the authenticated Consumer and as soon as practicable, return an OAuth2 Error
5. **SHALL** modify the existing authorisation grant referenced by the `cdr_arrangement_id`, if provided, while maintaining the `cdr_arrangement_id` between such authorisations
8. **SHALL** revoke previously issued Access and refresh tokens when modifying existing authorisation grants;
2. **SHALL** support the Provider CDR Arrangement Revocation Endpoint (PCARE);
9. **SHALL** publish the url for the [Provider CDR Arrangement Revocation Endpoint (PCARE)] as an attribute named `cdr_arrangement_revocation_endpoint` within the OpenID Connect Discovery 1.0 [@!OIDC-Discovery] endpoint response;

### Request Object

The request object submitted to the authorisation server:

1. **SHALL** support an optional integer parameter of `sharing_duration` representing the number of seconds requested for data sharing. The default value of this parameter **SHALL** be `0` and the maximum value **SHALL NOT** exceed `31536000`.
3. **SHALL** support an optional string parameter `cdr_arrangement_id` referencing an existing CDR Arrangement (see CDR Sharing Arrangement Identifier);
4. **SHALL** reject requests containing a `cdr_arrangement_id` parameter that is unknown, expired or not associated with the requesting Initiator

### Claims

The authorisation server:

1. **SHALL** include within the Introspection Endpoint response:
   1. the `exp` claim, with a value equal to the expiry time of the underlying CDR Arrangement and;
   2. the `cdr_arrangement_id` claim, containing the assigned [CDR Sharing Arrangement Identifier]
2. **SHALL** ensure the `cdr_arrangement_id` claim is persistent and does not change between token interactions;

## Provider CDR Arrangement Revocation Endpoint (PCARE)

The Provider CDR Arrangement (PCARE) Revocation endpoint accepts a CDR Sharing Arrangement Identifier and immediately revokes the CDR Sharing Arrangement.

### PCARE Discovery

The PCARE endpoint **SHALL** be advertised using the `cdr_arrangement_revocation_endpoint` attribute at the OpenID Connect Discovery 1.0 [@!OIDC-Discovery] endpoint.

### PCARE Request

The PCARE endpoint receives requests using the HTTP POST [@!RFC7231] method with parameters sent as `application/x-www-form-urlencoded` data as defined in [@!W3C.REC-html5-20141028].

In addition to authentication credentials described in Section 2.2 of [@!RFC7523] the endpoint includes the CDR Sharing Arrangement Identifier as an additional parameter, specified as follows:

- `cdr_arrangement_id`: **REQUIRED**  The CDR Sharing Arrangement Identifier previously provided in the token and introspection endpoint responses.

An example request to the Provider CDR Arrangement Revocation Endpoint is as follows:

```
POST https://data.provider.com.au/arrangements/revoke
HTTP/1.1
Host: data.Provider.com.au
Content-Type: application/x-www-form-urlencoded

client_id=s6BhdRkqt3&
client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&
client_assertion=eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjEyNDU2In0.ey ...&
cdr_arrangement_id=5a1bf696-ee03-408b-b315-97955415d1f0
```

On receipt of a valid request the Provider authorisation server:

1. **SHALL** validate the client credentials as per Section 2.2 of [@!RFC7523] and;
2. **SHALL** validate the `cdr_arrangement_id` relates to a valid CDR Sharing Arrangement and;
3. **SHALL** verify the CDR Sharing Arrangement referenced by specified `cdr_arrangement_id` belongs to validated client
4. If all of the above conditions are met the authorisation server:
   - **SHALL** immediately invalidate the CDR Arrangement
   - **SHALL** immediately invalidate all refresh tokens associated with the CDR Arrangement
   - **SHALL** immediately invalid all access tokens associated with the CDR Arrangement

### PCARE Response

#### Successful Response

In the event of a successful revocation, the authorisation server:

1. If the CDR Sharing Arrangement is revoked, **SHALL** respond with a 204 HTTP status code containing no content
2. If the CDR Sharing Arrangement was already revoked prior to the request being received, **SHOULD** respond with a 204 HTTP status code containing no content

#### Failure Response

In the event of a failure to validate the CDR Arrangement conditions the authorisation server:

1. **SHALL** respond with a  422 Unprocessable Entity response containing a JSON payload structured as as described below and;
2. **SHOULD** respond with a 503 Service Unavailable if currently unavailable and **MAY** supply a `Retry-After` indicating the earliest retry time

##### Failure Response Body

The failure response body is an object with a single attribute of `errors` containing an array of one object with the following fields:

- `code` with the value of `urn:au-cds:error:cds-all:Authorisation/InvalidArrangement`
- `title` with the value of `The arrangement could not be found.`
- `detail` with a value containing the provided `cdr_arrangement_id`

A non-normative example of the payload is provided below:

```
{
    "errors": [
        {
            "code": "urn:au-cds:error:cds-all:Authorisation/InvalidArrangement",
            "title": "The arrangement could not be found.",
            "detail": "b3f0c9d0-457d-4578-b0cd-52e443ae13c5"
        }
    ]
}
```

# Initiator

The following provisions apply to participants operating individual Initiators.

## Authorisation Client

An Initiators authorisation client:

1. **SHALL** support the provisions specified in Section 4 of [@!DATARIGHTPLUS-INFOSEC-BASELINE]
2. **SHALL** support the submission of Request Object parameters outlined in [Request Object]
2. **SHALL** call the [Provider CDR Arrangement Revocation Endpoint (PCARE)] following;
   1. revocation request is requested by the Consumer using the Initiator Consumer Dashboard or;
   2. where a refresh token has not been issued, as soon as practicable following the completion of data collection
4. **SHALL** support the [Initiator CDR Arrangement Revocation Endpoint (ICARE)]
5. **SHALL** retry requests to the [Provider CDR Arrangement Revocation Endpoint (PCARE)] where the response received was a HTTP Status 503. The retry time **SHOULD** be informed by the presence of the `Retry-After` header.

## Initiator CDR Arrangement Revocation Endpoint (ICARE)

The Initiator CDR Arrangement (ICARE) Revocation endpoint accepts a CDR Sharing Arrangement Identifier and immediately revokes the CDR Sharing Arrangement.

### ICARE Request

The protected resource calls the ICARE endpoint using an HTTP POST [@!RFC7231] request with parameters sent as `application/x-www-form-urlencoded` data as defined in [@!W3C.REC-html5-20141028].

`cdr_arrangement_jwt`
**REQUIRED**. A single signed [@!JWT] containing the following parameters:

1. `cdr_arrangement_id` representing the CDR Sharing Arrangement Identifier previously provided in the introspection endpoint responses;
2. `iss`: Provider ID issued by the Ecosystem Authority
3. `sub`: Provider ID issued by the Ecosystem Authority
4. `aud`: URI to the ICARE endpoint
5. `jti`: A unique identifier for the token
6. `exp`: Expiration time on or after the JWT must not be accepted

In addition, the client includes an `Authorization` header parameter with a Bearer token containing a single signed [@!JWT] with the following parameters:

1. `iss`: Provider ID issued by the Ecosystem Authority
2. `sub`: Provider ID issued by the Ecosystem Authority
3. `aud`: URI to the ICARE endpoint
4. `jti`: A unique identifier for the token
5. `exp`: Expiration time on or after the JWT must not be accepted

For example, a Provider may request CDR Arrangement revocation with the following request:

```
POST https://data.Initiator.com.au/arrangements/revoke HTTP/1.1
Host: data.Initiator.com.au
Content-Type: application/x-www-form-urlencoded
Authorization: Bearer eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjEyNDU2In0.ey …

cdr_arrangement_jwt=eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjEyNDU2In0.ey ...&
```

The Initiator first validates the Provider credentials and then verifies whether the CDR Sharing Arrangement Identifier exists and was issued by the Provider making the CDR Arrangement Revocation request. If this validation fails, the request is refused and the Provider is informed of the error by the ICARE endpoint as described below.

In the next step, the Initiator invalidates the CDR Arrangement and **SHOULD** discard associated tokens. Data Minimisation events **SHOULD** also be triggered on receipt of this event.

### ICARE Response

#### Successful Response

In the event of a successful revocation, the Initiator:

1. if the CDR Sharing Arrangement is revoked, **SHALL** respond with a 204 HTTP status code containing no content
2. if the CDR Sharing Arrangement was already revoked prior to the request being received, **SHOULD** respond with a 204 HTTP status code containing no content

#### Failure Response

In the event of a failure to validate the CDR Arrangement conditions the Initiator:

1. **SHALL** respond with a  422 Unprocessable Entity response containing a JSON payload structured as as described below and;
2. **SHOULD** respond with a 503 Service Unavailable if currently unavailable and **MAY** supply a `Retry-After` indicating the earliest retry time

##### Failure Response Body

The failure response body is an object with a single attribute of `errors` containing an array of one object with the following fields:

- `code` with the value of `urn:au-cds:error:cds-all:Authorisation/InvalidArrangement`
- `title` with the value of `The arrangement could not be found.`
- `detail` with a value containing the provided `cdr_arrangement_id`

A non-normative example of the payload is provided below:

```
{
    "errors": [
        {
            "code": "urn:au-cds:error:cds-all:Authorisation/InvalidArrangement",
            "title": "The arrangement could not be found.",
            "detail": "b3f0c9d0-457d-4578-b0cd-52e443ae13c5"
        }
    ]
}
```

# Implementation Considerations

Where a Initiator, by way of the [Request Object], requests to update an existing authorisation grant through the supply of the `cdr_arrangement_id` some ecosystems may apply additional expectations with respect to the authorisation flow. Within the Australian Consumer Data Right this is summarised within the _CX Standards: Amending Authorisation_ section of the [@!CDS].

# Security Considerations

The CDR Sharing Arrangement Identifier **SHALL NOT** be guessable, derivable nor identify the Consumer.

{backmatter}

<reference anchor="CDS" target="https://consumerdatastandardsaustralia.github.io/standards"><front><title>Consumer Data Standards (CDS)</title><author><organization>Data Standards Body (Treasury)</organization></author></front> </reference>

<reference anchor="DATARIGHTPLUS-ROSETTA" target="https://datarightplus.github.io/datarightplus-rosetta/draft-authors-datarightplus-rosetta.html"> <front><title>DataRight+ Rosetta Stone</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="DATARIGHTPLUS-INFOSEC-BASELINE" target="https://datarightplus.github.io/datarightplus-infosec-baseline/draft-authors-datarightplus-infosec-baseline.html"> <front><title>DataRight+ Security Profile: Baseline</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="OIDC-Discovery" target="https://openid.net/specs/openid-connect-discovery-1_0.html"> <front> <title>OpenID Connect Discovery 1.0 incorporating errata set 1</title> <author initials="N." surname="Sakimura" fullname="Nat Sakimura"> <organization>NRI</organization> </author> <author initials="J." surname="Bradley" fullname="John Bradley"> <organization>Ping Identity</organization> </author> <author initials="M." surname="Jones" fullname="Mike Jones"> <organization>Microsoft</organization> </author> <author initials="E." surname="Jay"> <organization>Illumila</organization> </author><date day="8" month="Nov" year="2014"/> </front> </reference>

<reference anchor="JWT" target="https://datatracker.ietf.org/doc/html/rfc7519"> <front> <title>JSON Web Token (JWT)</title> <author fullname="M. Jones"> <organization>Microsoft</organization> </author> <author initials="J." surname="Bradley" fullname="John Bradley"> <organization>Ping Identity</organization> </author><author fullname="N. Sakimura"> <organization>Nomura Research Institute</organization> </author> <date month="May" year="2015"/></front> </reference>


