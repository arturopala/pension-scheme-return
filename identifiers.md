# Pension Scheme Return – Business Identifiers

This document lists every identifier the service handles: where it comes from, where it goes, and how it is validated. The validation rules come from three sources:

- **Local**: checks in this repository (regexes, binders, normalisers).
- **Library**: checks built into domain types from dependencies. **None apply here.** The service does not use `uk.gov.hmrc.domain` (`Nino`, `PsaId`, …) or any other wrapper type. Every identifier is a plain `String`.
- **Contract**: patterns in the bundled ETMP API 1999 JSON schemas (`conf/resources/schemas/*.json`). These apply only to the **submission payload**. URL parameters and read responses have no contract file in this repo.

Terminology:

- **SRN** = Scheme Reference Number (PODS registry key).
- **PSTR** = Pension Scheme Tax Reference (ETMP key).
- The codebase uses **"fbNumber" / "psrFormBundleNumber" / "reportFormBundleNumber"** for the same thing.
- **"fbVersion" / "psrVersion" / "reportVersion"** are the return version.
- **"psaPspId"** holds either a PSA ID or a PSP ID.
- **"credentialRole"** is the PSA or PSP role.

## Summary

| Identifier | Identifies | Format (contract) | Local validation | Enters via | Main uses |
|---|---|---|---|---|---|
| SRN | Scheme in the PODS registry | none here | `^S[0-9]{10}$` | `srn` header (untrusted) | `schemeReferenceNumber` header to the association check |
| PSTR | Scheme in ETMP | none for the URL. Payload `refNumber` alt. `[0-9]{8}[A-Z]{2}` | **none** | path `:pstr` (GET), body `reportDetails.pstr` (POST), encrypted email path (all untrusted). ETMP response `schemeDetails.pstr` | IF URL path, audit `pensionSchemeTaxReference`, logs |
| PSA ID | Administrator | `psaId ^A[0-9]{7}$` (declaration) | none | **auth enrolment `HMRC-PODS-ORG`/`PSAID` (trusted)**. Body `psrDeclaration.submitterId`/`optAuthorisingPSAID`, encrypted email path (untrusted) | association header `psaId`, audit, ETMP declaration |
| PSP ID | Practitioner | `submitterId ^[A\|0\|1\|2][0-9]{7}$` | none | **auth enrolment `HMRC-PODSPP-ORG`/`PSPID` (trusted)**. Body, encrypted email path | association header `pspId`, audit, ETMP declaration |
| Credential role | PSA vs PSP | `submittedBy ∈ {PSA, PSP}` | `requestRole ∈ {PSA, PSP}` (case-insensitive) | `requestRole` header, enrolment, body `submittedBy`, email path `submittedBy` | chooses the enrolment, audit labels |
| externalId | Auth user | n/a | presence | auth retrieval (trusted) | **unused** after retrieval |
| Form bundle number | One return version in ETMP | none | none | query `fbNumber`; versions response `reportFormBundleNumber` | IF query `psrFormBundleNumber` |
| Period start / PSR version | A return by period + version | none in URL | none | query `periodStartDate`, `psrVersion`, `startDate(s)`, `fromDate`/`toDate` | IF query, earliest-date filter |
| fbVersion / fbstatus | Version and status of the return being amended | none (not sent) | none (`toInt` at use) | body `reportDetails` | `CipPsrStatus` for audit |
| recordVersion / memberPSRVersion | Optimistic version of a section / member | `^[0-9]{3}$` | none | body; ETMP response | ETMP payload round-trip |
| NINO | Scheme member | `^[A-Z]{2}[0-9]{6}[A-Z]$` | none | body `personalDetails.nino` | ETMP payload, audit payload |
| idNumber (party) | Loan recipient, seller, buyer, lessee etc. (NINO / CRN / UTR by type) | `^[A-Z0-9]{8,10}$` | none | body `*.idNumber` | ETMP `…IdentityType.idNumber` |
| Employer ID | Contributing employer (CRN / UTR) | `^[A-Z0-9]{8,10}$` | none | body `employerType.value` | ETMP `organisationIdentity.idNumber` |
| CRN (shares) | Company whose shares are held | `^[A-Z0-9]{8,10}$` | none | body `optCrnNumber` | ETMP `crnNumber` |
| Transfer scheme ref | Registered scheme (PSTR) or QROPS | `^(Q[0-9]{6}\|[0-9]{8}[A-Z]{2})$` | none | body `transferSchemeType.value` | ETMP `refNumber` |
| Land registry title | Property | `^.{1,35}$` | none | body `landRegistryTitleNumberValue` | ETMP `landRegistryReference` / `reasonNoReference` |
| Postcode / country | Property address | UK postcode regex (≤10), ISO-2 enum | none | body `addressDetails` | ETMP `ukPostCode` / `addressLine5`, `countryCode` |
| CorrelationId | IF request | n/a | n/a | generated (UUID) | IF header |
| Email requestId | Email send request | n/a | none | email callback path | audit `emailInitiationRequestId` |

