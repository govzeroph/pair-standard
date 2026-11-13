# PAIR Specification

**Publicly Accessible Initiative Registry**

Version 0.1.0 (draft) · 2026-11-12

## 1. Purpose

PAIR defines how a civic initiative is described in a machine-checkable file, so that a
collection of such files can answer one question reliably:

> Does something that does this already exist?

The specification covers the shape of an entry and the rules a validator enforces. It does
not mandate a particular search implementation, user interface, or hosting arrangement.

## 2. Conformance language

The keywords MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be interpreted as described
in RFC 2119.

A **registry** is a collection of entry files. A **validator** is any tool that checks entries
against this specification. A **consumer** is any tool that reads a registry to answer
questions for a person.

## 3. Repository layout

A conforming registry MUST store entries as individual files under `initiatives/`:

```
initiatives/
  lgu-budget-tracker.yaml
  flood-report-ph.yaml
```

One file describes exactly one initiative. Entries MUST be YAML 1.2. Nested directories under
`initiatives/` are not permitted in this version.

## 4. Entry format

### 4.1 Example

```yaml
id: lgu-budget-tracker
name: LGU Budget Tracker
status: live
summary: Shows approved and disbursed budget per local government unit.
capabilities:
  - Check how much budget my city or municipality received this year
  - See what a specific budget line was actually spent on
  - Compare spending between two local government units
scope: national
url: https://example.org/budget-tracker
repo: https://github.com/example/budget-tracker
contact: Civic Data Collective <hello@example.org>
tags: [budget, transparency, local-government]
last_verified: 2026-11-12
```

### 4.2 Field summary

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Stable slug, matches filename |
| `name` | string | yes | Human-readable name |
| `status` | enum | yes | `idea`, `building`, `live`, `dormant` |
| `summary` | string | yes | One sentence, what it is |
| `capabilities` | string[] | yes | What a person can do with it |
| `scope` | string | yes | `national` or a named place |
| `last_verified` | date | yes | ISO 8601, `YYYY-MM-DD` |
| `url` | string | conditional | Required when `status` is `live` |
| `repo` | string | no | Source code location |
| `contact` | string | no | Person or organisation to reach |
| `tags` | string[] | no | Lowercase slugs, max 8 |
| `superseded_by` | string | no | `id` of the entry that replaces this one |

Unknown top-level fields MUST cause validation to fail. This is deliberate: silent acceptance
of misspelled or invented fields is how registry schemas drift apart.

### 4.3 `id`

A stable identifier for the initiative.

- MUST match `^[a-z0-9]+(-[a-z0-9]+)*$`
- MUST be between 3 and 64 characters
- MUST equal the filename without its `.yaml` extension
- MUST be unique within the registry
- MUST NOT be reused for a different initiative, even after removal
- SHOULD NOT encode status, year, or organisation, since those change

### 4.4 `name`

The name people use for the initiative. Between 2 and 100 characters. Plain text, no markdown.

### 4.5 `status`

Exactly one of:

| Value | Meaning |
|---|---|
| `idea` | Proposed. No usable implementation exists yet. |
| `building` | Under active development. Not yet usable by the public. |
| `live` | Usable by the public right now. |
| `dormant` | Was live or in development, no longer maintained. |

`status` carries most of the registry's practical value. "Does this exist already" has a very
different answer for `live` than for `dormant`, and a consumer MUST surface the status
alongside any entry it presents.

An entry MUST NOT be deleted when an initiative dies. Set `status: dormant` instead, so that
the knowledge that it was attempted, and by whom, is preserved.

### 4.6 `summary`

One sentence describing what the initiative is, written for someone who has never heard of it.

- MUST be 200 characters or fewer
- MUST be plain text, no markdown
- SHOULD NOT include the initiative's own name, which is already in `name`
- SHOULD NOT contain promotional language

### 4.7 `capabilities`

The field that makes the registry findable, and the one contributors most often get wrong.

Each entry is a plain-language statement of something a person can actually do with the
initiative, written from that person's side rather than the builder's.

- MUST contain between 1 and 10 items
- Each item MUST be 140 characters or fewer
- Each item MUST be plain text, no markdown
- Each item SHOULD begin with a verb
- Each item SHOULD be phrased the way a member of the public would describe the task
- Items MUST NOT describe implementation (frameworks, data pipelines, architecture)

The reason for the user-side rule is vocabulary mismatch. Someone asks "can I check what my
city spent on road repairs", while a builder writes "fiscal transparency dashboard with
disbursement drill-down". Those two sentences share almost no words, and no amount of search
tuning fully recovers from a description that never says what the thing lets you do.

Good:

