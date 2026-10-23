# Pension Scheme Return – External Dependencies

All outbound HTTP calls go through **`HttpClientV2`** from `bootstrap-backend-play-30`, enabled by `HttpClientV2Module`. Base URLs come from `ServicesConfig.baseUrl(<service>)`, which reads `microservice.services.<service>.{protocol,host,port}`. Each path is a `String.format` template under `serviceUrls.*` in `conf/application.conf`. `AppConfig` concatenates the base URL and the template **once, at start-up** (as `val`s). The connector then fills the `%s` placeholders and hands the **complete URL string** to the `url"$url"` interpolator as a single argument. Because of this, the values substituted into it are not encoded per segment (see `identifiers.md`).

Every outbound call has its timeout set to `ifs.timeout` (40 s), including the one to the scheme registry, which is not an IF endpoint. Responses are read with a pass-through `HttpReads[HttpResponse]`, and the status is mapped by hand in `HttpResponseHelper.handleErrorResponse`.

## Overview

| # | Dependency | Type | Config key | Accessed via |
|---|---|---|---|---|
| 1 | ETMP via Integration Framework (IF) – PSR APIs | HTTP/JSON, bearer token | `microservice.services.if-hod`, `serviceUrls.{get-overview,get-versions,submit-standard-psr,get-standard-psr}` | `PsrConnector` (called from `PsrOverviewService`, `PsrVersionsService`, `PsrSubmissionService`) |
| 2 | Pensions Scheme service – association check | HTTP/JSON, no service auth header | `microservice.services.pensionsScheme`, `serviceUrls.is-psa-associated` | `SchemeDetailsConnector` (from `PsrAuth`) |
| 3 | Auth service | HMRC auth client | `microservice.services.auth` | `DefaultAuthConnector` bound in `Module`; `PsrAuth` |
| 4 | Datastream / audit | HMRC audit connector | bootstrap `auditing.*` | `AuditServiceImpl`, `ApiAuditUtil`, `JsonErrorHandler` |
| 5 | Email service (inbound callback) | HTTP POST into this service | `queryParameter.encryption.key` | `EmailResponseController` |

---

## 1. ETMP via Integration Framework

- **Base URL**: `microservice.services.if-hod` (local default `localhost:10702`).
- **Headers** (`PsrConnector.integrationFrameworkHeader`), sent on every call:
  - `Environment`: `microservice.services.if-hod.env` (default `local`; `ist0` in application.conf)
  - `Authorization`: `Bearer ` + `microservice.services.if-hod.authorizationToken` (default `local`)
  - `Content-Type: application/json`
  - `CorrelationId`: a random UUID, cut to 36 characters, generated per request
- **Timeout**: `ifs.timeout`.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Get PSR overview | `GET /pension-online/reports/overview/pods/{pstr}/PSR?fromDate={from}&toDate={to}` | `PsrConnector.getOverview` | 200 → `Seq[PsrOverviewEtmpResponse]` (strict `.as`). 404 → empty list. Other codes → `handleErrorResponse`. |
| Get PSR versions | `GET /pension-online/reports/{pstr}/PSR/versions?startDate={date}` | `PsrConnector.getVersions` | 200 → `Seq[PsrVersionsEtmpResponse]`. **503 → empty list (warn)**. 404 → empty list. Other codes → `handleErrorResponse`. |
| Submit standard / pre-populated PSR | `POST /pension-online/scheme-return/{pstr}` | `PsrConnector.submitStandardPsr` | Body is `PsrSubmissionEtmpRequest` JSON, schema-checked **before** sending (API 1999 v1.2.0 / v1.1.5 / v1.2.0-optional). 200 → pass-through. Other codes → `handleErrorResponse`. A `PensionSchemeReturnPost` audit event is fired on every outcome. |
| Get standard PSR | `GET /pension-online/scheme-return/{pstr}?psrFormBundleNumber={fb}` **or** `?periodStartDate={d}&psrVersion={v}` | `PsrConnector.getStandardPsr` | `buildParams` throws `BadRequestException("Missing url parameters")` when neither combination is present. 200 → `PsrSubmissionEtmpResponse` (strict `.as`). **422 with body containing `PSR_NOT_FOUND` → `None`**. Other codes → `handleErrorResponse`. |