## Scheme identifiers: SRN and PSTR

**Lifecycle.** The PODS registry issues the SRN when the scheme is registered. HMRC/ETMP issues the PSTR. This service never maps one to the other. The frontend sends both.

**SRN**
- Source: the `srn` request header (untrusted) on every frontend endpoint.
- Local rule: `Srn.apply` regex `^S[0-9]{10}$`. Failure → 400 `Invalid scheme reference number`. `Srn` is a plain case class with an `Option` factory and no `require`, so bad input never becomes a 500.
- Use: sent only as the `schemeReferenceNumber` header to `GET /pensions-scheme/is-psa-associated`. It is not audited, not used in any ETMP call, and not logged.

**PSTR**
- Sources (all untrusted):
  - GET endpoints: the `:pstr` path segment (Play URL-decodes it).
  - POST endpoints: `minimalRequiredSubmission.reportDetails.pstr` in the body.
  - Email callback: the encrypted `:pstr` segment.
  - Also returned by ETMP in `schemeDetails.pstr` and copied into `reportDetails.pstr` of the retrieve response.
- Uses: interpolated into the IF URL path for every ETMP operation, put in the audit `pensionSchemeTaxReference` field, and logged at info level. It is **not** part of the submission payload (`EtmpReportDetails` has no `pstr`).
- Validation: none locally, none from a library, and no contract pattern (the PSTR is never inside a schema-validated body). The only place the PSTR format appears in the contracts is `refNumber`'s alternative `[0-9]{8}[A-Z]{2}`.
- ⚠️ **Observation (authorisation gap):** `PsrAuth` authorises the user against the **SRN header**, but the ETMP call is made for the **PSTR** from the path or body. Nothing checks that the PSTR belongs to that SRN. A user associated with any one scheme can pass that scheme's SRN and read, or submit, the return of another PSTR.
- ⚠️ **Observation (URL injection):** `config.getOverviewUrl.format(pstr, fromDate, toDate)` (and the same for versions, submit and get) builds the full URL as a string, which is then passed as the single argument of `url"$url"`. Values are therefore parsed as URL structure, not encoded. A PSTR or date containing `?`, `&`, `#` or `%2F`-decoded characters can change the path or add query parameters on the IF call (for example `fbNumber=x&periodStartDate=…`).

## User and enrolment identifiers

| Identifier | Trusted source | Untrusted sources | Uses |
|---|---|---|---|
| PSA ID | `HMRC-PODS-ORG` enrolment, identifier `PSAID` | `psrDeclaration.submitterId`, `psrDeclaration.optAuthorisingPSAID`, email path `:psaId` (encrypted) | `psaId` header to the association check, `PsrAuthContext.psaPspId`, audit `pensionSchemeAdministratorId`, ETMP `psrDeclaration.submitterId`/`psaId` |
| PSP ID | `HMRC-PODSPP-ORG` enrolment, identifier `PSPID` | as above | `pspId` header, audit `pensionSchemePractitionerId`, ETMP declaration |
| externalId | `Retrievals.externalId` | none | stored in `PsrAuthContext` and **never read** |
| Credential role | derived from the enrolment | `requestRole` header, body `submittedBy`, email `:submittedBy` | chooses the enrolment, audit field naming, ETMP `submittedBy` |

