%%%
Title = "CDR+ Action: Sharing Arrangement V1"
area = "Internet"
workgroup = "cdrplus-parity"

[seriesInfo]
name = "Internet-Draft"
value = "draft-authors-cdrplus-sharing-arrangement-v1-latest"
stream = "IETF"
status = "experimental"

date = 2022-06-27T00:00:00Z

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

Describes the CDR Sharing Arrangement V1

.# Notational Conventions

The keywords "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",  "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in [@!RFC2119].

{mainmatter}

# Scope


# Terms & Definitions

This specification uses the terms "Consumer", "Data Holder", "Data Recipient", "Personally Identifiable Information (PII)",
"Pairwise Pseudonymous Identifier (PPID)", "Software Product"  as defined by [@!CDRPLUS-INFOSEC-BASELINE].

CDR Sharing Arrangement
CDR Sharing Arrangement Identifier
Once Off Authorisation

# Data Holder

The following provisions apply to participants considered a Data Holder.

## Authorisation Server

The authorisation server:

1. **SHALL** issue access tokens in alignment with [@!CDRPLUS-INFOSEC-BASELINE] and on successful completion of an authorisation flow
2. **SHALL** issue refresh tokens with an expiry matched to the CDR Arrangement;
2. **SHOULD NOT** issue refresh tokens which which a `sharing_duration` was not provided or was provided with a value of `0` (see Request Object)
3. **MUST** support the Holder CDR Arrangement Revocation Endpoint (HCARE)
5. **MUST** include the CDR Arrangement ID as an attribute in its Introspection Endpoint response
6. **MUST** support the [@!OIDC-Core] `acr` claim

### Request Object

The request object submitted to the authorisation server:

1. **MUST** support a claim parameter of `sharing_duration` representing the number of seconds, as an integer, to request a CDR Sharing Arrangement;
2. **MUST** consider `sharing_duration` lengths exceeding `31536000` to be equal to `31536000`;
3. **MUST** treat a `sharing_duration` of `0` as a Once Off Authorisation;
4. **MUST** treat a missing `sharing_duration` as a Once Off Authorisation;
4. **SHOULD** reject `sharing_duration` values less than `0`;
4. **MUST** support a claim parameter of `cdr_arrangement_id` referencing an existing CDR Arrangement (see CDR Arrangement Identifier);
5. **MUST** reject requests containing an unknown `cdr_arrangement_id`;

### Claims

The authorisation server:

1. **MUST** support the claim `refresh_token_expires_at` representing the expiry of the refresh token as a numeric epoch. If no refresh token has been provided a `0` **SHOULD** be returned;
2. **MUST** support the claim `sharing_expires_at` representing the expiry of the CDR Arrangement. If a `sharing_duration` was not specified in the authorisation request object then a `0` **SHOULD** be returned;
3. **SHOULD** ensure the `exp` claim within refresh tokens is equal to the `sharing_expires_at` claim;
1. **MUST** support the claim `cdr_arrangement_id` representing a unique identifier for the CDR Arrangement;
2. **MUST** ensure the `cdr_arrangement_id` is non-guessable;
3. **MUST** ensure the `cdr_arrangement_id` does not identify the Consumer;
4. **SHOULD** ensure the `cdr_arrangement_id` is collision resistant;
5. **MUST** ensure the `cdr_arrangement_id` claim, if specified within the request object and valid, remains static across authorisation flows;
6. **MUST** reject requests referencing a CDR Arrangement which has expired;
7. **MUST** reject requests referencing a CDR Arrangement that does not match the authenticated Consumer;
8. **MUST** revoke previously issued access and refresh tokens when a CDR Arrangement is updated

### Amending Authorisation

5. **MUST** trigger an amendment authorisation flow if a `cdr_arrangement_id` is provided

## Holder CDR Arrangement Revocation Endpoint (HCARE)

The Holder CDR Arrangement (HCARE) Revocation endpoint accepts a CDR Arrangement Identifier and immediately revokes the CDR Sharing Arrangement.

### HCARE Discovery

