# Healthcare Provider Integration Workflow Specification

## Purpose & Scope

The Healthcare Provider Integration Workflow enables seamless communication and data exchange between the Personal Health Assistant and healthcare systems using FHIR (Fast Healthcare Interoperability Resources) standards. This workflow facilitates care coordination, provider communication, and integration with Electronic Health Records (EHR) systems.

## Clinical Objectives

- **Seamless Care Coordination**: Enable smooth transitions between the health assistant and healthcare providers
- **Standardized Data Exchange**: Use FHIR R4 standards for interoperability
- **Real-time Provider Communication**: Facilitate immediate provider notifications and consultations
- **EHR Integration**: Synchronize patient data with provider systems
- **Care Continuity**: Maintain comprehensive care records across all touchpoints

## FHIR Resource Integration

### Core FHIR Resources
- **Patient**: Patient demographics and identifiers
- **Practitioner**: Healthcare provider information
- **Organization**: Healthcare facilities and practices
- **Encounter**: Patient-provider interactions
- **Observation**: Clinical measurements and assessments
- **Condition**: Diagnoses and health conditions
- **MedicationRequest**: Prescriptions and medication orders
- **AllergyIntolerance**: Allergy and adverse reaction information
- **CarePlan**: Treatment plans and care coordination
- **Communication**: Provider-patient communications

## Architecture Diagram

```
[Provider Integration Request]
           ↓
    [FHIR Integration Webhook]
           ↓
    [Request Type Classification]
    ├── [Patient Data Sync]
    ├── [Provider Notification]
    ├── [Appointment Scheduling]
    └── [Care Plan Updates]
           ↓
    [FHIR Resource Preparation]
           ↓
    [Provider Authentication & Authorization]
           ↓
    [EHR System API Connection]
           ↓
    [FHIR Bundle Creation]
           ↓
    [Data Validation & Mapping]
           ↓
    [Provider System Integration]
    ├── [Epic/Cerner/Allscripts APIs]
    ├── [Direct FHIR Endpoints]
    └── [HIE (Health Information Exchange)]
           ↓
    [Response Processing & Validation]
           ↓
    [Memory MCP - Integration Logging]
           ↓
    [Provider Response & Status Update]
```

## N8N Workflow Configuration

### Node 1: FHIR Integration Webhook
**Type**: Webhook
**Purpose**: Receive healthcare provider integration requests

**Configuration:**
```json
{
  "httpMethod": "POST",
  "path": "/webhook/provider-integration",
  "authentication": "headerAuth",
  "options": {
    "rawBody": true,
    "cors": {
      "enabled": true,
      "allowedOrigins": ["https://health-assistant.dextechlabs.com"]
    }
  }
}
```

**Expected Input Schema:**
```json
{
  "requestId": "string (UUID)",
  "patientId": "string (UUID)",
  "requestType": "string (patient_sync|provider_notification|appointment_request|care_plan_update|medication_reconciliation)",
  "timestamp": "string (ISO 8601)",
  "priority": "string (low|medium|high|urgent)",
  
  "providerDetails": {
    "practitionerId": "string",
    "organizationId": "string",
    "facilityNPI": "string",
    "practitionerNPI": "string",
    "ehrSystemType": "string (epic|cerner|allscripts|athenahealth|other)",
    "fhirEndpoint": "string (URL)",
    "apiVersion": "string (R4|R5)"
  },
  
  "patientData": {
    "identifiers": {
      "internalId": "string",
      "mrn": "string",
      "insuranceId": "string (optional)"
    },
    "demographics": {
      "firstName": "string",
      "lastName": "string",
      "dateOfBirth": "string",
      "gender": "string",
      "address": "object",
      "phone": "string",
      "email": "string (optional)"
    }
  },
  
  "clinicalData": {
    "currentAssessment": {
      "chiefComplaint": "string",
      "symptoms": ["string"],
      "severity": "number (1-10)",
      "assessment": "object",
      "recommendations": ["string"]
    },
    "conditions": [
      {
        "code": "string (ICD-10)",
        "display": "string",
        "clinicalStatus": "string",
        "verificationStatus": "string",
        "onsetDateTime": "string"
      }
    ],
    "medications": [
      {
        "code": "string (RxNorm)",
        "display": "string",
        "dosage": "string",
        "frequency": "string",
        "status": "string"
      }
    ],
    "allergies": [
      {
        "code": "string (SNOMED)",
        "display": "string",
        "criticality": "string",
        "type": "string"
      }
    ],
    "observations": [
      {
        "code": "string (LOINC)",
        "display": "string",
        "value": "string",
        "unit": "string",
        "effectiveDateTime": "string"
      }
    ]
  },
  
  "communicationRequest": {
    "subject": "string",
    "message": "string",
    "urgency": "string",
    "categoryCode": "string",
    "requestedResponse": "string (acknowledgment|consultation|referral)"
  },
  
  "appointmentRequest": {
    "serviceType": "string",
    "specialty": "string",
    "preferredTimeframe": "string",
    "duration": "number (minutes)",
    "visitType": "string (in_person|telehealth|phone)"
  }
}
```

