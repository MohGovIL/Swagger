# PCM Swagger/OpenAPI Implementer Changelog: `2026-01-05` Contract to `0.3.2`

## Purpose and scope

This changelog covers the implementer-facing changes from the MoH `2026-01-05` Swagger contract to PCM `0.3.2`. It addresses endpoints, transport and OAuth behavior, HTTP requests and responses, FHIR interaction payloads, search parameters, status codes, headers, schemas, examples, migration actions, and conformance tests. Both contracts use OpenAPI 3.1 and FHIR R4.

## Change classifications

The following labels are used throughout this guide:

| Label | Meaning for an implementer |
| --- | --- |
| **Breaking** | Existing URLs, credentials, serialized values, request bodies, response parsers, or workflows must change. |
| **New capability** | A new endpoint or supported interaction is available. |
| **Contract hardening** | Behavior that was previously loose, generic, or ambiguous is now expressed as an enforceable OpenAPI 3.1 schema or explicit runtime rule. Existing payloads may now be rejected. |
| **Clarification** | The Swagger now states behavior that clients must follow, even if an implementation may already have behaved that way. |

## Executive summary

The `0.3.2` Swagger is a breaking API revision, not a documentation-only refresh. The highest-impact changes are:

1. **Breaking:** The production OAuth origin changed from `https://pcm.fhir.health.gov.il` to `https://pcm2m.health.gov.il`; the production FHIR base changed from `https://pcm.fhir.health.gov.il/r4` to `https://pcm2m.health.gov.il/r4`.
2. **Breaking:** mTLS is now required on every PCM HTTP endpoint, including OAuth and SMART discovery, authorization, FHIR metadata, token, introspection, and all FHIR interactions. OAuth clients can no longer assume that discovery is anonymously retrievable.
3. **New capability:** Standalone SMART authorization-code flow was added through `GET /authorize` and PKCE S256. The client now selects a space-delimited standard SMART App Launch v2 patient-level resource-scope set; examples use `patient/*.rs`, and `user/...` scopes are unsupported.
4. **Breaking:** `GET /.well-known/jwks.json` and `jwks_uri` were removed from the public contract.
5. **Breaking:** `private_key_jwt` assertion signing changed from `ES384` to `ES256`, client identity changed from an Organization URL to an opaque PCM-assigned `client_id`, and assertion content is now tightly specified.
6. **Breaking:** The generic `client_credentials` token form became a strict union of authorization-code, PCM-management, introspection-authentication, Data Source-access, and presence-token request shapes.
7. **Breaking:** Token responses now have a fixed 30-second lifetime, always include `scope`, never include refresh tokens, and remain Bearer tokens even when certificate-correlation evidence is present.
8. **Breaking:** Introspection responses became typed, closed response variants. The old top-level `patient` and `cnf` fields are gone; typed `fhirContext` entries and `pcm_authentication_context` replace them.
9. **Contract hardening:** Every public FHIR `POST` and `PUT` now uses a dedicated closed request schema. Sending a complete resource back to PCM is no longer a safe update strategy.
10. **New capability:** Every declared FHIR type search is now available both as `GET /{type}` and as form-encoded `POST /{type}/_search`.
11. **New capability:** `POST /Organization` now supports an authorized Service Provider-child creation interaction.
12. **Breaking:** Organization certificate inventory and OAuth redirect projections are explicitly read-only and are prohibited in Organization writes.
13. **Breaking:** Endpoint create/update has a strict `active` / `suspended` / terminal `off` lifecycle, and the fixed payload coding changed from IHE XPHR to the standard `endpoint-payload-type|any` coding.
14. **Breaking:** HealthcareService lifecycle is now represented by a server-managed business-status extension. `PUT /HealthcareService/{id}` changes provider-instance business status; it no longer carries catalog update requests.
15. **New capability:** Non-material catalog changes now use `POST /HealthcareService/{id}/$request-catalog-update` and return a FHIR `Parameters` receipt with HTTP `200`, not a `202 OperationOutcome`.
16. **Breaking:** Consent create and deactivation requests now require one exact legal `Consent.policy`; `policyRule` is not accepted.
17. **Contract hardening:** Search authorization is explicitly a hard ceiling. Filters and `_include` can only narrow or safely expand the caller-authorized set; they never grant visibility.
18. **Breaking:** Three PCM coding-system URLs changed from the national `http://fhir.health.gov.il` namespace to `http://pcm.fhir.health.gov.il`.

## Recommended migration order

| Priority | Area | Required action |
| ---: | --- | --- |
| 1 | Connectivity | Replace operational production URLs and make a client certificate available to discovery, authorization, token, introspection, metadata, and FHIR calls. |
| 2 | OAuth assertions | Switch to `ES256`, use the assigned opaque `client_id`, enforce the exact assertion audiences, and regenerate a fresh assertion per request/retry. |
| 3 | Token clients | Implement the strict request variant that applies to each token class; replace `system/*.cruds` with `system/*.crus` where PCM management or introspection authentication is intended. |
| 4 | Introspection consumers | Replace the old generic response parser with active Data Source-access, active online-presence, active batch-presence, and inactive variants. |
| 5 | FHIR writers | Stop serializing complete returned resources into `POST` or `PUT`; implement the interaction-specific request schemas. |
| 6 | HealthcareService | Add business status, the provider transition matrix, and the dedicated catalog-update operation. |
| 7 | Consent | Add the fixed policy, strict patient identifier handling, closed deactivation body, and business-status parsing. |
| 8 | Organization and Endpoint | Remove security projections and Organization state from writes; implement Service Provider creation and Endpoint lifecycle rules. |
| 9 | Search and sync | Add `_lastUpdated`, new business-status/linkage searches, optional `POST _search`, typed search Bundles, and authorization-safe include handling. |
| 10 | Validation | Update exact coding-system URI comparisons, generated models, test fixtures, error handling, and HTTP header handling. |

## API surface at a glance

### Combined route inventory

The combined entry point grew from 14 path items to 19:

| Change | Route | Notes |
| --- | --- | --- |
| Removed | `GET /.well-known/jwks.json` | No public JWKS endpoint is declared in `0.3.2`. |
| Added | `GET /authorize` | Standalone SMART authorization-code initiation. |
| Added | `POST /Organization/_search` | Form-encoded equivalent of Organization GET search. |
| Added | `POST /Endpoint/_search` | Form-encoded equivalent of Endpoint GET search. |
| Added | `POST /HealthcareService/_search` | Form-encoded equivalent of HealthcareService GET search. |
| Added | `POST /Consent/_search` | Form-encoded equivalent of Consent GET search. |
| Added | `POST /HealthcareService/{id}/$request-catalog-update` | Resource-instance-level FHIR operation for non-material catalog proposals. |

`POST /Organization` is also new, but it was added as a second operation on the existing `/Organization` path and therefore does not add another path item.

The FHIR specification changed from 10 paths and 21 operations to 15 paths and 27 operations. The OAuth specification still has four paths and four operations because `/authorize` replaced the removed JWKS route.

### Endpoint-level interaction changes

| Endpoint | Reference contract | `0.3.2` contract | Migration impact |
| --- | --- | --- | --- |
| `/.well-known/smart-configuration` | mTLS discovery; client-credentials-oriented metadata | mTLS discovery; standalone SMART plus client credentials; ES256; no JWKS | Update discovery parser and advertised capabilities. |
| `/.well-known/oauth-authorization-server` | Anonymous metadata; JWKS advertised | mTLS metadata; authorization endpoint advertised; no JWKS; ES256 | Configure mTLS before metadata fetch. |
| `/.well-known/jwks.json` | `GET` supported | Removed | Remove dependencies on public PCM signing keys. |
| `/authorize` | Absent | `GET` supported | Implement SMART authorization initiation if using the standalone flow. |
| `/token` | Generic client-credentials form | Five strict request variants, including authorization code and presence | Select and serialize one exact request type. |
| `/introspect` | Generic request and generic response object | Two strict authentication forms and four response variants | Replace request and response models. |
| `/Organization` | Search only | Search plus authorized Service Provider-child creation | Add create workflow only for eligible callers. |
| `/Organization/_search` | Absent | Form-encoded search | Optional alternative to GET search. |
| `/Organization/{id}` | Read, HEAD, generic update | Read, HEAD, closed type-specific update | Replace update serializer. |
| `/Endpoint` | Search and generic create | Search and closed create | Replace create serializer and payload coding. |
| `/Endpoint/_search` | Absent | Form-encoded search | Optional alternative to GET search. |
| `/Endpoint/{id}` | Read, HEAD, generic update | Read, HEAD, closed lifecycle/update | Implement suspension, address, replacement, and retirement rules. |
| `/HealthcareService` | Search and generic create | Search and strict two-shape create | Replace create serializer; parse business status. |
| `/HealthcareService/_search` | Absent | Form-encoded search | Add registry/catalog filters if needed. |
| `/HealthcareService/{id}` | Read, HEAD, `active` update, and catalog proposal through PUT | Read, HEAD, provider-instance business-status update only | Replace state update and stop catalog PUT. |
| `/HealthcareService/{id}/$request-catalog-update` | Absent | FHIR operation | Use for eligible non-material catalog changes. |
| `/Consent` | Search and generic create | Search and closed create with fixed legal policy | Replace create serializer. |
| `/Consent/_search` | Absent | Form-encoded search | Optional alternative to GET search. |
| `/Consent/{id}` | Read, HEAD, broad status update description | Read, HEAD, owner deactivation only | Remove public approval/rejection/revocation use. |

## 1. Deployment URLs, identifiers, and mandatory mTLS

### Production origins changed

**Classification: Breaking**

| Purpose | Reference value | `0.3.2` production value |
| --- | --- | --- |
| OAuth issuer/origin | `https://pcm.fhir.health.gov.il` | `https://pcm2m.health.gov.il` |
| FHIR REST base | `https://pcm.fhir.health.gov.il/r4` | `https://pcm2m.health.gov.il/r4` |
| Authorization endpoint | Not present | `https://pcm2m.health.gov.il/authorize` |
| Token endpoint | `https://pcm.fhir.health.gov.il/token` | `https://pcm2m.health.gov.il/token` |
| Introspection endpoint | `https://pcm.fhir.health.gov.il/introspect` | `https://pcm2m.health.gov.il/introspect` |

Non-production environments use the origin and FHIR base assigned to that environment. Implementers should make operational endpoints environment configuration, not derive them from canonical FHIR URLs.

This distinction is critical:

- Operational URLs under `https://pcm2m.health.gov.il` are network destinations.
- Canonical profile, extension, CodeSystem, SearchParameter, identifier-system, policy, and context-role URIs continue to use their published `pcm.fhir.health.gov.il` identifiers where specified.
- Participant URLs under `example.org` in examples remain placeholders.

Do not perform a global string replacement from `pcm.fhir.health.gov.il` to `pcm2m.health.gov.il`. Doing so would corrupt identifiers such as:

```text
http://pcm.fhir.health.gov.il/identifier/pcm-consent-id
http://pcm.fhir.health.gov.il/identifier/pcm-organization-id
http://pcm.fhir.health.gov.il/cs/pcm-service-business-status
https://pcm.fhir.health.gov.il/consent-policy/medical-information-mobility-law-2024
```

### mTLS now covers every PCM HTTP endpoint

**Classification: Breaking and contract hardening**

The reference OAuth authorization-server metadata and JWKS operations explicitly used `security: []`. In `0.3.2`:

- OAuth authorization-server metadata requires mTLS.
- SMART configuration requires mTLS.
- FHIR CapabilityStatement metadata requires mTLS.
- `/authorize` requires mTLS.
- `/token` requires mTLS.
- `/introspect` requires mTLS.
- all FHIR searches, reads, creates, and updates require mTLS;
- FHIR resource operations additionally require an OAuth bearer token.

mTLS remains transport authentication. It is not the OAuth client-authentication method. Token and direct-assertion introspection callers additionally authenticate with `private_key_jwt`.

#### Why this changed

The public contract now applies one transport-security boundary consistently. Discovery and metadata disclose security and endpoint configuration and are part of the protected PCM surface, not anonymously accessible bootstrap endpoints.

#### Migration action

