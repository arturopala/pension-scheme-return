# Pension Scheme Return – Data Validation

On the way in, validation is **structural, not semantic**. Requests are read with Play JSON macro `Format`s, which check that required fields exist, that types are right and that enumerations hold known values, but they check no formats, lengths or ranges. The one local format rule is a regex on the SRN header. Formats are enforced on the way **out**: before a submission is sent, the transformed ETMP payload is checked against the bundled API 1999 JSON Schema. Reads (overview, versions, retrieve) send their path and query values to ETMP without checking them.

Unless an action catches it, every failure ends up in bootstrap's `JsonErrorHandler`. No controller has a `recover`/`recoverWith`, so the status code comes from the exception type: `HttpException` → its `responseCode`, `UpstreamErrorResponse` → `reportAs`, `AuthorisationException` → 401, anything else → **500**. Exceptions thrown synchronously inside `Action.async` (such as `requiredHeaders`) take the same route, so a synchronous `BadRequestException` still becomes a 400.

## Validation layers at a glance

| # | Layer | Mechanism | Where | On failure |
|---|---|---|---|---|
| 1 | Route binding | Play routes, all `String`/`Option[String]`/`Seq[String]` | `conf/app.routes` | Missing query param → 400 (Play `onClientError`) |
| 2 | Required headers | Presence check | `PsrBaseController.requiredHeaders` | `BadRequestException` → 400 |
| 3 | SRN format | Regex `^S[0-9]{10}$` | `Srn.apply`, `PsrAuth.authorisedUser` | `BadRequest("Invalid scheme reference number")` → 400 |
| 4 | Role header | `PSA`/`PSP`, case-insensitive, optional | `PsrAuth.authorisedUser` | `BadRequestException` → 400 |
| 5 | Auth and enrolment | `HMRC-PODS-ORG` or `HMRC-PODSPP-ORG`, plus `externalId` | `PsrAuth` | 401 |
| 6 | Scheme association | Remote check of PSA/PSP ↔ SRN | `SchemeDetailsConnector.checkAssociation` | `UnauthorizedException` → 401 |
| 7 | Body presence | `request.body.asJson` | `PsrBaseController.requiredBody` | `BadRequestException` → 400 |
| 8 | Body structure | `.as[PsrSubmission]` (macro formats + custom enum `Reads`) | `PsrSubmitController` | `JsResultException` → **500** |
| 9 | Outbound schema | networknt Draft-4 validation of the ETMP payload | `PsrSubmissionService`, `JSONSchemaValidator` | `PensionSchemeReturnValidationFailureException` → 400 |
| 10 | Upstream status mapping | Manual status match | `PsrConnector`, `HttpResponseHelper` | 400/404/4xx/5xx passed on, 400 on submit → 417 |
| 11 | Upstream body parsing | `response.json.as[...]` | `PsrConnector`, `SchemeDetailsConnector` | 500 |
| 12 | Reverse mapping | `Either[TransformerError, _]`, plus `.get`/`.head`/partial matches | `StandardPsrFromEtmp` and related | `Left` → 500 with `{"error": ...}`; exceptions → 500 |
| 13 | Email callback | AES decrypt, email regex, `EmailEvents` reads | `EmailResponseController` | 403 / 400 / 500 |
| 14 | Body size | `play.http.parser.maxMemoryBuffer = 10MB` | `application.conf` | 413 |

## Route and path parameters

| Parameter | Rule | Code |
|---|---|---|
| `pstr` (path, all GETs) | Any non-`/` segment. No format check. | `app.routes` |
| `fromDate`, `toDate` (overview) | Required strings, no date parsing. `fromDate` is replaced by `max(fromDate, "2015-04-06")` using **string** comparison. | `PsrOverviewController.getOverview` |
| `startDate` (versions) | Required string. No check, and no earliest-date filter. | `PsrVersionsController.getVersions` |
| `startDates` (versions/years) | Repeated query param. Values `< "2015-04-06"` (string comparison) are dropped. An empty list returns `[]`. | `PsrVersionsController.getVersionsForYears` |
| `fbNumber`, `periodStartDate`, `psrVersion` (get standard) | All optional. One of `fbNumber`, or both `periodStartDate` and `psrVersion`, must be present; otherwise `BadRequestException("Missing url parameters")` → 400. Not format-checked. | `PsrConnector.buildParams` |
| Email callback segments | `email`, `psaId`, `pstr`, `schemeName` and `userName` must decrypt with `queryParameter.encryption`. `submittedBy`, `requestId`, `reportVersion` and `taxYear` are free text. | `EmailResponseController.decryptDetails` |

