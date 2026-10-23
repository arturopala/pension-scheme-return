# Pension Scheme Return – Summary

## Purpose

This is the backend for the **Pension Scheme Return (PSR)**, a feature of HMRC's "Manage your pension schemes" (PODS) service. A pension scheme return is the annual report a registered pension scheme makes for a tax period. It covers the scheme's bank account and membership counts, total assets and cash, loans, land and property, bonds, shares, other assets, and member-level payments such as contributions, transfers, lump sums and surrenders, followed by a declaration.

The service has no UI. It is called by the PSR frontend for logged-in **Pension Scheme Administrators (PSAs)** and **Pension Scheme Practitioners (PSPs)**. It also receives delivery callbacks from the email platform. It keeps no data of its own. Every return is stored in HMRC's tax platform (ETMP), which it reaches through the Integration Framework. Its main job is to authorise the user against the scheme, translate between the frontend's data model and the ETMP model, check outgoing returns against the published API contract, and audit what was submitted.

## Main Capabilities

### 1. Return overview for a scheme

- Lists the returns that exist for a scheme within a date range. For each return it gives the period, its due date, the notice-to-file date, the number of versions, whether a compiled or submitted version exists, and the report type (standard or SIPP).
- Requests for dates before 6 April 2015 (a configurable earliest date) are moved forward to that date.
- If the scheme has no returns, the list is empty rather than an error.

### 2. Version history

- Lists every version of a return for one period start date, or for several start dates in one call. Each version has its form-bundle number, version number, status (compiled, submitted and in progress, or submitted and processed), date, and who submitted it.
- Start dates before the earliest allowed date are dropped from multi-year requests.
- If there are no versions, or ETMP is temporarily unavailable, the list comes back empty.

### 3. Compile and submit a standard return

- Takes a whole return (or part of one) from the frontend and maps it to the ETMP format. Along the way it derives counts, maps codes and turns yes/no flags into ETMP values.
- A return **with** a declaration is sent as *Submitted*. Without one it is sent as *Compiled*, a saved draft.
- When an existing return is amended, the service marks it as "changed compiled" or "changed submitted", based on the incoming version and status. This marker is used for auditing only.
- Before sending, the payload is checked against the ETMP JSON schema. A feature flag picks between schema versions 1.2.0 and 1.1.5. If the payload fails the check, the request is rejected and nothing is sent.
- If ETMP rejects the payload as a bad request, the caller gets *Expectation Failed* rather than *Bad Request*.

### 4. Submit a pre-populated return

- Uses the same flow as a standard submission, but checks against a relaxed "optional" schema. That schema makes most transaction-detail fields optional, so a return carried over from the previous year can be saved before the user has finished it.

### 5. Retrieve a return

- Fetches one return by form-bundle number, or by period start date and version. It maps the return back into the frontend model, including members, employer contributions, transfers, assets, shares and loans.
- If the return does not exist, the caller gets *Not Found*. If the stored data cannot be mapped back (for example, an employer with neither an ID nor a reason), the caller gets a structured error.

### 6. Email delivery tracking

- Receives delivery-status callbacks for return-confirmation emails. Most details travel encrypted in the callback URL.
- Writes one audit event per status (sent, delivered, permanent bounce, complained). "Opened" events are ignored.

## Cross-cutting Concerns

- **Authentication and authorisation**: users must be signed in with a PSA or PSP enrolment. Each request names the scheme by its scheme reference number in a header. The service asks the pension scheme registry whether the user's PSA or PSP ID is associated with that scheme. An optional role header can force the check to use the PSA or the PSP identity. ⚠️ The return itself is chosen by a different identifier (the tax reference, PSTR, from the path or body), and nothing checks that the PSTR belongs to the authorised scheme. See `identifiers.md`.
- **Auditing**: every submission is audited with the full payload, the submitter's identity and role, the scheme, the HTTP outcome and the "changed" status. Email delivery events are audited too. Retrievals are only logged, not audited.
- **Validation and transformation**: incoming requests are checked for structure only (required fields, types and enumerations). Formats are enforced by the ETMP JSON schema on outbound submissions. A two-way mapping layer converts between the two models.
- **Data protection**: the service stores nothing. Identifiers in email callback URLs are encrypted with a symmetric key. Audit events contain personal data from the return, such as NINOs, names and dates of birth.
- **Feature toggles**: `submitPsrSchemaVersionV120` chooses which schema submissions are checked against (v1.2.0 or v1.1.5). `earliestPsrPeriodStartDate` sets the earliest period that can be queried.

## Dependencies (generic)

| Kind | Role |
|---|---|
| Tax platform (ETMP via Integration Framework) | System of record for returns: overview, versions, submit and retrieve |
| Pension scheme registry | Checks whether a PSA or PSP is associated with a scheme |
| Identity / auth platform | Sign-in, enrolments (PSA / PSP IDs), external user ID |
| Audit platform | Submission and email-delivery audit events |
| Email platform (inbound) | Delivery-status callbacks |

## In One Sentence

A stateless backend that lets authorised pension scheme administrators and practitioners view, compile, submit and retrieve their scheme's annual Pension Scheme Return in HMRC's tax platform, checking every submission against the API contract and auditing it.