### Node 2: Request Type Classification
**Type**: Switch
**Purpose**: Route requests based on integration type

**Configuration:**
```json
{
  "mode": "expression",
  "output": "={{ $json.body.requestType }}",
  "rules": {
    "patient_sync": 0,
    "provider_notification": 1,
    "appointment_request": 2,
    "care_plan_update": 3,
    "medication_reconciliation": 4
  }
}
```

### Node 3A: Patient Data Sync Processing
**Type**: Code (JavaScript)
**Purpose**: Prepare patient data for FHIR synchronization

**Implementation:**
```javascript
const integrationRequest = $json.body;
const patientData = integrationRequest.patientData;
const clinicalData = integrationRequest.clinicalData;

// Create FHIR Patient Resource
const fhirPatient = {
  resourceType: "Patient",
  id: patientData.identifiers.internalId,
  identifier: [
    {
      use: "usual",
      type: {
        coding: [
          {
            system: "http://terminology.hl7.org/CodeSystem/v2-0203",
            code: "MR",
            display: "Medical Record Number"
          }
        ]
      },
      system: "https://health-assistant.dextechlabs.com/patient-id",
      value: patientData.identifiers.mrn || patientData.identifiers.internalId
    }
  ],
  active: true,
  name: [
    {
      use: "official",
      family: patientData.demographics.lastName,
      given: [patientData.demographics.firstName]
    }
  ],
  telecom: [
    {
      system: "phone",
      value: patientData.demographics.phone,
      use: "home"
    }
  ],
  gender: patientData.demographics.gender.toLowerCase(),
  birthDate: patientData.demographics.dateOfBirth
};

// Add email if provided
if (patientData.demographics.email) {
  fhirPatient.telecom.push({
    system: "email",
    value: patientData.demographics.email,
    use: "home"
  });
}

// Add address if provided
if (patientData.demographics.address) {
  fhirPatient.address = [
    {
      use: "home",
      type: "both",
      line: [patientData.demographics.address.street],
      city: patientData.demographics.address.city,
      state: patientData.demographics.address.state,
      postalCode: patientData.demographics.address.zipCode,
      country: patientData.demographics.address.country || "US"
    }
  ];
}

// Create FHIR Encounter Resource
const fhirEncounter = {
  resourceType: "Encounter",
  id: `encounter-${integrationRequest.requestId}`,
  status: "finished",
  class: {
    system: "http://terminology.hl7.org/CodeSystem/v3-ActCode",
    code: "VR",
    display: "virtual"
  },
  type: [
    {
      coding: [
        {
          system: "http://snomed.info/sct",
          code: "185317003",
          display: "Telephone consultation"
        }
      ]
    }
  ],
  subject: {
    reference: `Patient/${patientData.identifiers.internalId}`
  },
  period: {
    start: integrationRequest.timestamp,
    end: new Date().toISOString()
  },
  reasonCode: [
    {
      text: clinicalData.currentAssessment.chiefComplaint
    }
  ]
};

// Create FHIR Observation Resources for symptoms and assessments
const fhirObservations = [];

if (clinicalData.currentAssessment.severity) {
  fhirObservations.push({
    resourceType: "Observation",
    id: `pain-severity-${integrationRequest.requestId}`,
    status: "final",
    category: [
      {
        coding: [
          {
            system: "http://terminology.hl7.org/CodeSystem/observation-category",
            code: "survey",
            display: "Survey"
          }
        ]
      }
    ],
    code: {
      coding: [
        {
          system: "http://loinc.org",
          code: "72133-2",
          display: "Pain severity - 0-10 verbal numeric rating"
        }
      ]
    },
    subject: {
      reference: `Patient/${patientData.identifiers.internalId}`
    },
    encounter: {
      reference: `Encounter/encounter-${integrationRequest.requestId}`
    },
    effectiveDateTime: integrationRequest.timestamp,
    valueInteger: clinicalData.currentAssessment.severity
  });
}

// Create FHIR Condition Resources
const fhirConditions = clinicalData.conditions?.map((condition, index) => ({
  resourceType: "Condition",
  id: `condition-${integrationRequest.requestId}-${index}`,
  clinicalStatus: {
    coding: [
      {
        system: "http://terminology.hl7.org/CodeSystem/condition-clinical",
        code: condition.clinicalStatus || "active",
        display: condition.clinicalStatus || "Active"
      }
    ]
  },
  verificationStatus: {
    coding: [
      {
        system: "http://terminology.hl7.org/CodeSystem/condition-ver-status",
        code: condition.verificationStatus || "confirmed",
        display: condition.verificationStatus || "Confirmed"
      }
    ]
  },
  code: {
    coding: [
      {
        system: "http://hl7.org/fhir/sid/icd-10-cm",
        code: condition.code,
        display: condition.display
      }
    ]
  },
  subject: {
    reference: `Patient/${patientData.identifiers.internalId}`
  },
  encounter: {
    reference: `Encounter/encounter-${integrationRequest.requestId}`
  },
  onsetDateTime: condition.onsetDateTime
})) || [];

// Create FHIR MedicationRequest Resources
const fhirMedicationRequests = clinicalData.medications?.map((medication, index) => ({
  resourceType: "MedicationRequest",
  id: `medication-${integrationRequest.requestId}-${index}`,
  status: medication.status || "active",
  intent: "order",
  medicationCodeableConcept: {
    coding: [
      {
        system: "http://www.nlm.nih.gov/research/umls/rxnorm",
        code: medication.code,
        display: medication.display
      }
    ]
  },
  subject: {
    reference: `Patient/${patientData.identifiers.internalId}`
  },
  encounter: {
    reference: `Encounter/encounter-${integrationRequest.requestId}`
  },
  authoredOn: integrationRequest.timestamp,
  dosageInstruction: [
    {
      text: `${medication.dosage} ${medication.frequency}`,
      timing: {
        repeat: {
          frequency: 1,
          period: 1,
          periodUnit: "d"
        }
      }
    }
  ]
})) || [];

// Create FHIR AllergyIntolerance Resources
const fhirAllergies = clinicalData.allergies?.map((allergy, index) => ({
  resourceType: "AllergyIntolerance",
  id: `allergy-${integrationRequest.requestId}-${index}`,
  clinicalStatus: {
    coding: [
      {
        system: "http://terminology.hl7.org/CodeSystem/allergyintolerance-clinical",
        code: "active",
        display: "Active"
      }
    ]
  },
  verificationStatus: {
    coding: [
      {
        system: "http://terminology.hl7.org/CodeSystem/allergyintolerance-verification",
        code: "confirmed",
        display: "Confirmed"
      }
    ]
  },
  type: allergy.type || "allergy",
  criticality: allergy.criticality || "low",
  code: {
    coding: [
      {
        system: "http://snomed.info/sct",
        code: allergy.code,
        display: allergy.display
      }
    ]
  },
  patient: {
    reference: `Patient/${patientData.identifiers.internalId}`
  }
})) || [];

return {
  ...integrationRequest,
  fhirBundle: {
    resourceType: "Bundle",
    id: `health-assistant-sync-${integrationRequest.requestId}`,
    type: "transaction",
    timestamp: new Date().toISOString(),
    entry: [
      { resource: fhirPatient, request: { method: "PUT", url: `Patient/${fhirPatient.id}` } },
      { resource: fhirEncounter, request: { method: "POST", url: "Encounter" } },
      ...fhirObservations.map(obs => ({ resource: obs, request: { method: "POST", url: "Observation" } })),
      ...fhirConditions.map(cond => ({ resource: cond, request: { method: "POST", url: "Condition" } })),
      ...fhirMedicationRequests.map(med => ({ resource: med, request: { method: "POST", url: "MedicationRequest" } })),
      ...fhirAllergies.map(allergy => ({ resource: allergy, request: { method: "POST", url: "AllergyIntolerance" } }))
    ]
  },
  integrationTimestamp: new Date().toISOString()
};
```