- When both enrolments are present and there is no `requestRole`, the PSA ID wins (`getPsaPspId`).
- ⚠️ **Observation:** the declaration's `submitterId`, `optAuthorisingPSAID` and `submittedBy` come from the body and are **never compared** with the authenticated PSA or PSP ID. A PSP could submit a declaration that names any PSA ID as submitter. The ETMP contract checks only the format.
- ⚠️ **Observation:** `credentialRole` is `"PSA"`/`"PSP"` when no `requestRole` header is sent, but `"psaId"`/`"pspId"` when it is. Audit labelling only recognises `"PSA"`, so a PSA submitting with `requestRole: PSA` is audited as a practitioner.
- In the email callback, the PSA/PSP ID and the role (`submittedBy`) come only from the URL. They are trusted because the values decrypt, not because of any authentication.
- Contract mismatch: `submitterId` is `^([A|0|1|2]{1}[0-9]{7})$`. The `|` inside the character class is a literal, so `|1234567` passes the contract. This is probably meant to be `[A012]`.

## Party / tax identifiers (inside the return)

| Identifier | Frontend key(s) | Upstream key | Local rule | Contract rule |
|---|---|---|---|---|
| Member NINO | `memberDetails[].personalDetails.nino` / `reasonNoNINO` | `memberDetails[].personalDetails.nino` / `reasonNoNINO` | none (neither, both, or one may be sent) | `^[A-Z]{2}[0-9]{6}[A-Z]$`; reason `^.{1,160}$` |
| Loan recipient ID | `loanTransactions[].recipientIdentityType.{identityType,idNumber,reasonNoIdNumber,otherDescription}` | `recipientIdentityType.{indivOrOrgType,idNumber,reasonNoIdNumber,otherDescription}` | `identityType` enum only | `indivOrOrgType ∈ 01..04`, `idNumber ^[A-Z0-9]{8,10}$` |
| Property / share / asset counterparties | `optPropertyAcquiredFrom`, purchaser / lessee identity (`PropertyAcquiredFrom`) | `acquiredFromType`, `purchaserType`, … (`EtmpIdentityType`) | enum only | as above |
| Employer (contributions) | `employerContributions[].employerType.{employerType, value \| reason}` | `organisationIdentity.{orgType,idNumber,reasonNoIdNumber,otherDescription}` | `employerType` ∈ `UKCompany`/`UKPartnership`/`Other`; `value` or `reason` present | `orgType ∈ 01..03`, `idNumber ^[A-Z0-9]{8,10}$` |
| Shares company CRN | `shareIdentification.optCrnNumber` / `optReasonNoCRN` | `crnNumber` / `reasonNoCRN` | none | `^[A-Z0-9]{8,10}$` / `^.{1,160}$` |
| Transfer scheme | `transferSchemeType.{key,value}` | `transferSchemeType.{schemeType,refNumber,otherDescription}` | `key` enum | `schemeType ∈ 01..03`, `refNumber ^(Q[0-9]{6}\|[0-9]{8}[A-Z]{2})$` |
| Land registry title | `landRegistryTitleNumberKey` + `landRegistryTitleNumberValue` | `landRegistryReferenceExists` + `landRegistryReference` **or** `reasonNoReference` | none | `^.{1,35}$` |
| Postcode | `addressDetails.postCode` | `ukPostCode` (if UK) or `addressLine5` (if not) | none | `ukPostCode` UK regex; `addressLine5` `^.{1,35}$` |

**Normalisation.** Identifier values are not normalised at all: no upper-casing, no whitespace stripping, no space removal. The only transformations are structural:
- Identity type mapping: `individual→01`, `ukCompany→02`, `ukPartnership→03`, `other→04`. The reverse mapping `stringToIdentityType` is non-exhaustive (unknown → `MatchError` → 500).
- Employer org type mapping: `UKCompany→01`, `UKPartnership→02`, `Other→03`. On the way back, a company or partnership with neither ID nor reason → `TransformerError.NoIdOrReason`.
- Transfer scheme: `registeredPS→01`, `qualifyingRecognisedOverseasPS→02` (both use `refNumber`), `other→03` (`otherDescription`). A registered-scheme reference is therefore a PSTR-shaped value, and a QROPS reference is `Q` + 6 digits. The contract accepts either shape for both types.
- The land registry value goes to `landRegistryReference` when the key is true, otherwise to `reasonNoReference`. On read it is restored with `headOption.get`, so if both are missing the result is a 500.
- The UK postcode is sent as `ukPostCode` only when `landOrPropertyInUK` is true. Otherwise it goes into `addressLine5`, which skips the postcode regex.