- Configure OAuth discovery and SMART libraries to present the assigned client certificate during metadata fetch.
- Ensure browser, reverse-proxy, or native-app handling for `/authorize` can complete the mTLS connection.
- Do not expect an application-level JSON `401` when TLS negotiation itself fails; TLS failures occur before the HTTP application contract.
- Keep mTLS-certificate selection separate from `private_key_jwt` signing-key selection. They may be the same registered certificate or different registered certificates.

## 2. OAuth and SMART discovery

### OAuth authorization-server metadata changed

**Classification: Breaking**

`GET /.well-known/oauth-authorization-server` now advertises the standalone authorization-code flow and no longer advertises a JWKS URI.

| Metadata element | Reference contract | `0.3.2` |
| --- | --- | --- |
| Retrieval security | No OpenAPI security requirement | mTLS required |
| `issuer` | Old OAuth origin | Current environment OAuth origin |
| `authorization_endpoint` | Absent | Required |
| `token_endpoint` | Required | Required, current origin |
| `introspection_endpoint` | Required | Required, current origin |
| `jwks_uri` | Required | Removed |
| `grant_types_supported` | `client_credentials` | `authorization_code`, `client_credentials` |
| `response_types_supported` | Absent | Required; current value includes `code` |
| `code_challenge_methods_supported` | Absent | Required; current value includes `S256` |
| Token assertion algorithm | Example advertised `ES384` | ES256 is the current required baseline |
| Introspection assertion algorithm | Example advertised `ES384` | ES256 is the current required baseline |
| `tls_client_certificate_bound_access_tokens` | Absent | Required and fixed to `false` |
| `registration_endpoint` | Absent | Optional schema property, omitted until a supported registration API is enabled |

Clients must tolerate additional metadata properties and additional advertised algorithm values. At present, assertions must use ES256. The forward-compatibility requirement means a parser must not fail merely because a future metadata response contains another compatible advertised value alongside ES256.

### SMART configuration changed

**Classification: Breaking**

`GET /.well-known/smart-configuration` now describes a real standalone launch rather than placeholder “none” response and PKCE values.

| SMART field | Reference example | `0.3.2` example |
| --- | --- | --- |
| `authorization_endpoint` | Absent | Current `/authorize` endpoint |
| `grant_types_supported` | `client_credentials` | `authorization_code`, `client_credentials` |
| `response_types_supported` | `none` | `code` |
| `response_modes_supported` | `none` | `query` |
| Assertion algorithm | `ES384` | `ES256` |
| `jwks_uri` | Present | Removed |
| `tls_client_certificate_bound_access_tokens` | Absent | `false` |
| `code_challenge_methods_supported` | `none` | `S256` |
| `scopes_supported` | Enumerates legacy per-resource `system/...read` and `system/...write` values | Omitted; PCM uses standard SMART v2 scope syntax and advertises no custom non-FHIR scopes |
| Standalone scope handling | Absent | Client-selected, space-delimited standard SMART v2 patient-level resource scopes; examples use `patient/*.rs`, and `user/...` scopes are unsupported |
| Capabilities | `client-confidential-asymmetric`, `permission-v2` | Adds `launch-standalone` and `permission-patient`; `permission-user` is not advertised because PCM does not support user-level app-launch scopes |

The fixed backend-management `system/*.crus` rule still exists in the applicable `/token` request schemas. It is not a list of app-launch scopes and is no longer advertised through `scopes_supported`.

The current SMART schema permits an optional `registration_endpoint`, but the example omits it and no registration operation is defined. Do not infer dynamic client registration from the schema property. Likewise, do not interpret the absence of `scopes_supported` as the absence of SMART resource scopes: PCM accepts standard SMART App Launch v2 syntax and does not use that array to enumerate the standard language.

### Public JWKS was removed

**Classification: Breaking**

The following are both removed:

```text
GET /.well-known/jwks.json
jwks_uri in OAuth/SMART metadata
```

PCM access tokens are opaque Bearer tokens and are evaluated through introspection. `private_key_jwt` assertions are signed by the client with a key registered out of band for that opaque client ID. The current public contract therefore does not give clients a PCM JWKS endpoint to call.

#### Migration action

- Remove startup or refresh logic that fetches `/.well-known/jwks.json`.
- Do not attempt local JWT validation of opaque PCM access tokens.
- Continue to create and sign client assertions with the caller’s registered private key.
- Use `/introspect` when the Data Source needs the authoritative subject-token authorization context.

## 3. New standalone SMART authorization-code flow

### `GET /authorize`

**Classification: New capability**

The new authorization request requires all of the following query parameters:

| Parameter | Required value or rule |
| --- | --- |
| `response_type` | Exactly `code` |
| `client_id` | Opaque PCM-assigned Service Provider application client ID |
| `redirect_uri` | Exact registered standalone SMART redirect URI |
| `scope` | Non-empty, space-delimited standard SMART App Launch v2 patient-level resource-scope string selected by the client; the reference example is `patient/*.rs`; `user/...` is unsupported |
| `state` | Non-empty, unpredictable client-generated value |
| `code_challenge` | PKCE verifier challenge, 43–128 characters |
| `code_challenge_method` | Exactly `S256` |
| `aud` | Exact FHIR REST base for the environment; production is `https://pcm2m.health.gov.il/r4` |

