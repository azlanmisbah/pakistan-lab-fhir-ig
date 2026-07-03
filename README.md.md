# Pakistan Lab Data — FHIR R4 Implementation Guide

### A proposed implementation guide for representing diagnostic laboratory data from legacy analyzers in Pakistani laboratory environments

**Status:** Draft — v0.1
**Published:** July 2026
**Author:** Azlan Misbah, Cathlum Systems
**Contact:** azlan.misbah@gmail.com

> **Disclaimer:** This document is a technical implementation guide based on operational experience. It is not affiliated with, endorsed by, or representative of any government agency, regulator, or standards body. It is published as an independent working draft.

---

## In Plain Terms

Most diagnostic labs in Pakistan run modern analyzers that produce real clinical data, but that data is never captured in a structured, shareable form — it is read off a screen, written on paper, and lost. There is currently no agreed national standard for what that data should look like if it *were* captured digitally.

This document proposes one. It describes, in concrete terms, how a lab result — a haemoglobin value, a full blood count, a pathology note — can be represented using FHIR R4, the international standard already used by health systems elsewhere in the world. It is based on a working implementation validated against real analyzer output in operational testing environments, not a theoretical proposal. It is published as a draft, openly, so that other people building similar systems in Pakistan can align with it, challenge it, or improve it before fragmentation sets in.

---

## 1. Purpose and Scope

Pakistan does not yet have a government-mandated FHIR baseline profile for laboratory data. Rather than wait for one, this guide documents a working implementation, so that a shared reference point exists before fragmentation sets in — before every builder in this space invents an incompatible format independently.

Scope is deliberately narrow: diagnostic laboratory results only. This is not a general electronic health record specification. It covers how a result leaves a legacy analyzer and becomes a structured, standards-compliant clinical record.

FHIR R4 was chosen over a locally-invented format for one practical reason: any structured format built in isolation eventually has to be translated into something else to be useful outside its own system. FHIR is already the format that international EHR platforms, research tooling, and (eventually) any national health data initiative are likely to expect. Building to it now is cheaper than migrating to it later.

## 2. Background

Legacy laboratory analyzers — hematology, biochemistry, and similar instruments common in Pakistani labs — typically output results as raw ASTM or HL7 v2 streams over a serial or network interface. In practice, almost none of this output reaches any structured system. It is read manually and transcribed, or simply printed.

This guide describes the FHIR R4 representation used by a system that intercepts that raw output directly at the analyzer and converts it in real time. The mapping described here has been validated against real analyzer output in operational testing environments.

## 3. Reference Architecture

The mapping in this guide assumes the following data path from analyzer to structured record:

```
  Legacy Analyzer
        │
        │  raw ASTM / HL7 v2 stream (serial or network)
        ▼
  Ingestion Layer
        │
        │  parses, validates, and normalises the raw stream
        ▼
  FHIR R4 Resources
        │
        │  Patient · Observation · DiagnosticReport
        ▼
  Storage / Downstream Systems
        (local database, reporting, EHR integration)
```

This document covers the third stage — the shape of the FHIR resources produced — not the specific ingestion implementation, which is expected to vary by analyzer manufacturer and deployment.

## 4. Resources In Scope

### 3.1 Patient

A minimal identity resource. Two identity paths are supported — see Section 5.

| Field | Notes |
|---|---|
| `identifier` | National ID (CNIC), when provided by the patient |
| `name` | Full name as captured at intake |
| Contact information | Modelled as a separate, linked contact resource rather than embedded directly, to allow more than one contact method per patient over time |

### 3.2 Observation

One Observation resource per individual test result (e.g. one for Haemoglobin, one for White Blood Cell count, within a single panel).

| Field | Notes |
|---|---|
| `status` | `final` for completed results |
| `identifier` | Local accession/order identifier, namespaced (e.g. `http://healthbox.pk/accession`) |
| `code.text` | Test name as reported by the analyzer |
| `subject` | Patient reference or display name |
| `referenceRange` | Included when the analyzer provides a normal range |
| `value[x]` | See below |