### Node 4: Provider Authentication & Authorization
**Type**: HTTP Request
**Purpose**: Authenticate with provider EHR system

**Configuration:**
```json
{
  "method": "POST",
  "url": "https://{{ $json.providerDetails.ehrSystemType }}-auth.healthsystem.com/oauth2/token",
  "authentication": "genericCredentialType",
  "sendHeaders": true,
  "headerParameters": {
    "parameters": [
      {
        "name": "Content-Type",
        "value": "application/x-www-form-urlencoded"
      }
    ]
  },
  "sendBody": true,
  "bodyParameters": {
    "parameters": [
      {
        "name": "grant_type",
        "value": "client_credentials"
      },
      {
        "name": "client_id",
        "value": "={{ $credentials.ehrIntegration.clientId }}"
      },
      {
        "name": "client_secret",
        "value": "={{ $credentials.ehrIntegration.clientSecret }}"
      },
      {
        "name": "scope",
        "value": "patient/*.read patient/*.write"
      }
    ]
  }
}
```

### Node 5: EHR System API Connection
**Type**: HTTP Request
**Purpose**: Send FHIR bundle to provider system

**Configuration:**
```json
{
  "method": "POST",
  "url": "={{ $json.providerDetails.fhirEndpoint }}",
  "sendHeaders": true,
  "headerParameters": {
    "parameters": [
      {
        "name": "Content-Type",
        "value": "application/fhir+json"
      },
      {
        "name": "Authorization",
        "value": "Bearer {{ $json.access_token }}"
      },
      {
        "name": "Accept",
        "value": "application/fhir+json"
      }
    ]
  },
  "sendBody": true,
  "bodyContent": "={{ JSON.stringify($json.fhirBundle) }}"
}
```