**Error mapping** (`HttpResponseHelper.handleErrorResponse`):

| Upstream status | Exception thrown | Caller sees (via `JsonErrorHandler`) |
|---|---|---|
| 400 | `BadRequestException` (logs a warning if the body contains `INVALID_PAYLOAD`) | 400. On submit it is turned into `ExpectationFailedException`, so the caller gets **417**. |
| 404 | `NotFoundException` | 404 |
| other 4xx | `UpstreamErrorResponse(status, reportAs = status)` | the same status |
| 5xx | `UpstreamErrorResponse(status, reportAs = status)` | the same 5xx |
| anything else (e.g. 3xx, 201) | `UnrecognisedHttpResponseException` | 500 |

The upstream response body is included in the exception message. `JsonErrorHandler` does not suppress `HttpException` messages, so the IF error body reaches the caller.

**Usage notes**

- Overview and versions apply `earliestPsrPeriodStartDate` (`2015-04-06`) in the controller before calling. Overview moves `fromDate` forward to that date. Versions-for-years drops earlier start dates and calls `getVersions` for each remaining date in parallel (`Future.sequence`).
- `getStandardPsr` calls `ApiAuditUtil.firePsrGetAuditEvent`, which **only logs**. No audit event is sent for reads.
- ⚠️ **Observation:** if the versions endpoint returns 503, the frontend gets an empty version list with status 200, which looks the same as "no versions".
- ⚠️ **Observation:** `PsrConnector` logs the full URL (including the PSTR) at info level, and logs the full ETMP response body at debug level in `getStandardPsr`. That response contains member personal data.

## 2. Pensions Scheme service – PSA/PSP association

- **Base URL**: `microservice.services.pensionsScheme` (local `localhost:8203`).
- **Headers**: `psaId: <id>` **or** `pspId: <id>` (the header name is the id type), `schemeReferenceNumber: <SRN>`, `Content-Type: application/json`. The caller's `HeaderCarrier` is forwarded as normal. No IF bearer token is sent.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Is the user associated with the scheme? | `GET /pensions-scheme/is-psa-associated` | `SchemeDetailsConnector.checkAssociation` | 200 → `response.json.as[Boolean]`. Other codes → `handleErrorResponse`. Called on **every** authorised request; there is no caching. |

Callers: `PsrAuth.checkPsa`, `checkPsp` and `checkPsaOrPsp`. A `false` result raises `UnauthorizedException`, which the caller sees as 401.

## 3. Auth

`DefaultAuthConnector` against `microservice.services.auth` (local `localhost:8500`).

| Guard | Requires | Retrieves | Then |
|---|---|---|---|
| `PsrAuth.authorisedAsPsrUser(srn)` | `srn` header matching `^S[0-9]{10}$` (otherwise 400). Enrolment `HMRC-PODS-ORG` **or** `HMRC-PODSPP-ORG`. | `externalId`, `allEnrolments` | Reads the `requestRole` header (case-insensitive): `PSA` uses the `PSAID` identifier of `HMRC-PODS-ORG`; `PSP` uses the `PSPID` identifier of `HMRC-PODSPP-ORG`; if the header is absent, PSA is preferred and PSP is the fallback; any other value gives 400. Then calls the association check. Passes a `PsrAuthContext(externalId, psaPspId, credentialRole)` to the action. |

- Used by: `PsrOverviewController`, `PsrVersionsController` and `PsrSubmitController`. That is every endpoint except the email callback.
- ⚠️ **Observation:** the `credentialRole` value depends on which path ran. `checkPsaOrPsp` sets `"PSA"`/`"PSP"`, but `checkPsa` and `checkPsp` set the constants `"psaId"`/`"pspId"`. `ExtendedAuditEvent.psaOrPspIdDetails` matches only on `"PSA"`. So when the caller sends `requestRole: PSA`, the submission audit event labels the PSA ID as `pensionSchemePractitionerId`.
- ⚠️ **Observation:** authorisation is checked against the **SRN header**, but data is read or written by the **PSTR**, which comes from the path or the body. The two are never linked (see `identifiers.md`).