**Value handling** depends on result type:
- Numeric results → `valueQuantity`, with `value` and `unit`
- Qualitative or narrative results (e.g. pathology notes) → `valueString`
- Embedded visual results (e.g. scattergram images from hematology analyzers) → `valueAttachment`, with `contentType` and base64 `data`

This three-way split exists because legacy analyzers routinely output all three types within a single panel, sometimes within a single message, and a rigid numeric-only model silently drops the qualitative and visual results — which are often clinically significant.

### 3.3 DiagnosticReport

Groups the Observations produced by a single test order into one reportable unit, matching how a lab technically issues one panel per sample.

## 5. Identity Handling

**In plain terms:** not every patient wants or is able to provide a national ID. This guide supports two identity paths so that a system built to it does not force a choice between capturing data and respecting a patient's privacy.

**Path 1 — Anchored identity.** When a patient provides a CNIC, it is used as a deterministic identifier, allowing results to be reliably linked across visits and, where relevant, across facilities.

**Path 2 — Token identity.** In deployments where patients decline to provide identifying information — this has occurred in practice, particularly in privacy-sensitive clinical contexts — the system supports an anonymous token identity instead: a locally generated reference tied to the physical sample, with in-person result collection. No name, phone number, or national ID is fabricated or assumed on the patient's behalf under this path.

A national identity layer that only works when patients are willing to be identified is incomplete. This guide treats both paths as first-class, not the second as a fallback error state.

## 6. Terminology — LOINC Mapping

Test names as reported by analyzers are mapped to LOINC codes where a mapping exists. This table is small and actively growing as more analyzer types are integrated; it is not yet comprehensive.

| Local Test Name | LOINC Code |
|---|---|
| Hemoglobin | 718-7 |
| White Blood Cell | 26464-8 |
| Glucose | 2345-7 |
| Cholesterol | 2093-3 |
| Urine Protein | 2888-6 |
| WBC Scattergram | 11156-7 |

Unmapped test names are tagged `LOCAL` rather than dropped, pending formal LOINC assignment.

## 7. Handling Non-Standard Analyzer Output

Legacy analyzers frequently produce output that does not cleanly conform to the ASTM or HL7 specification they nominally implement — truncated frames, embedded binary or image data using encodings that can contain characters resembling markup, and inconsistent field ordering between manufacturers. A parser that assumes strictly well-formed input will drop or crash on a meaningful share of real-world messages.

This guide does not prescribe a specific parsing implementation, but notes that any compliant system needs deliberate handling for: multi-frame message reassembly, embedded attachment data that may contain reserved characters, and manufacturer-specific field positioning (addressed at the implementation level via per-analyzer configuration profiles, out of scope for this document).

## 8. Known Limitations (v0.1)

This is an honest list, not a complete one:

- Validated in production against one analyzer manufacturer to date; broader multi-manufacturer validation is in progress
- The LOINC mapping table (Section 6) covers a small initial set of common tests
- No formal conformance testing tooling exists yet for third parties wishing to validate against this guide
- This guide has not been reviewed by any government or standards body; it is a working draft, not an endorsed standard

## 9. Versioning and Contribution

This is a living document. Version history will be tracked here as it evolves.

| Version | Date | Change |
|---|---|---|
| v0.1 | July 2026 | Initial draft, based on live production deployment |

Feedback, issues, and proposed changes are welcome from anyone building similar systems in Pakistan or comparable environments. The goal of publishing this early and openly is alignment before fragmentation, not ownership of a fixed standard.

---

## Appendix: Example Observation Resource

```json
{
  "resourceType": "Observation",
  "status": "final",
  "identifier": [
    {
      "system": "http://healthbox.pk/accession",
      "value": "EXAMPLE-0001"
    }
  ],
  "code": {
    "text": "Hemoglobin"
  },
  "valueQuantity": {
    "value": 14.2,
    "unit": "g/dL"
  },
  "referenceRange": [
    {
      "text": "13.0-17.0"
    }
  ]
}
```
