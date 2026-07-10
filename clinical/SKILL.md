---
name: medplum-clinical
description: "FHIR resource mapping for clinical documentation in Medplum. Use when creating SOAP notes, encounter records, visit templates, or structured clinical data."
---

# Clinical — SOAP Notes, FHIR Mapping, Visit Templates

Stock LLMs will create a single `DocumentReference` for a SOAP note and call it done. That's not how Medplum works. A SOAP note is a structured transaction Bundle of multiple FHIR resources.

## SOAP Note = FHIR Transaction Bundle

| SOAP Section | FHIR Resource | Code | Purpose |
|---|---|---|---|
| Subjective | Observation | LOINC 10210-3 | Patient-reported symptoms |
| Objective (vitals) | Observation | LOINC 8310-5, 8480-6, 8462-4 | Vital signs |
| Objective (exam) | Observation | LOINC 10210-3 | Physical exam findings |
| Assessment | Condition | SNOMED 439401001 | Diagnoses |
| Assessment | ClinicalImpression | — | Clinical reasoning |
| Plan | CarePlan | SNOMED 736350006 | Treatment plan |
| Plan | MedicationRequest | — | Prescriptions |
| Context | Encounter | — | The visit itself |
| Container | Composition | — | Links all sections |

### Example Bundle structure

```json
{
  "resourceType": "Bundle",
  "type": "transaction",
  "entry": [
    {
      "fullUrl": "urn:uuid:encounter-1",
      "resource": {
        "resourceType": "Encounter",
        "status": "finished",
        "class": { "code": "AMB", "system": "http://terminology.hl7.org/CodeSystem/v3-ActCode" },
        "subject": { "reference": "Patient/PATIENT_ID" },
        "participant": [{ "individual": { "reference": "Practitioner/PRACTITIONER_ID" } }]
      },
      "request": { "method": "POST", "url": "Encounter" }
    },
    {
      "fullUrl": "urn:uuid:subjective-1",
      "resource": {
        "resourceType": "Observation",
        "status": "final",
        "category": [{ "coding": [{ "code": "survey", "system": "http://terminology.hl7.org/CodeSystem/observation-category" }] }],
        "code": { "coding": [{ "code": "10210-3", "system": "http://loinc.org" }], "text": "Physical findings" },
        "subject": { "reference": "Patient/PATIENT_ID" },
        "encounter": { "reference": "urn:uuid:encounter-1" },
        "valueString": "Patient reports headache for 3 days, worse in morning"
      },
      "request": { "method": "POST", "url": "Observation" }
    }
  ]
}
```

🔴 **Use `urn:uuid:` references within transaction Bundles.** The FHIR server resolves them to actual IDs on create. Don't use placeholder IDs.

## Visit Templates

Medplum recommends using `PlanDefinition + Questionnaire + $apply` for structured visit documentation, not raw Bundle creation.

### Resources involved

| Resource | Purpose |
|---|---|
| PlanDefinition | Defines the overall visit type (e.g., "Annual Physical", "Sick Visit") |
| ActivityDefinition | Defines individual activities within the visit |
| Questionnaire | Structured data collection forms |
| ChargeItemDefinition | Billing rules for the visit |

### Creating a visit template

```bash
# PlanDefinition
curl -X POST http://SERVER:8103/fhir/R4/PlanDefinition \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceType": "PlanDefinition",
    "name": "Annual Physical",
    "status": "active",
    "type": { "coding": [{ "code": "clinical-protocol", "system": "http://hl7.org/fhir/plan-definition-type" }] },
    "action": [{
      "title": "Vital Signs",
      "definitionCanonical": "Questionnaire/VITAL_SIGNS_ID"
    }, {
      "title": "Physical Exam",
      "definitionCanonical": "Questionnaire/PHYSICAL_EXAM_ID"
    }]
  }'
```

Then use `$apply` to instantiate from the plan:

```bash
curl -X POST "http://SERVER:8103/fhir/R4/PlanDefinition/PLAN_ID/$apply" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"subject": {"reference": "Patient/PATIENT_ID"}}'
```

## Always Use the FHIR API

🔴 **Never create clinical resources by inserting into PostgreSQL directly.** The FHIR server maintains indexed columns and search parameters that direct DB edits bypass. Use:

```bash
# Create single resource
curl -X POST http://SERVER:8103/fhir/R4/Observation ...

# Create transaction Bundle (multiple resources atomically)
curl -X POST http://SERVER:8103/fhir/R4 -d @bundle.json

# Read resource
curl http://SERVER:8103/fhir/R4/Patient/PATIENT_ID -H "Authorization: Bearer $TOKEN"

# Search resources
curl "http://SERVER:8103/fhir/R4/Patient?name=Smith" -H "Authorization: Bearer $TOKEN"
```

## DoseSpot / E-Prescribing

DoseSpot integration is a **paid Medplum feature**. Self-hosted instances get "Access Check Failed: Forbidden" when attempting to use it. This is expected — it requires a Medplum license and Health Gorilla connector setup.