```yaml
capabilities:
  - Check how much my city spent on road repairs last year
  - Get notified when a new budget document is published for my area
```

Bad:

```yaml
capabilities:
  - Fiscal transparency dashboard
  - Built on an ETL pipeline with nightly ingestion
  - The best budget tool in the country
```

### 4.8 `scope`

Where the initiative applies. Either the literal string `national`, or the name of a place as
commonly written, for example `Cebu City` or `Province of Rizal`.

An initiative that serves one city MUST NOT declare `national` scope.

### 4.9 `url`

Where a member of the public actually uses the initiative.

- MUST be an absolute `https://` URL when present
- MUST be present when `status` is `live`
- SHOULD point at the usable thing, not at a landing page or announcement post

### 4.10 `repo`

Source code location, if public. MUST be an absolute `https://` URL when present.

### 4.11 `contact`

A person or organisation who can answer questions about the initiative. Free text. An email
address or a profile link is sufficient. MUST NOT contain a personal phone number or home
address.

### 4.12 `tags`

Up to 8 lowercase slugs matching `^[a-z0-9]+(-[a-z0-9]+)*$`. Tags are a browsing aid and
MUST NOT be relied on as the primary search mechanism, since tag vocabularies drift.

### 4.13 `last_verified`

The date someone last confirmed the entry is still accurate.

- MUST be an ISO 8601 calendar date, `YYYY-MM-DD`
- MUST NOT be in the future

A registry's value is its accuracy about the present. Civic projects lapse, domains expire,
volunteers move on, and an entry that claims `live` while its site returns 404 actively
misleads the person who then decides not to build the thing.

Validators MUST emit a warning, not an error, when `last_verified` is more than 180 days old.
Consumers SHOULD present such entries as unverified rather than hiding them.

Automatic liveness checking, such as a scheduled job requesting every `url`, is out of scope
for this version. Freshness is maintained by people re-verifying entries and bumping the date.

### 4.14 `superseded_by`

When an initiative is replaced by or merged into another, set `status: dormant` and point
`superseded_by` at the surviving entry's `id`. The referenced `id` MUST exist in the registry,
and the reference MUST NOT be circular.

## 5. Validation rules

A conforming validator MUST reject an entry when any of the following holds.

1. The file is not valid YAML.
2. A required field is missing: `id`, `name`, `status`, `summary`, `capabilities`, `scope`, `last_verified`.
3. An unknown top-level field is present.
4. `id` does not match the required pattern, or does not equal the filename stem.
5. `id` is not unique within the registry.
6. `status` is not one of the four permitted values.
7. `status` is `live` and `url` is absent.
8. `capabilities` is empty, or has more than 10 items, or any item exceeds 140 characters.
9. `summary` exceeds 200 characters, or `name` is outside 2 to 100 characters.
10. `scope` is empty or not a string.
11. `url` or `repo` is present and is not an absolute `https://` URL.
12. `last_verified` is not a valid `YYYY-MM-DD` date, or is in the future.
13. `tags` has more than 8 items, or any tag does not match the slug pattern.
14. `superseded_by` is present and does not resolve to an existing `id`, or forms a cycle.

A conforming validator MUST emit a warning, without failing, when `last_verified` is more than
180 days old. Validators MUST NOT make network requests as part of validation.

## 6. Consumer expectations

A consumer answering a question from a member of the public:

- MUST show `status` and `last_verified` alongside any entry it presents
- MUST NOT claim an initiative does something that is not stated in its `capabilities`
- SHOULD state plainly when nothing in the registry matches, rather than offering a weak match
- SHOULD record unmatched questions, since an unanswered question is evidence of an unmet
  civic need and is useful to the people deciding what to build next

That last point is the registry's second purpose. The gaps are as informative as the entries.

## 7. Versioning

This specification uses semantic versioning.

- **Patch**: wording clarified, no change to what validates
- **Minor**: new optional field, or a relaxed rule. Existing valid entries stay valid.
- **Major**: a new required field, a removed field, or a tightened rule. Existing entries may
  need migration.

A registry MAY declare the specification version it targets in a `pair.yaml` file at its root.
Absent that declaration, consumers SHOULD assume the latest published version.

## 8. Licensing

This specification and the registry data are both released under CC0 1.0 Universal, placing
them in the public domain. Implementing this specification, reusing the data, or adapting the
document for another jurisdiction requires no permission and imposes no conditions.

Attribution is welcome but not required. The requirement was dropped deliberately: an
attribution chain follows a dataset into every system that ingests it, which discourages
exactly the reuse a registry exists to enable.

Reference tooling is licensed separately and is not covered by this grant. A conforming
implementation may be written from this document alone.