### Node 6: Response Processing & Validation
**Type**: Code (JavaScript)
**Purpose**: Process and validate provider system response

**Implementation:**
```javascript
const integrationRequest = $json;
const fhirResponse = $json.response || {};

const validation = {
  success: false,
  fhirValidation: {
    bundleProcessed: false,
    resourcesCreated: 0,
    resourcesUpdated: 0,
    errors: [],
    warnings: []
  },
  providerIntegration: {
    system: integrationRequest.providerDetails.ehrSystemType,
    endpoint: integrationRequest.providerDetails.fhirEndpoint,
    responseTime: 0,
    status: 'unknown'
  },
  dataMapping: {
    patientMapped: false,
    conditionsMapped: 0,
    medicationsMapped: 0,
    allergiesMapped: 0
  }
};

// Validate FHIR response structure
if (fhirResponse.resourceType === 'Bundle') {
  validation.fhirValidation.bundleProcessed = true;
  
  if (fhirResponse.entry && Array.isArray(fhirResponse.entry)) {
    fhirResponse.entry.forEach(entry => {
      if (entry.response) {
        if (entry.response.status && entry.response.status.startsWith('2')) {
          if (entry.response.status.includes('201')) {
            validation.fhirValidation.resourcesCreated++;
          } else if (entry.response.status.includes('200')) {
            validation.fhirValidation.resourcesUpdated++;
          }
        } else {
          validation.fhirValidation.errors.push({
            resource: entry.resource?.resourceType || 'unknown',
            status: entry.response.status,
            details: entry.response.outcome
          });
        }
      }
    });
  }
} else if (fhirResponse.resourceType === 'OperationOutcome') {
  // Handle operation outcome
  if (fhirResponse.issue) {
    fhirResponse.issue.forEach(issue => {
      if (issue.severity === 'error' || issue.severity === 'fatal') {
        validation.fhirValidation.errors.push({
          severity: issue.severity,
          code: issue.code,
          details: issue.details?.text || issue.diagnostics
        });
      } else {
        validation.fhirValidation.warnings.push({
          severity: issue.severity,
          code: issue.code,
          details: issue.details?.text || issue.diagnostics
        });
      }
    });
  }
}

// Determine overall success
if (validation.fhirValidation.bundleProcessed && 
    validation.fhirValidation.errors.length === 0 &&
    (validation.fhirValidation.resourcesCreated > 0 || validation.fhirValidation.resourcesUpdated > 0)) {
  validation.success = true;
  validation.providerIntegration.status = 'success';
} else if (validation.fhirValidation.errors.length > 0) {
  validation.providerIntegration.status = 'error';
} else {
  validation.providerIntegration.status = 'partial';
}

// Map data processing results
const originalBundle = integrationRequest.fhirBundle;
if (originalBundle && originalBundle.entry) {
  originalBundle.entry.forEach(entry => {
    switch (entry.resource.resourceType) {
      case 'Patient':
        validation.dataMapping.patientMapped = true;
        break;
      case 'Condition':
        validation.dataMapping.conditionsMapped++;
        break;
      case 'MedicationRequest':
        validation.dataMapping.medicationsMapped++;
        break;
      case 'AllergyIntolerance':
        validation.dataMapping.allergiesMapped++;
        break;
    }
  });
}

// Generate integration summary
const integrationSummary = {
  requestId: integrationRequest.requestId,
  patientId: integrationRequest.patientId,
  requestType: integrationRequest.requestType,
  providerSystem: integrationRequest.providerDetails.ehrSystemType,
  processingResult: validation.success ? 'SUCCESS' : 'FAILED',
  resourcesSynchronized: validation.fhirValidation.resourcesCreated + validation.fhirValidation.resourcesUpdated,
  errorsEncountered: validation.fhirValidation.errors.length,
  warningsGenerated: validation.fhirValidation.warnings.length,
  completedAt: new Date().toISOString()
};

return {
  ...integrationRequest,
  validation,
  integrationSummary,
  fhirResponse,
  processingComplete: true
};
```