⚠️ **Observation:** the date strings are compared lexicographically. That only works for ISO `yyyy-MM-dd` input, and a malformed value is sent to ETMP unchanged.

## Request headers

| Endpoint | Headers | Validation |
|---|---|---|
| `GET /psr/overview/:pstr`, `GET /psr/versions/:pstr`, `GET /psr/versions/years/:pstr` | `srn` (required), `requestRole` (optional) | `srn` present → 400 if missing. Regex → 400. `requestRole` must be `PSA`/`PSP` (any case) → 400 otherwise. |
| `POST /psr/standard`, `POST /psr/pre-populated`, `GET /psr/standard/:pstr` | `userName`, `schemeName`, `srn` (required), `requestRole` (optional) | Presence only, for `userName` and `schemeName`. They are not used for identity decisions, but they are copied into the audit events as the submitter and scheme names. |
| Email callback | none | n/a |

If `requestRole` names a role the user is not enrolled for (e.g. `PSP` for a PSA-only user), the result is `BadRequestException("Bad Request without psaPspId/credentialRole …")` → 400, not 403.

## Request bodies

### Submit standard / pre-populated PSR (`PsrSubmission`)

- Reader: `requiredBody.as[PsrSubmission]`, built from macro `Json.format` for every nested case class, plus these custom readers: `IdentityType` (`individual`/`ukCompany`/`ukPartnership`/`other`), `EmployerType` (`employerType` + `value` | `reason`), `PensionSchemeType` (`key`/`value`), `MemberState`, `EtmpPsrStatus`, `SubmitterType`, the share and asset enums.
- On failure: `JsResultException` is thrown inside the authorised block. It is not an `HttpException`, so the **caller gets 500** with the JSON-path error text. It is not a 400.
- `.as` runs **after** the auth and association calls. A malformed body still costs one auth call and one remote association lookup.

**What *is* checked**

- Required fields: everything not declared as `Option` must be present. For example `minimalRequiredSubmission.reportDetails.{pstr, periodStart, periodEnd}`, `schemeDesignatory.{openBankAccount, activeMembers, deferredMembers, pensionerMembers}`, `checkReturnDates`, and for each member `state`, `personalDetails.{firstName, lastName, dateOfBirth}`, `employerContributions` (as a list) and so on.
- Types: `LocalDate`/`LocalDateTime` are parsed, numbers are `Int`/`Double`, booleans are booleans.
- Enumerations, as listed above. `EtmpPsrStatus`, `SubmitterType` and similar custom `Reads` call `json.as[String]`, so a non-string value throws instead of returning a `JsError`. The result is the same 500.
- `EmployerType`: `UKCompany`/`UKPartnership` require `value` (the ID) **or** `reason`, and `Other` requires `value`.

**What is *not* checked locally** (and is left to the outbound schema or ETMP)

- No format rules on NINO, CRN, UTR, PSA/PSP IDs, PSTR, postcode, land registry title, transfer scheme reference or `fbVersion`.
- No conditional rules: `nino` vs `reasonNoNINO`, `idNumber` vs `reasonNoIdNumber`, and the declaration flags can all be absent or both present.
- No string lengths, no numeric ranges or negative checks, no counts or list sizes.
- No cross-field date logic: `periodStart ≤ periodEnd`, accounting periods inside the PSR period.
- **No check that `reportDetails.pstr` matches the authorised SRN.**
- `fbVersion` is unchecked, and `getCipPsrStatus` calls `version.toInt` on it, so a non-numeric `fbVersion` with a `fbstatus` raises `NumberFormatException` → **500**.

**Normalisation and derivation (`PsrSubmissionToEtmp` and sub-transformers)**