PCM does not publish an enum or one fixed value for app-launch scopes. It accepts standard SMART v2 patient-level resource scopes and rejects `user/...` scopes. Patient-level FHIR resource scopes can use SMART v2 query constraints, including the [experimental SMART search syntax](https://build.fhir.org/ig/HL7/smart-app-launch/scopes-and-launch-context.html#experimental-features-exp) for search modifiers, chaining, and reverse chaining. `_filter` is explicitly unsupported. Clients sending a constrained scope must URL-encode the complete parameter value, especially `&` characters inside the scope, so the authorization endpoint does not parse them as separate query parameters.

The requested set is input to authorization, not a guarantee. PCM can grant a different or narrower set based on the client, user, service definition, and authorization policy. The client must read the `scope` returned from the token endpoint.

The request itself requires mTLS. It does not contain a client assertion. Asymmetric OAuth client authentication occurs when the authorization code is exchanged at `/token`.

Possible documented outcomes are:

- `200 text/html` when an interactive authorization or consent page is required;
- `302` to the exact registered `redirect_uri`, with `code` and the original `state`;
- `302` to the exact registered `redirect_uri`, with OAuth `error` and the original `state`;
- `400` with an OAuth error for an invalid authorization request.

PCM does not redirect to an unregistered or mismatched redirect URI.

### Authorization-code exchange

The new `AuthorizationCodeTokenRequest` is a closed form containing:

```text
grant_type=authorization_code
code=<single-use authorization code>
redirect_uri=<exact URI used in the authorization request>
code_verifier=<43-to-128-character PKCE verifier>
client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
client_assertion=<fresh ES256 private_key_jwt>
```

The Swagger does not include `scope` or `resource` in this request variant. Do not reuse the client-credentials serializer for the code exchange.

The example client-facing token response returns `scope=patient/*.rs`. That value represents the app-launch grant returned to the client, not the final Data Source enforcement expression. PCM derives the effective resource, interaction, information-bucket, history, and other search constraints from the applicable service definition and authorization context. A Data Source receives that authoritative effective scope in the active introspection response; it may be narrower or otherwise different from the client-facing `patient/*.rs` example.

#### Why this changed

The earlier discovery document advertised no usable authorization response or PKCE method. `0.3.2` defines a complete confidential asymmetric standalone SMART flow while retaining client credentials for server-to-server token classes. Accepting the standard SMART v2 scope language also lets each client request the access it needs instead of forcing every application into one wildcard user scope. Keeping the app-launch grant distinct from the introspection result allows PCM to apply the approved service definition before a Data Source enforces access.

#### Migration action

- Generate and retain `state` and the PKCE verifier per authorization attempt.
- Validate the returned `state` before using the code.
- Enforce exact redirect URI matching.
- Generate a new ES256 client assertion for the token exchange.
- Select the minimum standard SMART v2 patient-level resource scopes required by the application and retain the requested set with the authorization transaction; do not request `user/...`.
- Read the returned `scope`; do not assume PCM granted the request unchanged.
- Do not send `_filter` in a scope.
- Treat the active introspection `scope`, not the client-facing wildcard example, as the authoritative Data Source permission set.

## 4. Client assertions and certificate correlation

### Signing algorithm and client identity changed

**Classification: Breaking**

| Assertion property | Reference contract | `0.3.2` |
| --- | --- | --- |
| JOSE `alg` | ES384 expected by examples and descriptions | Exactly ES256 at present |
| JOSE `typ` | `JWT` in examples | Exactly `JWT` |
| JOSE key hints | An old introspection example included `kid` | Current profile omits `kid`, `x5c`, `x5t#S256`, and `jku` from the header |
| `iss` | Organization URI in examples | Opaque PCM-assigned OAuth `client_id` |
| `sub` | Organization URI in examples | Same opaque `client_id` as `iss` |
| `aud` | Old token/introspection URL; token endpoint audience was tolerated by old introspection description | Exact receiving endpoint in the current environment |
| `jti` | Required | Required and unique |
| `iat` | Required | Required |
| `exp` | Short-lived | Required and no more than five minutes after `iat` |
| Retry behavior | Not explicit | A retry uses a newly generated assertion |

For production, the token assertion audience is:

```text
https://pcm2m.health.gov.il/token
```

For direct assertion authentication at introspection, it is:

```text
https://pcm2m.health.gov.il/introspect
```

Use the assigned endpoint values outside production. The previous compatibility text that allowed a token-endpoint audience on an introspection assertion is gone.

### mTLS and assertion-signing certificates

The issued access token is still a Bearer token. PCM does not advertise RFC 8705 certificate-bound access tokens.

When one registered certificate is used for both mTLS and assertion signing, that credential is used for both roles. If PCM registered different certificates for the client:

- the assertion should contain `cnf.x5t#S256`;
- the value identifies the certificate actually presented on the current mTLS connection;
- PCM validates a supplied value against the observed connection;
- the thumbprint is exactly 43 unpadded base64url characters, representing SHA-256 over the DER certificate;
- this correlates the two client credentials but does not bind the resulting Bearer access token.

#### Migration action

- Replace ES384 keys/signing configuration with the assigned ES256 credential.
- Stop putting an Organization URL into `iss` and `sub`; use the exact assigned opaque client ID.
- Remove JOSE header `kid` or embedded certificate/key-discovery fields unless later authoritative guidance explicitly reintroduces them.
- Create a unique `jti` and fresh assertion for each HTTP request, including retries.
- Use `cnf.x5t#S256` only according to the registered credential arrangement, and populate it with the mTLS certificate thumbprint rather than the assertion-signing certificate thumbprint.

## 5. Token endpoint request variants

### The generic form became a strict union

**Classification: Breaking and contract hardening**

The reference `TokenRequest` was one open object with `grant_type=client_credentials`, assertion fields, optional `scope`, and `resource`. The current `TokenRequest` is a `oneOf` union:

| Request schema | Grant | Scope | Resource | Additional context |
| --- | --- | --- | --- | --- |
| `AuthorizationCodeTokenRequest` | `authorization_code` | Not in this variant | Not in this variant | Code, exact redirect URI, PKCE verifier |
| `PCMManagementTokenRequest` | `client_credentials` | Exactly `system/*.crus` | Exact PCM FHIR base | No patient, Consent, service, B2B, or launch context |
| `IntrospectionBearerTokenRequest` | `client_credentials` | Exactly `system/*.crus` | Exact PCM introspection endpoint | Produces the short-lived bearer credential used to authenticate an introspection call |
| `DataSourceAccessTokenRequest` | `client_credentials` | Service-derived SMART v2 access scope; not management or presence scope | Exact address of the selected Data Source’s currently referenced active Endpoint | Strict `extensions.hl7-b2b` assertion content |
| `PresenceTokenRequest` | `client_credentials` | Exact online or batch count-only Patient-search pattern | Exact address of the selected Data Source’s currently referenced active Endpoint | PCM presence-orchestrator assertion profile |

Every variant has `additionalProperties: false`. A mixed request or a request containing fields from two variants fails rather than being partially interpreted.

#### Why this changed

The earlier generic form could not reliably express which combinations of grant, scope, audience, assertion context, and role were valid. The strict union makes token class selection unambiguous, prevents context intended for one flow from leaking into another, and lets clients fail invalid requests before transmitting credentials.

### Management scope changed

**Classification: Breaking**

The reference examples used:

```text
system/*.cruds
```

The current exact management and introspection-authentication scope is:

```text
system/*.crus
```

The current scope is an authorization ceiling. It does not grant every operation. Caller role, ownership, resource visibility, Organization state, writable fields, lifecycle rules, and the operation’s declared contract still apply.

### RFC 8707 resource handling

The one-resource-server rule remains and is now specialized by token class:

- exactly one `resource` is permitted per applicable client-credentials request;
- PCM management uses the exact FHIR base, not an individual resource URL;
- introspection-authentication uses the exact `/introspect` URL;
- Data Source-access and presence tokens use the exact current `Endpoint.address`;
- a suspended or `off` Endpoint is not eligible;
- an address replaced on the Endpoint is no longer a valid audience;
- multiple `resource` parameters or an invalid target return `invalid_target`.

Clients must obtain one token per resource server.

### Data Source-access B2B assertion became fully specified

**Classification: Breaking and contract hardening**

The earlier Swagger required a broadly described UDAP B2B extension with Organization, Consent reference, and purpose information. `0.3.2` defines a closed `DataSourceAccessAssertionClaims` object and describes use of the HL7 B2B object syntax without claiming the full UDAP token-request profile.

The assertion requires:

- `iss`, `sub`, `aud`, `jti`, `iat`, and `exp`;
- optional certificate-correlation `cnf`;
- an `extensions` object containing exactly `hl7-b2b`;
- no additional top-level Data Source-access assertion claims.

`extensions.hl7-b2b` requires:

| Field | Current rule |
| --- | --- |
| `version` | Exactly string `"1"` |
| `organization_id` | Requesting Service Provider’s stable PCM Organization identifier in `system#value` URI form |
| `purpose_of_use` | Non-empty unique array; release 1 value is `http://terminology.hl7.org/CodeSystem/v3-ActReason#TREAT` |
| `consent_policy` | Non-empty unique array; release 1 value is the fixed PCM legal-policy URI |
| `consent_reference` | Non-empty array of absolute, resolvable PCM Consent instance URLs using the environment’s HTTPS origin and exact `/r4/Consent/{logical-id}` path |

The Organization identifier is:

```text
http://pcm.fhir.health.gov.il/identifier/pcm-organization-id#{value}
```

It identifies the requesting Service Provider, not the target Data Source. `{value}` is the PCM Organization business identifier, not the FHIR logical id.

The exact legal policy URI is:

```text
https://pcm.fhir.health.gov.il/consent-policy/medical-information-mobility-law-2024
```

The purpose value changed from the reference example’s bare `TREAT` code to the URI-form value:

```text
http://terminology.hl7.org/CodeSystem/v3-ActReason#TREAT
```

The old example’s Organization-resource URL is not a valid `organization_id`, and a Consent business identifier is not a valid `consent_reference`.

An abbreviated decoded payload now looks like:

```json
{
  "iss": "pcm-client-hospital-a",
  "sub": "pcm-client-hospital-a",
  "aud": "https://pcm2m.health.gov.il/token",
  "jti": "unique-request-id",
  "iat": 1706640000,
  "exp": 1706640300,
  "extensions": {
    "hl7-b2b": {
      "version": "1",
      "organization_id": "http://pcm.fhir.health.gov.il/identifier/pcm-organization-id#PCM-ORG-HOSP-A-SP-001",
      "purpose_of_use": [
        "http://terminology.hl7.org/CodeSystem/v3-ActReason#TREAT"
      ],
      "consent_policy": [
        "https://pcm.fhir.health.gov.il/consent-policy/medical-information-mobility-law-2024"
      ],
      "consent_reference": [
        "https://pcm2m.health.gov.il/r4/Consent/consent-123"
      ]
    }
  }
}
```

### Presence-token request classes are new

**Classification: New capability**

The current Swagger defines two count-only patient-presence scopes:

| Variant | Scope form | Introspection context |
| --- | --- | --- |
| Online presence | Exactly one `system\|value` identifier | One typed Patient `fhirContext` item |
| Batch presence | At least two comma-separated `system\|value` identifiers | No `fhirContext`; the patient set remains encoded in `scope` |

Examples:

```text
system/Patient.s?identifier=http://fhir.health.gov.il/identifier/il-national-id|000000018
```

```text
system/Patient.s?identifier=system-a|value-a,system-b|value-b
```

The `s` permission is count-only. Presence requests use the PCM-owned presence-orchestrator credential profile and never use `system/*.crus`.

The Swagger intentionally does not yet define batch size limits, oversized-batch response behavior, or common batch timeout/retry rules. Implementers must not invent interoperability assumptions for those values.

### Token issuance revalidates operational state

**Classification: Clarification with runtime impact**

For Data Source-access and presence tokens, PCM checks that:

- the relevant Parent and child Organizations are operationally active;
- the target Data Source is active;
- its currently referenced participant Endpoint is active;
- `resource` exactly equals that Endpoint’s current `address`;
- for Data Source access, the Consent is active and usable;
- the referenced HealthcareService is a provider instance, not a catalog entry;
- both the provider instance and its catalog are `businessStatus=active` and `active=true`.

Endpoint status does not prevent an otherwise authorized Data Source from obtaining a PCM-management token needed to resume or replace its Endpoint.

## 6. Token responses and OAuth errors

### Token response is now closed and fixed

**Classification: Breaking and contract hardening**

The current `TokenResponse`:

- has `additionalProperties: false`;
- requires `access_token`, `token_type`, `expires_in`, and `scope`;
- fixes `token_type` to `Bearer`;
- fixes `expires_in` to `30` seconds;
- contains an opaque token;
- contains no refresh token;
- contains no standard top-level `cnf`;
- contains no patient, `fhirUser`, or launch context.

The reference response allowed an arbitrary configured lifetime and even described negative test lifetimes. That is no longer the public `0.3.2` response contract.

#### Migration action

- Make `scope` mandatory in the success parser.
- Treat `expires_in` as 30 seconds and still calculate expiry from receipt time safely.
- Remove test logic that expects negative lifetimes from the public contract.
- Do not parse the opaque token as a JWT.
- Do not persist or request refresh tokens for these flows.

### Token errors are now separated by category

**Classification: Contract hardening**

| HTTP status | Current response | Representative OAuth errors |
| ---: | --- | --- |
| `400` | `TokenRequestError` | `invalid_request`, `invalid_grant`, `unauthorized_client`, `invalid_scope`, `invalid_target` |
| `401` | `InvalidClient` | `invalid_client`, including an invalid assertion or mismatched supplied certificate thumbprint |

The reference Swagger used the same generic `OAuthError` component for both `400` and `401`.

Client code should branch first on HTTP status and then on `error`. Do not retry an unchanged request for structural, scope, target, code, or credential failures.

## 7. Introspection request changes

### Caller authentication is an exclusive choice

**Classification: Breaking and contract hardening**

The current `IntrospectionRequest` is a `oneOf`:

1. `BearerAuthenticatedIntrospectionRequest`
   - form body requires only the subject `token`;
   - optional `token_type_hint` is exactly `access_token`;
   - the caller credential is the Bearer token in the HTTP `Authorization` header;
   - mTLS is also required.
2. `DirectAssertionIntrospectionRequest`
   - form body requires the subject `token`, `client_assertion_type`, and `client_assertion`;
   - optional `token_type_hint` is exactly `access_token`;
   - the assertion audience is the exact introspection endpoint;
   - mTLS is also required.

Both forms are closed. Supplying both caller authentication methods is invalid.

The subject token in the form is distinct from the optional bearer caller credential in the HTTP header.

#### Migration action

- Model caller authentication separately from the token being inspected.
- Do not send assertion fields when using bearer caller authentication.
- Do not send the bearer caller credential when using direct assertion authentication.
- Change direct introspection assertions to the exact `/introspect` audience; token-endpoint audience reuse is no longer declared compatible.

## 8. Introspection response changes

### The generic object became four closed variants

**Classification: Breaking**

The current response is one of:

| Variant | Required distinguishing content |
| --- | --- |
| Active Data Source-access | `active=true`, non-presence scope, ordinary client ID, Service Provider `organization_id`, exactly three typed `fhirContext` items |
| Active online presence | `active=true`, online presence scope, `client_id=pcm-presence-orchestrator`, exactly one Patient context, no `organization_id` |
| Active batch presence | `active=true`, batch presence scope, `client_id=pcm-presence-orchestrator`, no `organization_id`, no `fhirContext` |
| Inactive | Exactly the privacy-safe `{"active":false}` shape |

Every active response requires:

- `active=true`;
- `scope`;
- opaque `client_id`;
- `pcm_authentication_context`;
- `exp`;
- `iss`;
- `aud`;
- `sub`.

`iat` may also be returned but is not required by the current base schema.

#### Why this changed

The reference response mixed client identity, patient context, certificate evidence, and token state in an open object. Typed variants let a Data Source distinguish ordinary access from online and batch presence without guessing from missing fields. The minimal inactive variant deliberately avoids leaking whether a token or authorization relationship exists.

### Data Source-access context is now typed

The active Data Source-access response requires exactly one of each:

| `fhirContext.type` | Exact role URI | Identifier |
| --- | --- | --- |
| `Patient` | `https://pcm.fhir.health.gov.il/fhir-context-role/authorization-patient` | Patient identifier system and value |
| `Consent` | `https://pcm.fhir.health.gov.il/fhir-context-role/consent` | `http://pcm.fhir.health.gov.il/identifier/pcm-consent-id` plus business identifier value |
| `HealthcareService` | `https://pcm.fhir.health.gov.il/fhir-context-role/healthcare-service` | `http://pcm.fhir.health.gov.il/identifier/pcm-healthcareservice-instance-id` plus business identifier value |

The response `organization_id` is the URI-form stable Service Provider identifier:

```text
http://pcm.fhir.health.gov.il/identifier/pcm-organization-id#{value}
```

The old response fields must be migrated as follows:

| Reference response | Current response |
| --- | --- |
| `client_id` as an Organization URL | Opaque PCM OAuth client ID |
| `organization_id` as a short logical value | Stable PCM Organization identifier in `system#value` URI form |
| Top-level `patient` token string | Typed Patient `fhirContext` item |
| Untyped Consent/HealthcareService context objects | Typed objects with exact role URIs and constrained identifiers |
| Top-level `cnf.x5t#S256` | `pcm_authentication_context` with both observed certificate roles |
| Old HTTP issuer value | Current environment HTTPS OAuth issuer |

### Authentication context replaced top-level `cnf`

Each active response contains:

```json
{
  "pcm_authentication_context": {
    "token_endpoint_mtls": {
      "x5t#S256": "<43-character base64url SHA-256 thumbprint>"
    },
    "client_assertion_signing": {
      "x5t#S256": "<43-character base64url SHA-256 thumbprint>"
    },
    "correlation_verified": true
  }
}
```

This is informational PCM context:

- `token_endpoint_mtls` is the certificate presented when the subject token was issued;
- `client_assertion_signing` is the certificate whose registered public key verified the assertion;
- `correlation_verified=true` confirms that PCM accepted the credential relationship and any submitted assertion `cnf`;
- it is not a standard token `cnf` claim;
- it does not turn the access token into a certificate-bound token;
- a Data Source may compare the current protected-resource mTLS certificate with `token_endpoint_mtls` under local policy.

### Inactive results are intentionally non-diagnostic

Unknown, inactive, wrong-audience, and other-Data-Source subject tokens receive:

```json
{
  "active": false
}
```

The closed inactive schema permits no explanatory identity or authorization fields. Implementers must not rely on inactive responses to distinguish non-existence from ineligibility.

### Runtime reevaluation and caching

**Classification: Clarification with runtime impact**

Introspection re-evaluates current:

- Service Provider and Parent state;
- target Data Source and Parent state;
- the Data Source’s currently referenced Endpoint and its address/status;
- Consent state;
- provider-instance HealthcareService state;
- catalog HealthcareService state.

An Endpoint suspension, retirement, or address replacement can therefore turn a previously issued subject token inactive without changing the Consent.

For one-time Consent access:

- each approved Data Source has an independent one-hour window;
- that Data Source’s first otherwise-valid active introspection starts its window;
- token issuance alone does not start the window;
- another Data Source has a separate window;
- after the source-specific window closes, introspection returns inactive and a new token does not reopen it.

An active introspection result may be cached only through the returned `exp`. Because current subject tokens last 30 seconds, the cache must never survive longer than the token’s remaining lifetime or 30 seconds. The Data Source must continue to enforce audience, scope, context, and every other returned constraint while using a cached result.

### Introspection errors are now specific

| HTTP status | Meaning | Current error behavior |
| ---: | --- | --- |
| `400` | Malformed or ambiguous request | `invalid_request`; subject token missing or both caller-auth methods supplied |
| `401` | Caller authentication failed | `invalid_client` for direct assertion or `invalid_token` for bearer caller authentication; bearer failure can include `WWW-Authenticate` |
| `403` | Authenticated caller lacks Data Source introspection authorization | `insufficient_scope` |
| `200` with `active=false` | Subject token is unknown, inactive, wrong audience, or not visible to this Data Source | Privacy-safe inactive response, not an OAuth endpoint error |

## 9. Cross-cutting FHIR interaction changes

### Generic write schemas were replaced

**Classification: Breaking and contract hardening**

| Interaction | Reference request schema | Current request schema |
| --- | --- | --- |
| Service Provider Organization create | Not supported | `PCMOrganizationServiceProviderCreateRequest` |
| Organization update | `Organization` | `PCMOrganizationUpdateRequest` |
| Endpoint create | `Endpoint` | `PCMEndpointCreateRequest` |
| Endpoint update | `Endpoint` | `PCMEndpointUpdateRequest` |
| HealthcareService create | `HealthcareService` | `PCMHealthcareServiceCreateRequest` |
| HealthcareService instance state update | `HealthcareService` | `PCMHealthcareServiceBusinessStatusUpdateRequest` |
| HealthcareService catalog proposal | Catalog content through generic PUT | `PCMRequestCatalogUpdateParameters` on a dedicated operation |
| Consent create | `Consent` | `PCMConsentCreateRequest` |
| Consent deactivation | `Consent` | `PCMConsentDeactivateRequest` |

Every current public request schema is closed with `additionalProperties: false` at its interaction boundary.

### Request-versus-response separation

The current Swagger distinguishes:

- lean or purpose-specific request projections that contain only caller-writable content; and
- complete authoritative response schemas containing PCM-managed identity, lifecycle, ownership, policy, linkage, and security projections.

Do not:

1. read a complete Organization, Endpoint, HealthcareService, or Consent;
2. change one field;
3. serialize the whole object back to `PUT`.

That pattern now sends prohibited content and should receive an OperationOutcome.

#### Why this changed

A complete returned resource contains identity, ownership, lifecycle, security projections, and server-derived values that the caller does not own. Separate closed request projections prevent an ordinary update from accidentally overwriting or claiming those fields and give implementers a deterministic serialization contract for each interaction.

### Atomic rejection replaces silent business-field handling

The reference HealthcareService descriptions said some non-writable values could be ignored or overwritten. The current top-level FHIR contract says:

- a prohibited business element rejects the entire request;
- an authorization failure rejects the entire request;
- an invalid transition rejects the entire request;
- no valid subset is silently applied;
- unsupported modifier extensions reject the request;
- unknown ordinary non-modifier extensions may be accepted, but PCM does not interpret them and does not guarantee round-trip preservation.

This is a breaking behavioral expectation even when the desired writable value itself has not changed.

### FHIR interaction metadata rules

The current Swagger makes these rules explicit:

| Interaction | `id` | `meta.versionId` / `meta.lastUpdated` | `meta.profile` |
| --- | --- | --- | --- |
| Create | A supplied resource `id` is ignored and PCM assigns the authoritative id | Prohibited | Optional declaration; does not select validation |
| Update | Body `id` is required and must equal the URL `{id}` | Prohibited | Optional declaration; does not select validation |

Unsupported `meta` business/security fields are prohibited. Direct HealthcareService POST and PUT requests also prohibit `meta.tag`; PCM adds the authoritative classification tag to returned resources. The complete catalog resource inside `$request-catalog-update` is the documented exception and keeps its fixed `catalog` tag.

The outer `$request-catalog-update` Parameters metadata also prohibits `meta.versionId` and `meta.lastUpdated`. Its embedded complete `proposedCatalog` may retain both optional values copied from the retrieved canonical; PCM removes them before semantic comparison, and neither provides optimistic concurrency.

Clients may include the appropriate request-profile canonical in `meta.profile`, and current examples do. PCM validates the actual interaction and body shape independently; a correct `meta.profile` cannot make a non-conforming body valid.

### Replacement semantics and concurrency

For the writable projection of an update:

- omitted optional writable fields are removed;
- repeating writable fields replace the complete stored collection;
- writes are not patches;
- accepted updates are last-write-wins only after full validation;
- `If-Match` is not supported;
- PATCH is not supported;
- update-as-create is not supported.

Clients must read or retain the complete desired writable projection before replacing a repeating collection such as Organization contacts.

### Successful-response headers were added

**Classification: Contract hardening**

The current Swagger adds:

- `Location` on successful `201` creates;
- `Content-Location` on successful `201` creates;
- `Last-Modified` on successful instance `GET`;
- `Last-Modified` on successful `HEAD`;
- `Last-Modified` on successful `PUT`.

`Location` and `Content-Location` are absolute URIs. `Last-Modified` uses HTTP-date format.

No ETag or `If-Match` contract was added. Do not infer optimistic versioned updates from `Last-Modified`.

### Error model expanded

FHIR failures continue to use `application/fhir+json` OperationOutcome bodies when handled by the FHIR API.

| HTTP status | Current intended category | Typical issue code |
| ---: | --- | --- |
| `400` | Malformed FHIR, prohibited field, or closed-schema violation | `invalid` |
| `401` | Missing or invalid FHIR bearer authentication | `login` or `security` |
| `403` | Wrong role, ownership, or operation authorization | `forbidden` |
| `404` | Unknown or deliberately hidden target | `not-found` |
| `409` | Duplicate or current-state transition conflict | `conflict` |
| `422` | Well-formed request that violates a business rule | `business-rule` |
| `default` | Server failure | OperationOutcome error |

Newly declared response codes include:

- `409` and `422` on Organization create/update;
- `409` and `422` on Endpoint create/update;
- `409` and `422` on HealthcareService create and instance update;
- `422` on Consent create;
- `409` and `422` on Consent deactivation;
- `422` on the catalog-update operation.

The old `202` catalog-update response on HealthcareService PUT was removed.

Privacy-aware use of `403` versus `404` must not be treated as a record-existence oracle.

Successful HEAD responses remain bodyless and now expose `Last-Modified`. The declared HEAD `401`, `403`, and `404` responses are also bodyless rather than OperationOutcome responses. TLS negotiation failures and OAuth-endpoint errors likewise retain their native protocol handling; only failures handled by the FHIR application use the FHIR OperationOutcome contract.

## 10. FHIR search changes

### Form-encoded `POST _search` is new

**Classification: New capability**

All four resource types now support:

```text
POST /Organization/_search
POST /Endpoint/_search
POST /HealthcareService/_search
POST /Consent/_search
Content-Type: application/x-www-form-urlencoded
```

The body is optional. Each operation is equivalent to the corresponding GET search with respect to:

- supported filters;
- authorization ceiling;
- result schema;
- visibility behavior;
- error behavior.

This does not deprecate GET search.

Repeated `_include` and `_include:iterate` values use form-style exploded encoding. Client generators must preserve the colon in `_include:iterate` and the hyphens in parameters such as `based-on` and `pcm-business-status`.

### Search parameters added to Swagger

| Resource | Parameters newly declared relative to the reference Swagger |
| --- | --- |
| Organization | `_lastUpdated` |
| Endpoint | `_lastUpdated` |
| HealthcareService | `_tag`, `based-on`, `pcm-business-status`, `_lastUpdated` |
| Consent | `pcm-business-status`, `_lastUpdated` |

The existing names `service-category`, `service-type`, `organization`, and `patient:identifier` were already present in the reference Swagger and remain the correct names.

### Include values are now typed

The old generic string `_include` schemas were replaced with resource-specific allowed arrays:

| Search | Parameter | Allowed values |
| --- | --- | --- |
| Organization | `_include` | `Organization:endpoint`, `Organization:partof` |
| Organization | `_include:iterate` | `Organization:endpoint`, `Organization:partof` |
| Consent | `_include` | `Consent:actor` |
| Consent | `_include:iterate` | `Organization:endpoint`, `Organization:partof` |

### Search responses are resource-specific

**Classification: Contract hardening**

| Search | Reference success schema | Current success schema |
| --- | --- | --- |
| Organization | Generic `Bundle` | `PCMOrganizationSearchBundle` |
| Endpoint | Generic `Bundle` | `PCMEndpointSearchBundle` |
| HealthcareService | Generic `Bundle` | `PCMHealthcareServiceSearchBundle` |
| Consent | Generic `Bundle` | `PCMConsentSearchBundle` |

Current search Bundles:

- are `type=searchset`;
- carry the applicable search-Bundle canonical in `meta.profile`;
- model `entry.fullUrl`;
- constrain the permitted resource types in each Bundle;
- permit OperationOutcome search entries where relevant;
- permit authorized included Organization and Endpoint resources in the applicable Organization or Consent Bundle.

### Authorization is a hard ceiling

**Classification: Clarification with security impact**

For every search, a primary result must:

1. satisfy all accepted filters; and
2. be authorized for the caller to read.

Combining filters only narrows the authorized set. `_include` and `_include:iterate` add only related resources that are independently visible to the caller. They never widen access.

An unsupported security-relevant parameter must not be interpreted as permission to return a broader set. Clients must not infer that a hidden resource does or does not exist from a search or include result.

## 11. Organization API changes

### `POST /Organization` now creates an authorized Service Provider child

**Classification: New capability with a closed contract**

This operation is available only after PCM separately authorizes Service Provider expansion.

The closed `PCMOrganizationServiceProviderCreateRequest` requires:

- `resourceType=Organization`;
- non-empty `name`.

It may contain:

- create interaction metadata;
- purpose-coded contact overrides;
- unknown ordinary non-modifier extensions.

It must not contain or attempt to choose:

- Organization logical or business identity;
- `active`;
- Organization role/type;
- Parent linkage;
- Endpoint linkage;
- OAuth redirect URI projection;
- certificate inventory;
- another PCM-managed field.

PCM derives the Parent and Service Provider role, assigns logical and business identifiers and state, and returns a complete `PCMOrganizationServiceProvider`.

The operation:

- cannot create a Parent;
- cannot create a Data Source;
- cannot claim another Parent;
- cannot self-assign another Organization role;
- does not allocate an OAuth client;
- does not register a certificate.

The `201` response includes `Location` and `Content-Location`. Specialized OperationOutcome examples distinguish unsupported child creation from wrong-role and cross-Parent attempts.

#### Why this changed

The reference Swagger had no Organization create interaction, forcing Service Provider expansion outside the declared FHIR surface. The new operation exposes only the safe participant-owned input while keeping regulated identity, role, state, and security registration under PCM control.

### Organization update is now type-specific and closed

**Classification: Breaking**

`PUT /Organization/{id}` now accepts `PCMOrganizationUpdateRequest`, an `anyOf` union:

| Target | Required writable body | Behavior |
| --- | --- | --- |
| Parent | `resourceType`, matching `id`, unchanged authoritative `name`; optional complete contacts | Replaces Parent-managed contact collection |
| Service Provider child | `resourceType`, matching `id`, unchanged authoritative `name`; optional complete contact overrides | Replaces child contact overrides |
| Data Source child | `resourceType`, matching `id`, unchanged authoritative `name`, exactly one Endpoint reference; optional complete contact overrides | Replaces the operational Endpoint link and contact overrides |

The repeated `name` is required by the FHIR Organization base shape. It is not a rename command. A mismatch is rejected.

Contact objects require `purpose`; `name`, `telecom`, and `address` are optional. Omission of an optional writable collection clears it.

The reference operation description allowed updates to applicable certificate thumbprints and OAuth redirect URI depending on Organization type. The current request schemas explicitly prohibit both:

```text
http://pcm.fhir.health.gov.il/StructureDefinition/ext-applicable-certificates
http://pcm.fhir.health.gov.il/StructureDefinition/ext-oauth2-redirect-uri
```

The following are also prohibited in participant updates:

- `identifier`;
- `active`;
- Organization role/type;
- Parent linkage;
- other identity/classification fields;
- security registration projections.

### Organization state and Data Source availability are separated

**Classification: Breaking clarification**

MoH/PCM exclusively controls every Parent and child `Organization.active` value. A participant must not attempt to deactivate or reactivate an Organization through FHIR.

An inactive Parent prevents operational use of its children without rewriting the child `active` values. Parent reactivation does not automatically override an independent child block.

A Data Source controls technical availability through its currently referenced Endpoint’s `status`, not through `Organization.active`.

### Authoritative Organization responses are now typed

**Classification: Contract hardening**

Reads and update responses use `PCMOrganization`, an `anyOf` union of:

- `PCMOrganizationParent`;
- `PCMOrganizationServiceProvider`;
- `PCMOrganizationDataSource`;
- `PCMOrganizationSystem`.

Returned participant Organizations now expose the stable official business identifier:

```text
http://pcm.fhir.health.gov.il/identifier/pcm-organization-id
```

Keep that business identifier separate from:

- the FHIR logical `id`;
- a Parent MoH institution or paying-entity identifier;
- an OAuth `client_id`;
- a certificate thumbprint;
- an Endpoint address.

Child Organizations have their own PCM Organization business identifier. They do not copy the Parent’s MoH identifier.

Returned resources require authoritative `id`, `meta.lastUpdated`, and applicable `meta.profile`. Service Provider redirect URI and applicable-certificate extensions are optional read-only projections.

An inactive Data Source may temporarily have no Endpoint during staged onboarding. An active Data Source must reference exactly one Endpoint. The Data Source update interaction itself requires exactly one Endpoint because that operation establishes or replaces the link.

### Contact coding namespace and purpose changed

**Classification: Breaking serialized URI change plus additive code**

Organization contact-purpose coding moved from:

```text
http://fhir.health.gov.il/cs/pcm-contactentity-type
```

to:

```text
http://pcm.fhir.health.gov.il/cs/pcm-contactentity-type
```

The examples also introduce the `OPS` purpose for FHIR/API technical operations. Data Sources should provide an `OPS` contact; Service Providers should provide one when their application operations contact differs from the Parent fallback.

#### Organization migration checklist

- [ ] Add `POST /Organization` only for the authorized Service Provider-child scenario.
- [ ] Generate a dedicated Service Provider create DTO.
- [ ] Split Parent, Service Provider, and Data Source update DTOs.
- [ ] Require body `id` to match the URL on updates.
- [ ] Repeat the unchanged authoritative `name`.
- [ ] Send the complete desired writable contact collection.
- [ ] Remove `active`, identity, type, `partOf`, redirect URI, and certificate inventory from writes.
- [ ] Treat child contacts as purpose-based overrides and send the complete desired override collection.
- [ ] Persist the PCM Organization business identifier separately from logical id.
- [ ] Update the contact-purpose coding-system URI and accept `OPS`.

## 12. Endpoint API changes

### Create now has a closed request

**Classification: Breaking and contract hardening**

`POST /Endpoint` now requires `PCMEndpointCreateRequest`:

| Field | Rule |
| --- | --- |
| `resourceType` | Exactly `Endpoint` |
| `status` | `active` or `suspended`; creation as `off` is invalid |
| `connectionType` | Exactly `http://terminology.hl7.org/CodeSystem/endpoint-connection-type\|hl7-fhir-rest` |
| `payloadType` | Exactly one fixed payload concept described below |
| `address` | Complete absolute HTTPS FHIR base URI |

PCM derives `managingOrganization` from the authenticated Data Source and assigns identity. Client-supplied ownership, identity, and certificate inventory are prohibited.

### Fixed payload coding changed

**Classification: Breaking**

Replace the old IHE XPHR coding:

```text
urn:oid:1.3.6.1.4.1.19376.1.2.3|urn:ihe:pcc:xphr:2007
```

with:

```text
http://terminology.hl7.org/CodeSystem/endpoint-payload-type|any
```

The current fixed CodeableConcept has exactly one coding. Its `display` is prohibited. `any` means a general FHIR REST Endpoint; it does not claim that the server supports every FHIR resource or interaction and does not widen authorization.

### Update now has an explicit lifecycle

**Classification: Breaking**

`PUT /Endpoint/{id}` uses `PCMEndpointUpdateRequest` and requires:

- `resourceType=Endpoint`;
- body `id` matching the URL;
- `status`;
- fixed FHIR REST `connectionType`;
- fixed payload type;
- complete desired `address`.

Allowed participant states and transitions:

| Current state | Requested state | Result |
| --- | --- | --- |
| `active` | `active` | Idempotent success |
| `active` | `suspended` | Temporarily stop technical use |
| `suspended` | `suspended` | Idempotent success |
| `suspended` | `active` | Resume after current Organization/policy checks pass |
| `active` or `suspended` | `off` | Permanent retirement, only after the Endpoint is no longer referenced |
| `off` | `off` | Idempotent success |
| `off` | `active` or `suspended` | `409` conflict; `off` is terminal |

The base Endpoint schema now quotes `"off"` explicitly. This preserves the intended FHIR string value in YAML tooling that might otherwise interpret unquoted `off` as a boolean.

### Address change and replacement rules

An address can change in place only while the current Endpoint is `suspended`. Once accepted:

- the old address stops being a valid resource audience immediately;
- tokens targeting the old address do not authorize the new Endpoint;
- active introspection for the old audience fails.

To replace Endpoint identity:

1. create and validate the replacement Endpoint;
2. update the Data Source Organization to reference the replacement;
3. retire the now-unreferenced old Endpoint as `off`.

A currently referenced Endpoint cannot be turned `off`.

Only the Data Source’s currently referenced Endpoint with `status=active` is eligible for Data Source-access and presence-token issuance and active introspection. Suspension or retirement does not itself change Consent lifecycle and does not remove operational reporting obligations.

### Thumbprint search is constrained

The `thumbprint` query parameter now has the exact pattern:

```text
^[A-Za-z0-9_-]{43}$
```

It searches PCM’s read-only associated-certificate inventory for authorized visible Endpoints. A match does not:

- register a certificate;
- authorize it;
- prove it is presented on the current connection;
- describe the complete trust store;
- identify its role in a particular exchange;
- establish access-token binding;
- widen the caller’s visible Endpoint set.

### Response schema changed

Endpoint reads, creates, and updates now return `PCMEndpoint`, not the generic `Endpoint` component. The authoritative response requires:

- logical `id`;
- response `meta` with `lastUpdated` and applicable profile;
- status;
- fixed connection type;
- fixed payload type;
- address;
- literal `managingOrganization`;
- optional PCM-managed certificate inventory.

Participant Endpoint responses accept `active`, `suspended`, and `off`; PCM-system Endpoints may use other base FHIR Endpoint states.

#### Why these changes were made

The reference generic Endpoint body mixed participant-controlled availability and address with PCM-controlled identity, ownership, and certificate inventory. The explicit lifecycle lets a Data Source suspend safely before changing an audience, replace an Endpoint without an authorization gap, and retire an old identity permanently without changing regulated Organization state.

#### Endpoint migration checklist

- [ ] Replace the generic Endpoint write model with separate create and update DTOs.
- [ ] Replace the XPHR payload coding with `endpoint-payload-type|any`.
- [ ] Remove `managingOrganization` and certificate extensions from requests.
- [ ] Support `suspended` before an address change.
- [ ] Treat `off` as a terminal string state.
- [ ] Implement replacement-before-retirement ordering.
- [ ] Recalculate token audience immediately after address or Organization linkage change.
- [ ] Add `409` and `422` handling.
- [ ] Treat thumbprint search as inventory lookup only.

## 13. HealthcareService API changes

### Business status is now authoritative

**Classification: Breaking**

Catalog and provider-instance responses now contain exactly one:

```text
extension.url =
http://pcm.fhir.health.gov.il/StructureDefinition/ext-service-business-status
```

with:

```text
valueCoding.system =
http://pcm.fhir.health.gov.il/cs/pcm-service-business-status
```

Allowed response codes are:

| Business status | `HealthcareService.active` | Meaning |
| --- | ---: | --- |
| `pending` | `false` | Under review; not usable |
| `active` | `true` | Approved and potentially usable |
| `rejected` | `false` | Rejected |
| `frozen` | `false` | Temporarily blocked by PCM/MoH |
| `suspended` | `false` | Reversibly stopped by the owning provider; instance only |
| `cancelled` | `false` | Terminal cancellation |

`active=true` if and only if business status is `active`. Clients must no longer treat `active` as an independently writable state selector.

### Catalog and instance response types are explicit

`PCMHealthcareService` is a union of:

- `PCMHealthcareServiceCatalog`;
- `PCMHealthcareServiceInstance`.

Catalog responses:

- use the `catalog` HealthcareService classification tag;
- use the catalog identifier system;
- have no `providedBy`;
- have no `basedOn`;
- cannot use provider-only `suspended`.

Provider-instance responses:

- use the `instance` HealthcareService classification tag;
- use the instance identifier system;
- require `providedBy`;
- require exactly one `basedOn` reference to the catalog;
- may carry the PCM-managed approval window.

The business identifier systems are:

```text
http://pcm.fhir.health.gov.il/identifier/pcm-healthcareservice-catalog-id
http://pcm.fhir.health.gov.il/identifier/pcm-healthcareservice-instance-id
```

The logical id, business identifier, owner, linkage, lifecycle, approval window, and inherited definition are authoritative response content.

### Create remains one endpoint but now has two closed request shapes

**Classification: Breaking and contract hardening**

Every direct `POST /HealthcareService` request must omit `meta.tag`. PCM adds the authoritative tag to each returned resource: `instance` on the provider registration and, for a new-service request, `catalog` on the linked catalog resource. A client-supplied tag is an ordinary prohibited field and returns `400` with the validation OperationOutcome.

PCM selects the create path from the body, not from a tag. A valid `basedOn` selects an existing-catalog registration. A complete writable definition without `basedOn` selects a new-service request. The nested `proposedCatalog` resource used by `$request-catalog-update` is the documented exception: because it is a complete catalog representation for comparison, it retains the fixed `catalog` classification tag.

The interaction profile canonical may appear in `meta.profile`, but it is optional and does not determine which shape PCM validates.

#### Existing-catalog registration

Use the `PCMHealthcareServiceExistingRegistrationRequest` shape:

- `resourceType=HealthcareService`;
- exactly one `basedOn` extension referring to an existing active, registrable catalog service;
- optional allowed interaction metadata and ordinary unknown extensions.

`meta.tag` and every server-managed field are prohibited.

Do not repeat:

- name;
- category or type;
- description;
- access mode or duration;
- information buckets;
- example queries;
- patient-bucket setting;
- `active` or business status;
- owner;
- identifiers;
- approval window.

PCM returns a complete pending provider instance. A provider/catalog pair cannot have another non-cancelled registration; a duplicate returns `409` and does not replace or reactivate the existing registration.

#### New-service proposal

Use `PCMHealthcareServiceNewServiceRequest` when no suitable active catalog exists. The request requires:

- `resourceType=HealthcareService`;
- unique non-empty `name`;
- at least one `category`;
- exactly one `type`, fixed to v3 ActCode `INFA`;
- non-empty patient-facing `extraDetails`;
- exactly one `accessMode`;
- exactly one `accessDuration`;
- at least one information bucket;
- at least one example query;
- exactly one explicit `allowPatientBucketChange`, even when `false`.

It omits:

- `meta.tag`;
- `basedOn`;
- identifiers;
- `providedBy`;
- `active`;
- business status;
- approval window.

PCM creates a coupled pending catalog and pending provider instance and returns the provider instance. Neither is usable for Consent or token authorization while pending.

Duplicate registration and duplicate proposed-name conflicts are now explicitly modeled as `409 OperationOutcomeServiceConflict`.

### Information-bucket request constraints are now machine-readable

**Classification: Contract hardening**

The current Swagger requires:

- at least one information-bucket extension;
- exactly one IL HDP bucket coding inside each bucket extension;
- one required UCUM `historyDepth` inside each bucket;
- at most one `taggedDetails` inside a bucket;
- a unique bucket code across the HealthcareService;
- at least one example-query extension;
- one explicit Boolean `allowPatientBucketChange`.

Known protected HealthcareService extensions cannot be smuggled through the unknown-extension branch.

The fixed information-access `type` is:

```text
http://terminology.hl7.org/CodeSystem/v3-ActCode|INFA
```

### Provider-instance PUT changed from `active` to business status

**Classification: Breaking**

The reference `PUT /HealthcareService/{id}` accepted a minimal `active=false` or `active=true` intent and also carried catalog proposals. The current operation is only an owning-provider instance lifecycle interaction.

`PCMHealthcareServiceBusinessStatusUpdateRequest` contains:

- `resourceType=HealthcareService`;
- body `id` equal to the URL id;
- exactly one business-status extension;
- target code `active`, `suspended`, or `cancelled`;
- optional permitted interaction metadata and unknown ordinary extensions.

It prohibits:

- `active`;
- definition fields;
- identifiers;
- owner;
- `basedOn`;
- approval window;
- HealthcareService classification tag (`meta.tag`);
- PCM/MoH-only target states;
- another known HealthcareService extension.

Allowed owner transitions:

| Current status | Requested target | Effect |
| --- | --- | --- |
| `pending` | `cancelled` | Withdraw pending registration |
| `active` | `suspended` | Stop new use immediately and reversibly |
| `suspended` | `active` | Resume after every current gate is revalidated |
| `suspended` | `cancelled` | Permanent termination |
| An owner-controlled status | Same status | Idempotent success |

Rejected owner operations include:

- direct `active -> cancelled`; suspend first;
- choosing `pending`, `rejected`, or `frozen`;
- changing state while frozen;
- reactivating cancelled;
- targeting a catalog resource.

An accepted update preserves the same logical id and instance business identifier. It advances `meta.lastUpdated`, derives `active`, and preserves the PCM-managed approval window.

For a coupled pending new-service proposal, withdrawal cancels both the pending instance and linked pending catalog. Withdrawal of an existing-catalog registration cancels only that instance.

Suspension blocks new Consent and active token/introspection use. PCM may temporarily inactivate affected eligible Consents and restore them only after full revalidation. Withdrawal or termination is terminal and permanently inactivates affected Consents.

### Catalog updates moved to a dedicated FHIR operation

**Classification: Breaking and new capability**

Do not send canonical changes to:

```text
PUT /HealthcareService/{id}
```

Use:

```text
POST /HealthcareService/{catalog-id}/$request-catalog-update
Content-Type: application/fhir+json
```

The body is `PCMRequestCatalogUpdateParameters`, a closed FHIR `Parameters` resource containing:

1. exactly one `proposedCatalog` resource with the complete proposed canonical representation;
2. optionally one `reason` string.

The proposed HealthcareService `id` must equal the operation target.

Only these fields may differ semantically:

- `name`;
- `extraDetails`;
- `extension[exampleQuery]`.

A changed name must remain unique.

The outer Parameters request prohibits `meta.versionId` and `meta.lastUpdated`. Within the embedded `proposedCatalog`, the following differences are comparison-neutral:

- `meta.versionId`;
- `meta.lastUpdated`;
- coding display text;
- representation ordering;
- unknown ordinary non-modifier extensions, which are not interpreted or guaranteed to round-trip.

Unsupported modifier extensions reject the request.

Differences in these fields are material or unsupported and reject the operation:

- category or type;
- information buckets;
- history depth;
- tagged details;
- access mode;
- access duration;
- `allowPatientBucketChange`;
- identifiers or tags;
- lifecycle state;
- ownership or linkage;
- another protected business field.

Material changes require a new-service proposal.

#### Response semantics changed

The reference PUT returned:

```text
202 Accepted
OperationOutcome
```

The dedicated operation returns:

```text
200 OK
PCMRequestCatalogUpdateAcceptedParameters
```

The response includes:

- one non-empty `requestId`;
- status exactly `received`;
- optional `message`.

This response is a synchronous receipt, not:

- approval;
- application of the change;
- a mutation of the active catalog;
- a continuing review-status resource.

The active catalog remains unchanged until PCM/MoH actively applies an accepted proposal.

### HealthcareService search changed

New GET and POST-search filters are:

- `_tag`;
- `based-on`;
- `pcm-business-status`;
- `_lastUpdated`.

Use:

```text
_tag=http://pcm.fhir.health.gov.il/cs/pcm-meta-tag|catalog
```

for catalog discovery, and:

```text
_tag=http://pcm.fhir.health.gov.il/cs/pcm-meta-tag|instance
```

for provider-registration registry searches.

Use `based-on` to find registrations linked to a catalog, and `organization` to find registrations owned by a Service Provider.

Catalog search is active/registrable discovery. `active=false` is not a supported way to enumerate pending, frozen, rejected, or cancelled catalog records.

Visible instance search behavior is lifecycle-aware:

- active instances participate in authorized discovery;
- pending instances may be visible for registration/review;
- rejected, frozen, and suspended instances are visible only to the owning Service Provider;
- cancelled instances remain owner-visible for audit and idempotent retry recovery but are excluded from ordinary discovery.

Direct read can expose a pending or frozen catalog to an authorized caller that obtained its URL from a visible instance `basedOn`. Rejected or cancelled catalogs outside client visibility return not found.

#### Why these changes were made

The reference PUT overloaded provider state changes and asynchronous catalog proposals, while `active` alone could not explain pending, frozen, suspended, rejected, or terminal states. The new model separates shared catalog definition, provider registration, owner lifecycle, and catalog review. Each write now has one purpose and an unambiguous response.

#### HealthcareService migration checklist

- [ ] Add parsing for the service business-status extension.
- [ ] Enforce `active=true` only with business status `active`.
- [ ] Keep catalog and instance logical/business identities separate.
- [ ] Omit `meta.tag` from direct HealthcareService create/update DTOs and parse the classification tag on returned resources.
- [ ] Build distinct existing-registration and new-service request DTOs.
- [ ] Remove server-managed fields from both create shapes.
- [ ] Implement `409` duplicate recovery rather than blind resubmission.
- [ ] Replace `active` PUTs with the owner business-status request.
- [ ] Implement the exact transition matrix and idempotent same-state handling.
- [ ] Move all eligible catalog wording/query proposals to `$request-catalog-update`.
- [ ] Treat `status=received` as a receipt only.
- [ ] Add `_tag`, `based-on`, `pcm-business-status`, and `_lastUpdated` searches.

## 14. Consent API changes

### Create request is now closed and requires legal policy

**Classification: Breaking**

`POST /Consent` now accepts `PCMConsentCreateRequest`, requiring:

| Field | Rule |
| --- | --- |
| `resourceType` | Exactly `Consent` |
| `status` | Exactly `proposed` |
| `scope` | Exactly `http://terminology.hl7.org/CodeSystem/consentscope\|patient-privacy` |
| `category` | Exactly one v3 ActCode `INFA` category |
| `patient` | Identifier-only logical reference |
| `policy` | Exactly one fixed legal policy |
| `extension[pcmService]` | Exactly one literal reference to a provider HealthcareService instance |

The exact legal policy is:

```json
{
  "authority": "https://www.gov.il/",
  "uri": "https://pcm.fhir.health.gov.il/consent-policy/medical-information-mobility-law-2024"
}
```

`policyRule` is not permitted.

The fixed policy is required in both create and deactivation requests. This satisfies the inherited FHIR R4 `ppc-1` requirement that a Consent contain `policy` or `policyRule` while preventing a caller from selecting the governing policy.

The create body must not contain:

- PCM Consent business identifier;
- creation timestamp;
- requestor actor;
- approved Data Source actors;
- server-managed access-mode category;
- provision or authorization period;
- Consent business status;
- another server-managed Consent field.

The `pcmService` reference must identify the caller’s active, usable provider-instance HealthcareService. A catalog reference, another provider’s instance, or a pending, rejected, frozen, suspended, cancelled, or otherwise blocked service is rejected. PCM also revalidates the linked catalog and relevant Organization state.

POST is not idempotent. If the client cannot determine whether create succeeded, search the caller-owned Consents by patient, service, and status before retrying. A blind retry can create a second Consent.

### Patient identifier validation is stricter

**Classification: Contract hardening**

`Consent.patient` remains identifier-only; `patient.reference` is not used.

The current Swagger distinguishes:

| Identifier type | Rule |
| --- | --- |
| Israeli national id | System exactly `http://fhir.health.gov.il/identifier/il-national-id`; value exactly nine digits and must pass the Israeli check-digit algorithm |
| Synthetic test id | System exactly `http://fhir.health.gov.il/identifier/il-hdp-test-id`; value exactly `"1"` or `"2"` |
| Passport | System must be a URI in the IL passport URI ValueSet; value must be non-empty with no leading or trailing whitespace |

Eligible synthetic-test Consents carry the `HTEST` security label in the authoritative response. Non-test Consents must not carry `HTEST`. The client does not submit that security label.

### Authoritative Consent response is much more specific

**Classification: Breaking response model**

Reads, creates, and updates now return `PCMConsent`, requiring:

- logical `id`;
- applicable response profile and `meta.lastUpdated`;
- exactly one official PCM Consent business identifier;
- `dateTime`;
- fixed legal policy;
- exactly two categories: fixed `INFA` plus the server-managed access mode;
- identifier-only patient;
- exactly one provider-instance `pcmService`;
- one server-managed business status;
- one permit provision;
- exactly one requestor actor;
- lifecycle-appropriate period and Data Source actors.

The business identifier system is:

```text
http://pcm.fhir.health.gov.il/identifier/pcm-consent-id
```

The new business-status extension uses:

```text
http://pcm.fhir.health.gov.il/StructureDefinition/ext-consent-business-status
```

and:

```text
http://pcm.fhir.health.gov.il/cs/pcm-consent-business-status
```

Lifecycle mapping:

| FHIR `Consent.status` | PCM business status | Meaning |
| --- | --- | --- |
| `proposed` | `requested` | Awaiting a decision |
| `active` | `active` | Approved and potentially authorizing |
| `rejected` | `rejected` | Rejected |
| `inactive` | `temporarily-inactive` | Reversibly blocked |
| `inactive` | `permanently-inactive` | Terminal end other than explicit provider cancellation |
| `inactive` | `service-provider-cancelled` | Terminal result of owner deactivation |
| `entered-in-error` | `permanently-inactive` | PCM correction; never a provider-selected target |

The generic Consent schema now also permits `entered-in-error`; response parsers that treated the reference four-value enum as exhaustive must be updated.

An active Consent requires a period with both start and end. Proposed and rejected Consents have no period and no approved Data Source actors. An active Consent can still have no approved Data Source actor, so `status=active` alone does not prove a usable Data Source exists.

The selected service supplies the initial requested/default access mode and maximum duration. An authorized patient decision, including an assisted or delegated flow outside the Service Provider API, may narrow continuous access to one-time and/or approve a shorter period or later shorten it. The authoritative returned Consent carries the requested mode while proposed and the effective approved mode and period after approval; PCM never widens the mode or extends the period beyond the service limits.

The logical id and business identifier remain stable across lifecycle changes.

### Public PUT is now owner deactivation only

**Classification: Breaking**

The reference description included PCM portal approval, rejection, revocation, and Service Provider deactivation in one broadly described update. The current public FHIR operation supports only deactivation by the Service Provider that created the Consent.

`PCMConsentDeactivateRequest` requires:

- `resourceType=Consent`;
- body `id` matching the URL;
- `status=inactive`;
- the same fixed `patient-privacy` scope;
- exactly one fixed `INFA` category;
- the same exact legal policy.

It omits and prohibits:

- patient;
- identifiers;
- service reference;
- requestor and Data Source actors;
- period;
- access-mode category;
- business status;
- another Consent business field.

PCM preserves the authoritative content and returns the same Consent identity with business status `service-provider-cancelled`.

The result is terminal when the deactivation request is accepted. PCM validates whether the current Consent is eligible for the transition. Patient approval, rejection, revocation, delegation, assisted flow, and reactivation are outside this public FHIR update.

### Consent search changed

New parameters:

- `pcm-business-status`;
- `_lastUpdated`.

Existing parameters remain:

- `_id`;
- `identifier`;
- `status`;
- `patient`;
- `patient:identifier=system|value`;
- `pcm-service`;
- supported `_include` and `_include:iterate`.

The current search response is `PCMConsentSearchBundle`, which may contain authorized Consent, included Organization, included Endpoint, and OperationOutcome entries.

#### Why these changes were made

The generic Consent body allowed callers to send content that PCM derives or owns and did not satisfy the inherited R4 policy invariant with an exact caller-safe value. The closed create/deactivation projections preserve a minimal client intent, the fixed policy satisfies the base invariant without allowing policy selection, and the business status separates detailed authorization lifecycle from the coarse base FHIR status.

#### Consent migration checklist

- [ ] Replace the generic create body with `PCMConsentCreateRequest`.
- [ ] Add exactly one fixed policy to create and deactivation.
- [ ] Remove `policyRule`.
- [ ] Remove identity, actor, access-mode, period, business-status, and other server-managed content from writes.
- [ ] Validate national id, test id, and passport identifier rules before submission.
- [ ] Require an active provider-instance service reference.
- [ ] Parse and persist Consent business status.
- [ ] Accept `entered-in-error` in authoritative responses.
- [ ] Replace broad status update logic with owner deactivation only.
- [ ] Add `409` and `422` handling.
- [ ] Recover uncertain POST results through search before retrying.

## 15. Exact coding-system URI changes visible in Swagger

**Classification: Breaking**

The following serialized coding systems changed:

| Use | Reference URI | Current URI |
| --- | --- | --- |
| PCM Organization role/type | `http://fhir.health.gov.il/cs/pcm-org-type` | `http://pcm.fhir.health.gov.il/cs/pcm-org-type` |
| PCM Organization contact purpose | `http://fhir.health.gov.il/cs/pcm-contactentity-type` | `http://pcm.fhir.health.gov.il/cs/pcm-contactentity-type` |
| PCM Consent access mode | `http://fhir.health.gov.il/cs/pcm-consent-access-mode` | `http://pcm.fhir.health.gov.il/cs/pcm-consent-access-mode` |

These are exact identifier changes. Update:

- serialized FHIR coding systems;
- token search values;
- equality comparisons;
- validation allowlists;
- terminology caches;
- database indexes keyed by `system|code`;
- test fixtures.

Do not “normalize” both old and new URIs as equivalent unless an explicit compatibility policy outside this Swagger tells you to do so. The current contract uses the PCM-owned values.

The change places PCM-specific terminology in the PCM-owned namespace instead of the broader national FHIR namespace. Because coding-system identity is defined by the complete URI, this organizational rationale does not make the old and new serialized values interchangeable.

## 16. Response schemas and generated-client impact

### Key component replacements

| Reference component use | Current component use |
| --- | --- |
| Generic `Organization` success bodies | `PCMOrganization` or `PCMOrganizationServiceProvider` |
| Generic `Endpoint` success bodies | `PCMEndpoint` |
| Generic `HealthcareService` success bodies | `PCMHealthcareService` or `PCMHealthcareServiceInstance` |
| Generic `Consent` success bodies | `PCMConsent` |
| Generic `Bundle` search bodies | One of four `PCM*SearchBundle` components |
| Generic `TokenRequest` object | Five-schema `oneOf` |
| Generic `IntrospectionRequest` object | Two-schema `oneOf` |
| Generic `IntrospectionResponse` object | Active/inactive union with three active variants |
| Inline OAuth error object | Reusable closed `OAuthErrorBody` and endpoint-specific responses |

The FHIR schema inventory increased from 36 components to 150, and OAuth from 6 to 37. This increase mainly represents contract precision: request/response separation, lifecycle variants, search forms, typed FHIR context, and exact extension/value constraints.

### OpenAPI 3.1 features are significant

The current schemas use:

- `const`;
- `oneOf`, `anyOf`, and `allOf`;
- `contains`, `minContains`, and `maxContains`;
- `not`;
- `additionalProperties: false`;
- Boolean property schemas such as `property: false`;
- conditional `if` / `then` / `else`;
- `contentMediaType` and `contentSchema` for decoded JWT guidance;
- custom descriptive extensions such as decoded assertion examples.

Some generators flatten unions or ignore array `contains` constraints incorrectly. Before regenerating production clients:

1. verify that the generator supports OpenAPI 3.1 rather than treating it as OpenAPI 3.0;
2. inspect generated union discriminators or wrapper types;
3. verify that closed request DTOs do not inherit complete response fields;
4. verify that form property names retain `:`, `-`, and `#`;
5. retain server-side negative tests even if a generator cannot express every JSON Schema constraint.

The runtime request contract remains authoritative; successful code generation is not evidence that a payload is accepted.

### FHIR literal references are tighter

The new `PCMLiteralReference` requires non-empty `reference` and prohibits identifier-only representation. It is used for important relationships such as:

- Organization `partOf`;
- Data Source `endpoint`;
- Endpoint `managingOrganization`;
- HealthcareService `providedBy`;
- HealthcareService `basedOn`;
- Consent actors;
- Consent `pcmService`.

`Consent.patient` is the deliberate identifier-only exception.

### Authoritative metadata is stronger

Current complete responses commonly require:

- logical `id`;
- `meta.lastUpdated`;
- applicable response canonical in `meta.profile`;
- stable business identifier where defined;
- server-managed lifecycle state.

Persist logical id and business identifier separately. Use logical id in REST URLs and literal references; use the named identifier system/value for durable reconciliation.

## 17. Swagger example changes

The FHIR Swagger example inventory increased from 33 to 68. Examples are now organized around request/response separation, lifecycle transitions, typed searches, and negative outcomes.

### Important replacements

| Removed reference example | Current replacement pattern |
| --- | --- |
| `OrganizationSearchBundle` | `OrganizationSourceSearchWithParentBundle`, `OrganizationServiceProviderByParentSearchBundle` |
| `HealthcareServiceSearchBundle` | `HealthcareServiceActiveCatalogSearchBundle`, `HealthcareServiceInstanceRegistrySearchBundle`, `HealthcareServicePendingRegistrationSearchBundle` |
| `HealthcareServiceInstanceDeactivate` | `HealthcareServiceInstanceSuspend`, `...Resume`, `...Terminate`, `...Withdraw`, each with an authoritative response counterpart |
| `OperationOutcomeCatalogUpdateAccepted` | `ParametersCatalogUpdateRequest`, `ParametersCatalogUpdateAccepted` |

### Newly covered positive scenarios

Current examples include:

- Service Provider child Organization creation;
- Organization logical id equal to business-id value without treating the identities as the same namespace;
- long-duration Service Provider/service scenarios;
- Endpoint suspended address update and old Endpoint retirement;
- Endpoint thumbprint inventory search;
- active catalog discovery;
- instance registry and pending registration search;
- provider instance suspend/resume/terminate/withdraw request and response pairs;
- Consent search by patient and by service;
- provider Consent deactivation and service-provider-cancelled response;
- catalog-update request and receipt Parameters;
- standalone authorization code, PCM management, introspection-authentication, Data Source-access, online-presence, and batch-presence token requests/responses;
- typed active and inactive introspection responses.

### Newly covered negative scenarios

Current OperationOutcome examples cover:

- invalid based-on linkage;
- authorization failure;
- duplicate service registration;
- duplicate service name;
- invalid state transition;
- canonical update attempted through PUT;
- material catalog change;
- invalid Service Provider child creation;
- wrong-role and cross-Parent Organization creation;
- general validation failure.

#### How to use the examples

Use examples as test fixtures paired with their declared schema. Do not infer optionality or writability from example absence alone. The closed request schema and endpoint description are authoritative.

## 18. Detailed migration plan by implementer role

### All implementers

1. Configure environment-specific OAuth and FHIR operational bases.
2. Preserve canonical and identifier URIs exactly; do not derive them from the runtime host.
3. Present mTLS on discovery and metadata as well as operational calls.
4. Update exact coding-system URIs.
5. Upgrade OpenAPI tooling to preserve 3.1 union and closed-schema behavior.
6. Split write DTOs from authoritative response DTOs.
7. Parse OperationOutcome for all application-level FHIR failures.
8. Capture `Location`, `Content-Location`, and `Last-Modified`.
9. Add 409 conflict and 422 business-rule handling.
10. Add reconciliation through `_lastUpdated` where appropriate without assuming `If-Match`.

### OAuth/SMART client implementers

1. Fetch discovery with mTLS.
2. Remove JWKS fetching and local opaque-token JWT validation.
3. Switch assertion signing from ES384 to ES256.
4. Use the opaque assigned `client_id` in `iss=sub`.
5. Use exact environment endpoint audiences.
6. Generate a fresh assertion and unique `jti` for every request and retry.
7. Implement PKCE S256 and state validation for standalone SMART.
8. Request the minimum required space-delimited SMART v2 patient-level resource scopes, URL-encode constrained scope values, and do not use `user/...` or `_filter`.
9. Select exactly one token-request variant.
10. Replace `system/*.cruds` with `system/*.crus` for management and introspection-authentication tokens.
11. Send one exact RFC 8707 resource.
12. Add the complete B2B claim object for Data Source access.
13. Treat all access tokens as opaque 30-second Bearer tokens.

### Service Provider implementers

1. Store the Service Provider Organization business identifier for B2B `organization_id`.
2. Use authorized `POST /Organization` only for Service Provider child expansion.
3. Remove redirect and certificate writes from FHIR.
4. Register HealthcareService through one of the two closed create shapes.
5. Omit `meta.tag` from direct HealthcareService POST and PUT bodies; keep the fixed `catalog` tag only in the complete `proposedCatalog` resource for `$request-catalog-update`.
6. Implement provider-instance business-status transitions.
7. Use the catalog-update operation only for allowed non-material changes.
8. Include the fixed legal policy in Consent create and deactivation.
9. Search after an uncertain Consent POST before retrying.

### Data Source implementers

1. Control technical availability through Endpoint status, not Organization active.
2. Create/update Endpoint with fixed FHIR REST connection and `any` payload coding.
3. Suspend before changing address.
4. Update the Data Source Organization reference before retiring an old Endpoint.
5. Treat `off` as terminal.
6. Obtain an introspection-authentication token targeted exactly to `/introspect`, or use a direct assertion, but not both.
7. Replace the introspection parser with the four current variants.
8. Enforce `aud`, `scope`, typed context, expiry, and local resource authorization.
9. Stop using active results at `exp`; cache for no more than the remaining 30-second token lifetime.
10. Treat `active=false` as deliberately non-diagnostic.

### FHIR search/synchronization implementers

1. Add `_lastUpdated` for all four resource types where incremental reconciliation is required.
2. Use `POST _search` when form bodies are operationally preferable; keep GET compatibility.
3. Encode repeatable include values correctly.
4. Parse the resource-specific search Bundle profiles.
5. Preserve `entry.fullUrl`.
6. Apply authorization-aware logic to primary and included resources.
7. Use `_tag` to distinguish HealthcareService catalogs from instances.
8. Use `based-on` and `pcm-business-status` for lifecycle-aware registry recovery.

## 19. Suggested conformance tests

### Connectivity and discovery

| Test | Expected result |
| --- | --- |
| Fetch OAuth metadata without a client certificate | mTLS negotiation or access fails; anonymous discovery is not supported |
| Fetch SMART configuration with the assigned certificate | Metadata contains authorization-code and client-credentials capabilities |
| Look for `jwks_uri` or call the old JWKS route | No current contract support |
| Use the old production OAuth or FHIR base | Request targets the retired contract origin rather than the current production origin |
| Replace canonical `pcm.fhir.health.gov.il` identifiers with `pcm2m` | Payload/search validation fails because runtime hosts and identifiers are distinct |

### Authorization and token issuance

| Test | Expected result |
| --- | --- |
| Sign a client assertion with ES384 | `invalid_client` under the current contract |
| Put an Organization URL in assertion `iss/sub` | Rejected; opaque assigned client ID is required |
| Reuse a `jti` or assertion on retry | Rejected or treated as replay; retry requires a fresh assertion |
| Use token endpoint as direct introspection assertion audience | Rejected; exact introspection audience is required |
| Send both authorization-code and client-credentials fields | `invalid_request`; request matches no single union branch |
| Send a valid space-delimited SMART v2 patient-level app-launch scope set | Request is evaluated against the client, user, service definition, and authorization policy; it is not rejected merely for differing from the example |
| Send a `user/...` app-launch scope | Rejected as unsupported; PCM does not advertise `permission-user` |
| Send a query-constrained SMART scope using `_filter` | Rejected as an unsupported scope |
| Compare the client-facing `patient/*.rs` example with Data Source introspection | Data Source enforces the effective service-derived `scope` returned by active introspection, which may differ |
| Request management scope `system/*.cruds` | `invalid_scope` |
| Request management resource as an individual FHIR resource URL | `invalid_target` |
| Supply two `resource` form values | `invalid_target` |
| Use a suspended/replaced Endpoint address as Data Source target | Token not issued |
| Omit B2B `version`, policy, purpose, Organization id, or Consent URL | Token request rejected |
| Use bare `TREAT` instead of URI-form purpose | Request does not conform |
| Use a Consent business identifier as `consent_reference` | Rejected; an absolute Consent instance URL is required |
| Parse a success token as a JWT | Incorrect; token is opaque |
| Expect a refresh token or launch patient | Incorrect; neither is returned |

### Introspection

| Test | Expected result |
| --- | --- |
| Send bearer caller authentication and direct assertion fields together | `400 invalid_request` |
| Introspect a token for another Data Source audience | `200 {"active":false}` |
| Introspect after Endpoint suspension or address replacement | `200 {"active":false}` |
| Expect top-level `patient` or `cnf` | Fields are absent in the current variants |
| Parse active Data Source context without exactly Patient, Consent, and HealthcareService entries | Response does not satisfy the current schema |
| Treat online and batch presence responses identically | Incorrect; online has one Patient context and batch has none |
| Cache an active result after returned `exp` | Incorrect and unsafe |
| Use token issuance time to start one-time access | Incorrect; the source’s first qualifying active introspection starts its window |

### Cross-cutting FHIR writes

| Test | Expected result |
| --- | --- |
| Echo a complete returned resource into PUT | Rejected when server-managed content is present |
| Include prohibited content plus one valid change | Entire request rejected; no partial application |
| Omit request `meta.profile` but otherwise conform | May be accepted; profile declaration is optional |
| Declare the correct request profile with an invalid body | Rejected based on actual body/interaction |
| Supply an unsupported modifier extension | Rejected |
| Supply an unknown ordinary non-modifier extension | May be accepted without interpretation or round-trip guarantee |
| Supply an update body id different from URL id | Rejected |
| Supply `meta.versionId` or `meta.lastUpdated` in a direct write or outer catalog-update Parameters metadata | Rejected as prohibited request metadata; it does not create optimistic concurrency |
| Retain `meta.versionId` or `meta.lastUpdated` inside `proposedCatalog` | Accepted and removed before semantic comparison; neither provides optimistic concurrency |
| Send `If-Match` or PATCH | Not supported |

### Organization

| Test | Expected result |
| --- | --- |
| Create a Parent or Data Source through `POST /Organization` | Rejected |
| Create a Service Provider child without prior authorization | Rejected |
| Include `active`, type, Parent, redirect URI, or certificate inventory in create | Entire request rejected |
| Change Organization `active` through PUT | Rejected |
| Change authoritative `name` through PUT | Rejected |
| Omit an optional writable contact collection | Collection is cleared |
| Data Source update omits Endpoint | Rejected |
| Data Source links another organization’s Endpoint | Rejected |

### Endpoint

| Test | Expected result |
| --- | --- |
| Create Endpoint as `off` | Rejected |
| Use the old XPHR payload coding | Rejected |
| Include `managingOrganization` in create/update | Rejected |
| Change address while active | Rejected; suspend first |
| Reactivate `off` | `409` conflict |
| Turn a referenced Endpoint `off` | Rejected |
| Search a valid thumbprint outside caller visibility | Hidden; thumbprint cannot widen authorization |
| Treat thumbprint match as proof of presented certificate | Incorrect |

### HealthcareService

| Test | Expected result |
| --- | --- |
| Direct create includes any `meta.tag` | `400`; prohibited-field validation error |
| Catalog-update `proposedCatalog` omits or changes its fixed `catalog` tag | Rejected |
| Existing registration repeats definition fields | Rejected |
| New-service request omits `allowPatientBucketChange` | Rejected |
| New-service request repeats a bucket code | Rejected |
| Duplicate provider/catalog registration | `409` |
| Owner updates `active` instead of business status | Rejected |
| Owner requests `active -> cancelled` directly | `409`; suspend first |
| Owner changes a frozen instance | Rejected |
| Owner attempts canonical PUT | Rejected; use the dedicated operation when eligible |
| Catalog operation changes duration or bucket | `422`; material change requires new-service flow |
| Treat catalog-operation `received` as approval | Incorrect; active catalog remains unchanged |

### Consent

| Test | Expected result |
| --- | --- |
| Create without the fixed legal policy | Rejected |
| Supply `policyRule` instead | Rejected |
| Supply another authority or policy URI | Rejected |
| Include requestor, access mode, provision, identifier, or business status in create | Entire request rejected |
| Reference a catalog rather than an active provider instance | Rejected |
| Submit malformed Israeli national id | Rejected |
| Use test system with value other than `1` or `2` | Rejected |
| Submit HTEST security for a real patient | Rejected |
| Deactivation repeats patient/service/actors/period | Rejected |
| A non-owner deactivates Consent | `403` or privacy-aware `404` |
| Immediately retry an uncertain create | Unsafe; search first |

## 20. Intentionally unsupported or not defined in `0.3.2`

Implementers should not infer support for:

- public JWKS retrieval;
- JWT self-validation of PCM opaque access tokens;
- OAuth dynamic client registration;
- FHIR PATCH;
- FHIR delete;
- FHIR transaction or batch write;
- update-as-create;
- version-aware `If-Match` updates;
- participant writes to Organization active state;
- participant writes to OAuth redirect or certificate inventory through FHIR;
- participant writes to Endpoint ownership;
- client creation of a catalog HealthcareService;
- canonical HealthcareService change through PUT;
- provider selection of `pending`, `rejected`, or `frozen`;
- public patient approval, rejection, revocation, delegation, or assisted Consent through the client FHIR update;
- a continuing FHIR review-status resource for catalog-update receipts;
- a standard certificate-bound access-token `cnf`;
- a fixed public batch-presence size limit, oversized-batch response, or common timeout/retry contract.

## 21. Final cutover checklist

- [ ] Runtime production OAuth origin is `https://pcm2m.health.gov.il`.
- [ ] Runtime production FHIR base is `https://pcm2m.health.gov.il/r4`.
- [ ] Non-production endpoints are externally configured.
- [ ] Canonical, identifier, policy, role, and CodeSystem URIs remain exact published values.
- [ ] mTLS is active for discovery, authorization, metadata, token, introspection, and FHIR.
- [ ] JWKS dependencies are removed.
- [ ] Assertions use ES256, opaque client IDs, exact audiences, unique `jti`, and fresh JWTs.
- [ ] Token request variants are serialized independently.
- [ ] Management/introspection scope is `system/*.crus`.
- [ ] Data Source-access B2B claims include version, Organization identifier URI, URI-form treatment purpose, fixed policy, and absolute Consent URL.
- [ ] Token success parser requires `scope` and handles fixed 30-second Bearer tokens.
- [ ] Introspection parser supports all current active/inactive variants.
- [ ] FHIR write DTOs are separate from response DTOs.
- [ ] Body id matching and full-replacement semantics are implemented.
- [ ] `Location`, `Content-Location`, and `Last-Modified` are handled.
- [ ] GET and optional POST `_search` are supported as required.
- [ ] Search results and includes remain authorization filtered.
- [ ] Organization create/update projections are implemented without security/state fields.
- [ ] Endpoint payload coding and lifecycle are migrated.
- [ ] HealthcareService business status and catalog operation are migrated.
- [ ] Consent fixed policy, business status, patient validation, and owner-only deactivation are migrated.
- [ ] Three moved coding-system URIs are updated.
- [ ] Positive, negative, lifecycle, duplicate, authorization, and retry-recovery tests pass before cutover.

---

## Historical MoH-Swagger repository entries (superseded)

The entries below are retained as a backward-reference record of earlier repository snapshots. They describe the pre-`0.3.2` contract and are not current implementation requirements. Where they conflict with the `0.3.2` changelog above or the published OpenAPI YAML files, the current OpenAPI YAML files are authoritative.

### [Unreleased] - 2026-05-31 (superseded snapshot)

#### Added

- Added OAuth authorization server metadata discovery, including advertised token/introspection endpoints, supported `private_key_jwt` authentication methods, supported assertion signing algorithms, and JWKS location.
- Added a JWKS discovery endpoint for PCM issuer public keys.
- Added `private_key_jwt` client authentication support for `/introspect`, in addition to bearer-token caller authentication.
- Added explicit RFC 8707 resource-indicator behavior for token issuance, including one access token per resource server and `invalid_target` for requests with multiple resource targets.
- Added HealthcareService flows for registering against an existing catalog service and requesting a new service when no suitable catalog entry exists.
- Added pending catalog and pending provider-instance examples for new-service registration responses.
- Added OperationOutcome examples for catalog update requests that are accepted for asynchronous processing or rejected because they require a new-service registration.
- Added documented HEAD checks for resource existence across the supported FHIR resources.
- Added `patient:identifier` as a documented Consent search parameter.

#### Changed

- Clarified that mTLS is a transport-level requirement while OAuth2 client authentication uses `private_key_jwt`; mTLS certificate evidence may be linked through `cnf` policy but is not advertised as the OAuth client authentication method.
- Updated token and introspection examples to use ES384-style client assertions and to distinguish PCM management tokens from data-source access tokens.
- Clarified introspection response semantics, including HealthcareService instance context, data-source audience values, and certificate confirmation evidence.
- Reframed service governance around HealthcareService lifecycle state and OperationOutcome responses instead of public approval resources.
- Clarified HealthcareService search visibility for active and inactive catalog/instance services, including authorization limits for inactive records.
- Reworked HealthcareService creation so providers either submit a minimal instance registration with `basedOn` for an existing catalog service, or submit a full new-service request that creates pending catalog and instance resources.
- Clarified that external clients do not directly create immediately active canonical catalog services; activation and review happen outside the public FHIR operation.
- Clarified HealthcareService update behavior for instance deactivation, reactivation requests, asynchronous catalog updates, and material catalog changes that must go through the new-service flow.
- Clarified Consent creation rules so consent requests must reference an existing active HealthcareService instance, not a catalog service or inactive instance, and token issuance repeats the service-state check.
- Updated submit examples to show server-accepted request payloads without client-supplied `meta.profile` where profiles are illustrative rather than sent on the wire.
- Normalized the patient bucket-change extension URL to `ext-allow-patient-bucket-change`.
- Refreshed the CapabilityStatement, FHIR examples, and generated Redocly documentation to match the current public API behavior.

#### Removed

- Removed public VerificationResult approval endpoints and examples.

### [Unreleased] - 2026-01-28 (superseded snapshot)

#### Added

- Split OpenAPI specs into `openapi-fhir.yaml` and `openapi-oauth.yaml`, with a combined `openapi.yaml`.

#### Changed

- OpenAPI documentation refreshed.