### Node 7: Communication Resource Creation (for provider notifications)
**Type**: Code (JavaScript)
**Purpose**: Create FHIR Communication resource for provider notifications

**Implementation:**
```javascript
const integrationRequest = $json;
const communicationRequest = integrationRequest.communicationRequest;

if (integrationRequest.requestType === 'provider_notification' && communicationRequest) {
  const fhirCommunication = {
    resourceType: "Communication",
    id: `communication-${integrationRequest.requestId}`,
    status: "completed",
    category: [
      {
        coding: [
          {
            system: "http://terminology.hl7.org/CodeSystem/communication-category",
            code: communicationRequest.categoryCode || "notification",
            display: "Notification"
          }
        ]
      }
    ],
    priority: integrationRequest.priority || "routine",
    subject: {
      reference: `Patient/${integrationRequest.patientData.identifiers.internalId}`
    },
    recipient: [
      {
        reference: `Practitioner/${integrationRequest.providerDetails.practitionerId}`
      }
    ],
    sender: {
      reference: "Organization/health-assistant"
    },
    sent: new Date().toISOString(),
    payload: [
      {
        contentString: communicationRequest.message
      },
      {
        contentString: `Subject: ${communicationRequest.subject}`
      }
    ],
    note: [
      {
        text: `Generated by Personal Health Assistant for patient assessment`,
        time: new Date().toISOString()
      }
    ]
  };

  return {
    ...integrationRequest,
    fhirCommunication,
    communicationCreated: true
  };
}

return {
  ...integrationRequest,
  communicationCreated: false
};
```