| Rule | Code |
|---|---|
| `psrStatus = Submitted` if `psrDeclaration` is present, else `Compiled` | `MinimalRequiredDetailsToEtmp` |
| `pstr`, `fbVersion`, `fbstatus` and `compilationOrSubmissionDate` are **not** sent in the payload; the PSTR only goes in the URL | `MinimalRequiredDetailsToEtmp`, `PsrSubmissionService` |
| Booleans → `"Yes"`/`"No"`, connected party → `"01"`/`"02"`, identity type → `"01".."04"`, employer org type → `"01".."03"`, transfer scheme → `"01"` (registered, ref) / `"02"` (QROPS, ref) / `"03"` (other, description) | `Transformer`, `IdentityType`, `EmployerContributionsTransformer` |
| Member counts `noOfContributions`, `noOfTransfersIn`, `noOfTransfersOut` are derived from list sizes; empty lists become absent | `MemberPaymentsTransformer.toEtmp` |
| `MemberState` → `memberStatus` (`New`/`Changed`/`Deleted`) | `MemberPaymentsTransformer` |
| `noOfLoans` is set only when `schemeHadLoans = true`. `loanTransactions` is emitted when the list is non-empty or the flag is true. `securityGiven` is derived from whether `optSecurityGivenDetails` is present | `LoansToEtmp` |
| Share flags and counts are derived per share type from the transactions. An empty `Shares` is dropped. A missing `totalValueQuotedShares` is replaced by the sentinel **`-0.01`**, which is turned back into `None` on read | `SharesToEtmp`, `SharesFromEtmp` |
| `optAuthorisingPSAID` → `psaId`. The declaration flags go to `psaDeclaration` or `pspDeclaration` depending on `submittedBy` | `PsrDeclarationToEtmp` |
| `CipPsrStatus` (audit only): `fbVersion > 1` or absent, combined with the status, gives `ChangedCompiled`/`ChangedSubmitted` | `PsrSubmissionService.getCipPsrStatus` |

## External validation services

| Service | Request | Decision | Fail mode |
|---|---|---|---|
| Pensions Scheme `is-psa-associated` | `psaId`/`pspId` + `schemeReferenceNumber` headers | `true` → proceed; `false` → 401 | **Fail closed.** A non-200 response propagates (404 → 404, 5xx → 5xx, other 4xx → same status), and a non-boolean body → 500. |

## Outbound payload validation

- **When:** synchronously, **before** sending, in `PsrSubmissionService.submitStandardPsr` and `submitPrePopulatedPsr`. Nothing is sent if validation fails.
- **Schema selection:**

| Flow | Schema file | Selector |
|---|---|---|
| Standard | `api-1999-submit-pension-scheme-return-request-schema-v1.2.0.json` | `submitPsrSchemaVersionV120 = true` (default) |
| Standard | `api-1999-submit-pension-scheme-return-request-schema-v1.1.5.json` | `submitPsrSchemaVersionV120 = false` |
| Pre-populated | `api-1999-submit-pension-scheme-return-request-schema-v1.2.0-optional.json` | always |

- v1.2.0 adds `checked` and `prePopulated` compared with v1.1.5. The "optional" variant drops required-ness from the loan, bond, land/property, lease, other-asset and share transaction detail fields (e.g. `totalIncomeOrReceipts`, `amountOutstanding`, `arrearsPrevYears`, `capRepaymentCY`, `intReceivedCY`) and from the section-level `…WereHeld`/`…WereDisposed` flags.
- **Mechanism:** `JsonSchemaFactory` V4. The schema is reloaded from the classpath **on every call**, and the payload is round-tripped through `JsValue.toString` → Jackson.
- **On failure:** `PensionSchemeReturnValidationFailureException` (a `BadRequestException`), whose message is the list of `ValidationMessage`s → **400**. The message goes to the caller and into the error log without any masking. networknt messages carry the JSON path and the rule; the library adds no masking of its own.
- **No unused schema files.** All three are referenced from `SchemaPaths`.
- **Retrieval** (`GET /psr/standard`) does no schema validation of the response.

## Upstream response validation

| Response | Validation | Failure result |
|---|---|---|
| Overview 200 | `as[Seq[PsrOverviewEtmpResponse]]`; all fields required, `psrReportType ∈ {Standard, SIPP}` | `JsResultException` → 500 |
| Versions 200 | `as[Seq[PsrVersionsEtmpResponse]]`; `reportStatus` enum | 500 |
| Get standard 200 | `as[PsrSubmissionEtmpResponse]` | 500 |
| Get standard → model | `StandardPsrFromEtmp.transform` returns `Either` | `Left(TransformerError)` → 500 with `{"error": "..."}` |
| Reverse mapping hazards | `accountingPeriods.head`, `psaDeclaration.get`/`pspDeclaration.get`, `classOfShares.get`, `memberLumpSumReceived.head`, `stringToIdentityType` (non-exhaustive match), `AssetsFromEtmp … headOption.get` | `NoSuchElementException`/`MatchError` → 500 |
| Association 200 | `as[Boolean]` | 500 |
| Submit 200 | Body read as `httpResponse.json` for the audit event only | A non-JSON 200 body throws in the audit callback (`andThen`); the caller still gets 204 |