The HCARE endpoint **MUST** be advertised using the `cdr_arrangement_revocation_endpoint` attribute at the OpenID Connect Discovery 1.0 [@!OIDC-Discovery] endpoint.

### HCARE Request

The HCARE endpoint receives requests using the HTTP POST [@!RFC7231] method with parameters sent as `application/x-www-form-urlencoded` data as defined in [@!W3C.REC-html5-20141028].

In addition to authentication credentials described in Section 2.2 of [@!RFC7523] the endpoint includes the CDR Arrangement Identifier as an additional parameter, specified as follows:

- `cdr_arrangement_id`: **REQUIRED**  The CDR Arrangement Identifier previously provided in the token and introspection endpoint responses.

An example request to the Holder CDR Arrangement Revocation Endpoint is as follows:

```
POST https://data.holder.com.au/arrangements/revoke
HTTP/1.1
Host: data.holder.com.au
Content-Type: application/x-www-form-urlencoded

client_id=s6BhdRkqt3&
client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&
client_assertion=eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjEyNDU2In0.ey ...&
cdr_arrangement_id=5a1bf696-ee03-408b-b315-97955415d1f0
```

The authorisation server:

1. **MUST** validate the client credentials as per Section X.X of OAuth2 Client Auth (ref?!) and;
2. **MUST** validate the `cdr_arrangement_id` relates to a valid CDR Sharing Arrangement and;
3. **MUST** verify the CDR Sharing Arrangement referenced by specified `cdr_arrangement_id` belongs to validated client
4. If all of the above conditions are met the authorisation server:
   - **MUST** immediately invalidate the CDR Arrangement
   - **MUST** immediately invalidate all Refresh Tokens associated with the CDR Arrangement
   - **MUST** immediately invalid all Access Tokens associated with the CDR Arrangement

### HCARE Response

#### Successful Response

In the event of a successful revocation, the authorisation server:

1. If the CDR Sharing Arrangement is revoked, **MUST** respond with a 204 HTTP status code containing no content
2. If the CDR Sharing Arrangement was already revoked prior to the request being received, **SHOULD** respond with a 204 HTTP status code containing no content

#### Failure Response

In the event of a failure to validate the CDR Arrangement conditions the authorisation server:

1. **MUST** respond with a ### HTTP Status Code containing a JSON payload as per Section X.X of [@!CDRPLUS-ERROR-CODES-BASELINE]
2. **SHOULD** respond with a 503 HTTP Status Code if currently unavailable and **MAY** supply a `Retry-After` indicating the earliest retry time

The following additional error code is defined for the HCARE endpoint:

| Code                                                        | Description                         | Error Detail                            |
|-------------------------------------------------------------|-------------------------------------|-----------------------------------------|
| `urn:au-cds:error:cds-all:Authorisation/InvalidArrangement` | The arrangement could not be found. | The provided `cdr_arrangement_id` value |


# Software Product

The following provisions apply to participants operating individual Software Products.

## Authorisation Client

A Software Product which operates as an authorisation client:

1. **SHALL** supported the provisions specified in Section X of [@!CDRPLUS-INFOSEC-BASELINE]
2. **MUST** call the HCARE when requested by a Consumer
2. **MUST** call the HCARE after successful collection if operating on a Once Off Authorisation and a Refresh Token was supplied by the Holder
3. **MAY** during Request Object creation provide the `cdr_arrangement_id` of an existing arrangement
4. **MUST** support the Recipient CDR Arrangement Revocation Endpoint (RCARE)
5. **MUST** retry requests to the HCARE where the response received was a HTTP Status 503. The retry time **SHOULD** be informed by the presence of the `Retry-After` header.

## Recipient CDR Arrangement Revocation Endpoint (RCARE)

The Recipient CDR Arrangement (RCARE) Revocation endpoint is a CDR specific endpoint that accepts a CDR Arrangement Identifier and immediately revokes the CDR Sharing Arrangement.

### RCARE Request

The protected resource calls the RCARE endpoint using an HTTP POST [@!RFC7231] request with parameters sent as `application/x-www-form-urlencoded` data as defined in [@!W3C.REC-html5-20141028].