### Node 8: Appointment Request Processing
**Type**: HTTP Request
**Purpose**: Create appointment request via FHIR Appointment resource

**Configuration:**
```json
{
  "method": "POST",
  "url": "={{ $json.providerDetails.fhirEndpoint }}/Appointment",
  "sendHeaders": true,
  "headerParameters": {
    "parameters": [
      {
        "name": "Content-Type",
        "value": "application/fhir+json"
      },
      {
        "name": "Authorization",
        "value": "Bearer {{ $json.access_token }}"
      }
    ]
  },
  "sendBody": true,
  "bodyContent": "{\n  \"resourceType\": \"Appointment\",\n  \"id\": \"appointment-{{ $json.requestId }}\",\n  \"status\": \"proposed\",\n  \"serviceType\": [\n    {\n      \"coding\": [\n        {\n          \"system\": \"http://terminology.hl7.org/CodeSystem/service-type\",\n          \"code\": \"{{ $json.appointmentRequest.serviceType }}\",\n          \"display\": \"{{ $json.appointmentRequest.serviceType }}\"\n        }\n      ]\n    }\n  ],\n  \"specialty\": [\n    {\n      \"coding\": [\n        {\n          \"system\": \"http://snomed.info/sct\",\n          \"code\": \"394802001\",\n          \"display\": \"General medicine\"\n        }\n      ]\n    }\n  ],\n  \"appointmentType\": {\n    \"coding\": [\n      {\n        \"system\": \"http://terminology.hl7.org/CodeSystem/v2-0276\",\n        \"code\": \"ROUTINE\",\n        \"display\": \"Routine appointment\"\n      }\n    ]\n  },\n  \"reasonCode\": [\n    {\n      \"text\": \"{{ $json.clinicalData.currentAssessment.chiefComplaint }}\"\n    }\n  ],\n  \"priority\": {{ $json.priority === 'urgent' ? 1 : 5 }},\n  \"minutesDuration\": {{ $json.appointmentRequest.duration || 30 }},\n  \"participant\": [\n    {\n      \"actor\": {\n        \"reference\": \"Patient/{{ $json.patientData.identifiers.internalId }}\"\n      },\n      \"required\": \"required\",\n      \"status\": \"needs-action\"\n    },\n    {\n      \"actor\": {\n        \"reference\": \"Practitioner/{{ $json.providerDetails.practitionerId }}\"\n      },\n      \"required\": \"required\",\n      \"status\": \"needs-action\"\n    }\n  ],\n  \"requestedPeriod\": [\n    {\n      \"start\": \"{{ $json.appointmentRequest.preferredTimeframe }}\"\n    }\n  ]\n}"
}
```

### Node 9: Memory MCP - Integration Logging
**Type**: Memory MCP
**Purpose**: Log integration events and outcomes

**Configuration:**
```json
{
  "server": "memory",
  "tool": "create_entities",
  "parameters": {
    "entities": [
      {
        "name": "{{ $json.patientId }}_provider_integration_{{ $json.requestId }}",
        "entityType": "provider_integration",
        "observations": [
          "Request ID: {{ $json.requestId }}",
          "Request type: {{ $json.requestType }}",
          "Provider system: {{ $json.providerDetails.ehrSystemType }}",
          "Integration status: {{ $json.integrationSummary.processingResult }}",
          "Resources synchronized: {{ $json.integrationSummary.resourcesSynchronized }}",
          "Errors encountered: {{ $json.integrationSummary.errorsEncountered }}",
          "FHIR endpoint: {{ $json.providerDetails.fhirEndpoint }}",
          "Completed at: {{ $json.integrationSummary.completedAt }}",
          "Patient mapped: {{ $json.validation.dataMapping.patientMapped }}",
          "Conditions mapped: {{ $json.validation.dataMapping.conditionsMapped }}"
        ]
      }
    ]
  }
}
```