Lowercase or spaced NINOs and CRNs from the frontend fail the outbound schema and produce a 400 listing the JSON path. This happens only on submission. Pre-populated submissions apply the same identifier patterns.

## Return / version identifiers

- **Form bundle number** is issued by ETMP per return version. It is returned in the versions response (`reportFormBundleNumber`) and sent back as the `fbNumber` query parameter, which becomes `?psrFormBundleNumber=` upstream. It is not validated and not encoded. It takes priority over `periodStartDate` + `psrVersion`, which are ignored when `fbNumber` is present.
- **psrVersion / reportVersion / fbVersion**: ETMP returns `reportVersion: Int` (versions) and `fbVersion: String` (retrieve). The frontend sends `fbVersion` back on amendments, and the service uses it only for `CipPsrStatus` via `version.toInt` (non-numeric → 500). It is not sent to ETMP.
- **recordVersion / memberPSRVersion**: ETMP optimistic-concurrency tokens per section and per member. They are copied both ways without change. Contract: `^[0-9]{3}$`.
- **Period dates** (`periodStartDate`, `startDate(s)`, `fromDate`/`toDate`) are plain strings in URLs. The earliest-date filter compares them as strings.

## Technical identifiers

| Identifier | Source | Use | Notes |
|---|---|---|---|
| IF `CorrelationId` | `randomUUID.toString.slice(0, 36)` per call | IF header | Not logged and not linked to the inbound `X-Request-ID`. The association call does not send one. |
| `HeaderCarrier` request/session IDs | inbound headers | forwarded by `HttpClientV2`, audit tags | standard bootstrap behaviour |
| Email `requestId` | email callback path (plain text) | audit `emailInitiationRequestId` | not validated |
| Email address | email callback path (encrypted) | audit `emailAddress` | must match `Constants.emailRegex`, otherwise 403 (which echoes the address) |

## Validation coverage matrix

| Identifier | Format checked locally? | Checked against the authenticated user? | Encoded when put in a URL? |
|---|---|---|---|
| SRN | ✅ regex | ✅ remote association check | n/a (header) |
| PSTR | ❌ | ❌ (not linked to the SRN) | ❌ |
| PSA / PSP ID (auth) | ❌ (trusted enrolment) | ✅ | n/a (header) |
| Declaration `submitterId` / `optAuthorisingPSAID` | ⚠️ contract only | ❌ | n/a |
| `requestRole` | ✅ enum | ✅ must have the matching enrolment | n/a |
| Form bundle number | ❌ | ❌ | ❌ |
| `periodStartDate` / `psrVersion` / dates | ❌ (string compare only) | ❌ | ❌ |
| `fbVersion` | ❌ (`toInt` throws) | n/a | n/a |
| NINO / CRN / idNumber / transfer ref | ⚠️ contract only, on submit | n/a | n/a |
| Email callback IDs | ⚠️ decrypts (email: regex) | ❌ (no auth) | n/a |

## Recommendations

1. Bind the PSTR to the authorised SRN. For example, have the association check (or a scheme-details lookup) return the scheme's PSTR, and reject any request whose path or body PSTR differs.
2. Check that the declaration `submitterId`/`submittedBy` match the authenticated PSA/PSP ID and role. Allow `optAuthorisingPSAID` only for PSP submissions.
3. Add local format checks for PSTR (`^[0-9]{8}[A-Z]{2}$`), dates (ISO parse), form bundle number and version. Build IF URLs with per-segment interpolation (`url"$base/pension-online/scheme-return/$pstr?psrFormBundleNumber=$fb"`) so values are encoded.
4. Fix the `credentialRole` inconsistency (`psaId`/`pspId` vs `PSA`/`PSP`) so audit events label submitters correctly.
5. Normalise NINO and CRN (upper-case, strip spaces) before validating against the schema, and return field-level 400s.
6. Correct the `submitterId` contract pattern (`[A012]`) with the API owner, or tighten it locally.
7. Remove or use the unused `externalId` retrieval, and send the inbound request ID as, or alongside, the IF `CorrelationId` so calls can be traced.