## Inbound callbacks

`POST /psr/email-response/...` (`EmailResponseController.sendAuditEvents`), with no authentication:

1. `decrypt(email)` runs **outside** the `try`. A bad ciphertext throws `SecurityException` → **500**.
2. The decrypted email must match `Constants.emailRegex`, otherwise **403** `Malformed email address : <decrypted email>`. ⚠️ The decrypted address is echoed in the response.
3. The other four encrypted segments are decrypted inside the `try`, but only `IllegalArgumentException` is caught, so a bad ciphertext → **500**.
4. The body is parsed with `parser.tolerantJson` (any content type) and validated as `EmailEvents`. Each `event` must be one of `Sent`, `Delivered`, `PermanentBounce`, `Opened`, `Complained`, otherwise **400** "Bad request received for email call back event".
5. `detected` must match `yyyy-MM-dd'T'HH:mm:ss'Z'` or `…ss.SSS'Z'`. `parseDateTime` throws `IllegalArgumentException` from inside `Reads.map`, which escapes `validate`, so the result is **500**, not 400.
6. `Opened` events are dropped. Each remaining event produces one audit event, and the response is `200`.

## Cache / opaque stores

None. The service has no persistence.

## Error mapping summary

| Source | Exception / value | HTTP to caller |
|---|---|---|
| `requiredHeaders` / `requiredBody` | `BadRequestException` | 400 |
| Invalid SRN | `BadRequest(...)` | 400 |
| Invalid `requestRole`, or role without matching enrolment | `BadRequestException` | 400 |
| No enrolment / not logged in | `AuthorisationException` | 401 |
| Missing `externalId`, or not associated | `UnauthorizedException` | 401 |
| Malformed `PsrSubmission` body | `JsResultException` | **500** |
| Non-numeric `fbVersion` | `NumberFormatException` | **500** |
| Schema validation failure | `PensionSchemeReturnValidationFailureException` | 400 |
| Missing retrieval params | `BadRequestException` | 400 |
| IF 400 on submit | `BadRequestException` → `ExpectationFailedException` | **417** |
| IF 400 on GETs | `BadRequestException` | 400 |
| IF 404 | `NotFoundException` (overview/versions: empty list) | 404 (or 200 `[]`) |
| IF 422 `PSR_NOT_FOUND` on get standard | `None` | 404 |
| IF 503 on versions | swallowed | 200 `[]` |
| IF other 4xx/5xx | `UpstreamErrorResponse(reportAs = status)` | same status |
| IF unexpected status | `UnrecognisedHttpResponseException` | 500 |
| Upstream JSON mismatch / transformer exceptions | `JsResultException`, `NoSuchElementException`, `MatchError` | 500 |
| `TransformerError` | `Left` | 500 + `{"error": ...}` |
| Email: bad ciphertext / bad `detected` | `SecurityException` / `IllegalArgumentException` | 500 |
| Email: bad address / bad event | `Forbidden` / `BadRequest` | 403 / 400 |

## Recommendations

1. Read the body with `validate[PsrSubmission]` (or `parse.json[PsrSubmission]`) and return **400** with the path errors instead of letting `JsResultException` become a 500. Do this **before** the auth and association calls.
2. Check that the PSTR (from the path and from `reportDetails.pstr`) belongs to the SRN the user was authorised for, e.g. by having the association check return the scheme's PSTR.
3. Add local format checks for the identifiers that go into URLs (PSTR, dates, form-bundle number, version), and encode them per segment or parameter instead of pre-formatting the full URL.
4. Guard `getCipPsrStatus` against a non-numeric `fbVersion`, and validate `fbVersion` against the `^[0-9]{3}$` version pattern.
5. Make the email callback fail with 400 for every decryption and date-parse error (move `decrypt(email)` inside the `try`, catch `SecurityException`, and turn `parseDateTime` into a `JsError`). Stop echoing the decrypted email address.
6. Cache the compiled `JsonSchema` instances instead of reloading them on every submission.
7. Replace `.get`/`.head`/partial matches in the `*FromEtmp` transformers with `TransformerError`s, so that unexpected ETMP data produces a clear error.
8. Reconsider turning a versions 503 into an empty list. It hides outages from the frontend.