### Node 10: Provider Response & Status Update
**Type**: Webhook Response
**Purpose**: Return integration results to requesting system

**Configuration:**
```json
{
  "respondWith": "json",
  "responseBody": {
    "success": "={{ $json.validation.success }}",
    "requestId": "={{ $json.requestId }}",
    "patientId": "={{ $json.patientId }}",
    "integrationCompleted": true,
    
    "providerIntegration": {
      "system": "={{ $json.providerDetails.ehrSystemType }}",
      "status": "={{ $json.validation.providerIntegration.status }}",
      "endpoint": "={{ $json.providerDetails.fhirEndpoint }}",
      "apiVersion": "={{ $json.providerDetails.apiVersion }}"
    },
    
    "dataSync": {
      "fhirBundleProcessed": "={{ $json.validation.fhirValidation.bundleProcessed }}",
      "resourcesCreated": "={{ $json.validation.fhirValidation.resourcesCreated }}",
      "resourcesUpdated": "={{ $json.validation.fhirValidation.resourcesUpdated }}",
      "patientMapped": "={{ $json.validation.dataMapping.patientMapped }}",
      "conditionsMapped": "={{ $json.validation.dataMapping.conditionsMapped }}",
      "medicationsMapped": "={{ $json.validation.dataMapping.medicationsMapped }}",
      "allergiesMapped": "={{ $json.validation.dataMapping.allergiesMapped }}"
    },
    
    "communicationResults": {
      "communicationCreated": "={{ $json.communicationCreated || false }}",
      "appointmentRequested": "={{ $json.requestType === 'appointment_request' }}"
    },
    
    "integrationSummary": "={{ $json.integrationSummary }}",
    
    "errors": "={{ $json.validation.fhirValidation.errors }}",
    "warnings": "={{ $json.validation.fhirValidation.warnings }}",
    
    "metadata": {
      "processingTime": "calculated_in_seconds",
      "fhirVersion": "R4",
      "integrationTimestamp": "={{ $json.integrationTimestamp }}",
      "completedAt": "={{ $json.integrationSummary.completedAt }}"
    }
  },
  "responseCode": 200
}
```

## Error Handling & Resilience

### Authentication Failures
```javascript
const authRetryConfig = {
  maxRetries: 3,
  backoffStrategy: 'exponential',
  initialDelay: 2000,
  authTokenCaching: true,
  tokenRefreshThreshold: 300 // 5 minutes before expiry
};
```

### FHIR Validation Errors
- Resource validation against FHIR R4 specification
- Terminology binding validation
- Reference integrity checking
- Graceful degradation for non-critical errors

### Network Resilience
- Circuit breaker pattern for provider API calls
- Automatic retry with exponential backoff
- Timeout management (30 seconds for FHIR calls)
- Fallback to alternative endpoints when available

## Security & Compliance

### OAuth 2.0/OIDC Integration
- Secure client credentials management
- Token lifecycle management
- Scope-based access control
- Audit logging for all API calls

### FHIR Security
- TLS 1.3 encryption for all communications
- SMART on FHIR compliance
- Role-based access control (RBAC)
- Patient consent verification

### Data Privacy
- PHI handling per HIPAA requirements
- Data minimization principles
- Purpose limitation enforcement
- Retention policy compliance

## Performance & Monitoring

### Response Time Targets
- Authentication: <2 seconds
- FHIR bundle processing: <10 seconds
- Provider API integration: <15 seconds
- Complete workflow: <30 seconds

### Monitoring Metrics
- Integration success rates by provider
- FHIR resource processing times
- Authentication token refresh rates
- Error rates and types by EHR system

This healthcare provider integration workflow ensures seamless, secure, and standardized communication with healthcare systems while maintaining full FHIR compliance and robust error handling.