`cdr_arrangement_jwt`
**REQUIRED**. A single signed [@!JWT] containing the following parameters:

1. `cdr_arrangement_id` representing the CDR Arrangement Identifier previously provided in the token and introspection endpoint responses;
2. `iss`: The Holder Brand ID as per Section X.X in [@!CDRPLUS-ADMISSION-CONTROL]
3. `sub`: The Holder Brand ID as per Section X.X in [@!CDRPLUS-ADMISSION-CONTROL]
4. `aud`: The URI to the RCARE Endpoint as per Section X.X in [@!CDRPLUS-ADMISSION-CONTROL]
5. `jti`: A unique identifier for the token
6. `exp`: Expiration time on or after the JWT must not be accepted

In addition, the client includes an `Authorization` header parameter with a Bearer token containing a single signed [@!JWT] with the following parameters:

1. `iss`: The Holder Brand ID as per Section X.X in [@!CDRPLUS-ADMISSION-CONTROL]
2. `sub`: The Holder Brand ID as per Section X.X in [@!CDRPLUS-ADMISSION-CONTROL]
3. `aud`: The URI to the RCARE Endpoint as per Section X.X in [@!CDRPLUS-ADMISSION-CONTROL]
4. `jti`: A unique identifier for the token
5. `exp`: Expiration time on or after the JWT must not be accepted

For example, a Holder may request CDR Arrangement revocation with the following request:

```
POST https://data.recipient.com.au/arrangements/revoke HTTP/1.1
Host: data.recipient.com.au
Content-Type: application/x-www-form-urlencoded
Authorization: Bearer eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjEyNDU2In0.ey …

cdr_arrangement_jwt=eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjEyNDU2In0.ey ...&
```

The Recipient first validates the Holder credentials and then verifies whether the CDR Arrangement Identifier exists and was issued by the Holder making the CDR Arrangement Revocation request. If this validation fails, the request is refused and the Holder is informed of the error by the RCARE endpoint as described below.

In the next step, the Recipient invalidates the CDR Arrangement and **SHOULD** discard associated tokens. Data Minimisation events **SHOULD** also be triggered on receipt of this event.

### RCARE Response


The HCARE endpoint receives requests using the HTTP POST [@!RFC7231] method with parameters sent as `application/x-www-form-urlencoded` data as defined in [@!W3C.REC-html5-20141028].

In addition to authentication credentials described in Section 2.2 of [@!RFC7523] the endpoint includes the CDR Arrangement Identifier as an additional parameter, specified as follows:

- `cdr_arrangement_id`: **REQUIRED**  The CDR Arrangement Identifier previously provided in the token and introspection endpoint responses.

An example request to the Holder CDR Arrangement Revocation Endpoint is as follows:

```
POST https://data.holder.com.au/arrangements/revoke
HTTP/1.1
Host: data.holder.com.au
Content-Type: application/x-www-form-urlencoded

client_id=s6BhdRkqt3&
client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&
client_assertion=eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjEyNDU2In0.ey ...&
cdr_arrangement_id=5a1bf696-ee03-408b-b315-97955415d1f0
```

The authorisation server:

1. **MUST** validate the client credentials as per Section X.X of OAuth2 Client Auth (ref?!)
2. **MUST** validate the provided JWT in accordance with [@!JWT]
2. **MUST** validate the `cdr_arrangement_id` relates to a valid CDR Sharing Arrangement
3. **MUST** verify the CDR Sharing Arrangement referenced by specified `cdr_arrangement_id` belongs to the client validated by (1)

If all of the above conditions are met the authorisation server:

1. **MUST** immediately invalidate the CDR Arrangement
2. **MUST** immediately invalidate all Refresh Tokens associated with the CDR Arrangement
3. **MUST** immediately invalid all Access Tokens associated with the CDR Arrangement

### RCARE Response

In the event of a successful revocation, the RCARE:

1. If the CDR Sharing Arrangement is revoked, **MUST** respond with a 204 HTTP status code containing no content
2. If the CDR Sharing Arrangement was already revoked prior to the request being received, **SHOULD** respond with a 204 HTTP status code containing no content