## 4. Auditing

`AuditServiceImpl.sendEvent` sends an `ExtendedDataEvent` with `auditSource = appName`, tagged with the request path. The call is fire-and-forget, and failures are logged.

| Audit type | Sent from | Detail fields |
|---|---|---|
| `PensionSchemeReturnPost` | `ApiAuditUtil.firePsrPostAuditEvent` (connector `andThen`, on success and on failure) | `pensionSchemeAdministratorId`/`schemeAdministratorName` or `pensionSchemePractitionerId`/`schemePractitionerName`, `pensionSchemeTaxReference`, `schemeName`, full ETMP `payload`, `httpStatus`, `response`, `errorMessage`, `psrStatus` (`ChangedCompiled`/`ChangedSubmitted`) |
| `PensionSchemeReturnEmailEvent` | `EmailResponseController` | PSA/PSP id and name fields, `emailInitiationRequestId`, `emailAddress`, `event`, `submittedBy`, `reportVersion`, `pensionSchemeTaxReference`, `schemeName`, `taxYear` |
| bootstrap `ServerInternalError` / `ClientError` / `ResourceNotFound` | `JsonErrorHandler.onServerError` | `transactionFailureReason` |

## 5. Inbound callers and callbacks

| Caller | Endpoints | Auth |
|---|---|---|
| PSR frontend | `GET /pension-scheme-return/psr/overview/:pstr`, `GET /psr/versions/:pstr`, `GET /psr/versions/years/:pstr`, `POST /psr/standard`, `POST /psr/pre-populated`, `GET /psr/standard/:pstr` | Bearer session + `PsrAuth` |
| Email service (callback) | `POST /pension-scheme-return/psr/email-response/:submittedBy/:requestId/:email/:psaId/:pstr/:reportVersion/:schemeName/:taxYear/:userName` | **None.** The path segments `email`, `psaId`, `pstr`, `schemeName` and `userName` are AES-encrypted with `queryParameter.encryption` (decrypted with `SymmetricCryptoFactory.aesCryptoFromConfig`). |

- This repository does not register the email callback URL anywhere. Whoever sends the confirmation email (presumably the frontend) must build the URL and encrypt the values with the same key.
- ⚠️ **Observation:** the callback endpoint has no authentication. Anyone who can reach it with values encrypted under the shared key can create audit events. A default value for `queryParameter.encryption.key` is committed in `conf/application.conf`; it has to be overridden in each deployed environment.

## Datastore

None. The service has no MongoDB or other persistence (no `hmrc-mongo` dependency). It is stateless, and ETMP is the system of record.

## Libraries

| Library | Version | Used for |
|---|---|---|
| `uk.gov.hmrc:bootstrap-backend-play-30` | 10.7.0 | Play 3.0 backend, `HttpClientV2`, auth client, audit, `JsonErrorHandler`, crypto |
| `uk.gov.hmrc:http-verbs-play-30` (transitive) | 15.8.0 | `HttpClientV2`, `url` interpolator, HTTP exceptions |
| `com.networknt:json-schema-validator` | 1.5.9 | Outbound payload validation (Draft 4) |
| `com.fasterxml.jackson.module:jackson-module-scala` | 2.21.1 | `ObjectMapper` used by the schema validator |
| `org.typelevel:cats-core` | 2.13.0 | `traverse` in the transformers |
| Scala | 3.5.2 | |

## Local development

- `sm2 -start PSR_ALL` starts the dependency stack. To run this service from source, stop `PENSION_SCHEME_RETURN` and run `sbt run` (port **10700**).
- Local defaults: auth `localhost:8500`, IF `localhost:10702` (env `ist0`, token `local`), pensions-scheme `localhost:8203`. The IF port is assumed to be served by a stub in the `PSR_ALL` profile; the stub is not part of this repository.
- `testOnlyDoNotUseInAppConf.routes` defines no test-only endpoints.