In the event of a failure to validate the CDR Arrangement conditions the authorisation server:

- **MUST** respond with a ### HTTP Status Code containing a JSON payload formatted as per Section X.X of [@!CDRPLUS-ERROR-CODES-BASELINE]
- **SHOULD** respond with a 503 HTTP Status Code if currently unavailable and **MAY** supply a `Retry-After` indicating the earliest retry time

#### Error Codes

The following additional error code is defined for the HCARE endpoint:

| Code                                                        | Description                         | Error Detail                            |
|-------------------------------------------------------------|-------------------------------------|-----------------------------------------|
| `urn:au-cds:error:cds-all:Authorisation/InvalidArrangement` | The arrangement could not be found. | The provided `cdr_arrangement_id` value |

{backmatter}

<reference anchor="OIDC-Core" target="http://openid.net/specs/openid-connect-core-1_0.html"> <front> <title>OpenID Connect Core 1.0 incorporating errata set 1</title> <author initials="N." surname="Sakimura" fullname="Nat Sakimura"> <organization>NRI</organization> </author> <author initials="J." surname="Bradley" fullname="John Bradley"> <organization>Ping Identity</organization> </author> <author initials="M." surname="Jones" fullname="Mike Jones"> <organization>Microsoft</organization> </author> <author initials="B." surname="de Medeiros" fullname="Breno de Medeiros"> <organization>Google</organization> </author> <author initials="C." surname="Mortimore" fullname="Chuck Mortimore"> <organization>Salesforce</organization> </author> <date day="8" month="Nov" year="2014"/> </front> </reference>

<reference anchor="OIDC-Discovery" target="https://openid.net/specs/openid-connect-discovery-1_0.html"> <front> <title>OpenID Connect Discovery 1.0 incorporating errata set 1</title> <author initials="N." surname="Sakimura" fullname="Nat Sakimura"> <organization>NRI</organization> </author> <author initials="J." surname="Bradley" fullname="John Bradley"> <organization>Ping Identity</organization> </author> <author initials="M." surname="Jones" fullname="Mike Jones"> <organization>Microsoft</organization> </author> <author initials="E." surname="Jay"> <organization>Illumila</organization> </author><date day="8" month="Nov" year="2014"/> </front> </reference>

<reference anchor="JWT" target="https://datatracker.ietf.org/doc/html/rfc7519"> <front> <title>JSON Web Token (JWT)</title> <author fullname="M. Jones"> <organization>Microsoft</organization> </author> <author initials="J." surname="Bradley" fullname="John Bradley"> <organization>Ping Identity</organization> </author><author fullname="N. Sakimura"> <organization>Nomura Research Institute</organization> </author> <date month="May" year="2015"/></front> </reference>

<reference anchor="CDRPLUS-INFOSEC-BASELINE" target="https://cdrplus.github.io/cdrplus-infosec-baseline/draft-cdrplus-infosec-baseline.html"> <front><title>CDR+ Security Profile: Baseline</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="CDRPLUS-ERROR-CODES-BASELINE" target="https://cdrplus.github.io/cdrplus-error-codes-baseline/draft-authors-cdrplus-error-codes-baseline.html"> <front><title>CDR+ Error Codes: Baseline</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="CDRPLUS-ADMISSION-CONTROL" target="https://cdrplus.github.io/cdrplus-admission-control/draft-authors-cdrplus-admission-control.html"> <front><title>CDR Plus: Admission Control</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="FAPI-1.0-Baseline" target="https://openid.net/specs/openid-financial-api-part-1-1_0.html"> <front><title abbrev="FAPI 1.0 Baseline">Financial-grade API Security Profile 1.0 - Part 1: Baseline</title><author initials="N." surname="Sakimura" fullname="Nat Sakimura"><organization>Nat Consulting</organization></author><author initials="J." surname="Bradley" fullname="John Bradley"><organization>Yubico</organization></author><author initials="E." surname="Jay" fullname="Illumila"><organization>Illumila</organization></author></front> </reference>
