# Patient Intake Workflow Specification

## Purpose & Scope

The Patient Intake Workflow serves as the primary entry point for all patient interactions with the Personal Health Assistant. It handles initial patient assessment, symptom collection, basic triage, and routing to appropriate care pathways while ensuring patient safety and clinical accuracy.

## Clinical Objectives

- **Primary Triage**: Rapid assessment and risk stratification
- **Symptom Collection**: Structured clinical interview process
- **Safety Screening**: Emergency condition detection
- **Care Routing**: Appropriate pathway determination
- **Context Establishment**: Patient history and preference integration

## Architecture Diagram

```
[Mobile App/Web Interface]
           ↓
    [Webhook Trigger]
           ↓
    [Input Validation]
           ↓
    [Patient Authentication]
           ↓
    [Emergency Screening] → [Emergency Workflow] (if critical)
           ↓
    [Symptom Collection]
           ↓
    [Memory MCP - History Retrieval]
           ↓
    [Sequential Thinking - Initial Assessment]
           ↓
    [BioMCP - Symptom Analysis]
           ↓
    [Risk Stratification]
           ↓
    [Care Pathway Routing]
           ↓
    [Response Generation]
           ↓
    [Memory MCP - Session Storage]
           ↓
    [Mobile App Response]
```

## N8N Workflow Configuration

### Node 1: Webhook Trigger
**Type**: Webhook
**Purpose**: Receive patient intake requests from mobile application

**Configuration:**
```json
{
  "httpMethod": "POST",
  "path": "/webhook/patient-intake",
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
  "patientId": "string (UUID)",
  "sessionId": "string (UUID)",
  "chiefComplaint": "string",
  "symptoms": {
    "primary": "string",
    "onset": "string",
    "duration": "string",
    "severity": "number (1-10)",
    "location": "string",
    "quality": "string",
    "associatedSymptoms": ["string"]
  },
  "demographics": {
    "age": "number",
    "gender": "string",
    "weight": "number (optional)",
    "height": "number (optional)"
  },
  "vitals": {
    "temperature": "number (optional)",
    "heartRate": "number (optional)",
    "bloodPressure": "string (optional)",
    "respiratoryRate": "number (optional)"
  },
  "emergencyContact": {
    "name": "string",
    "phone": "string",
    "relationship": "string"
  }
}
```

### Node 2: Input Validation & Sanitization
**Type**: Code (JavaScript)
**Purpose**: Validate and sanitize incoming patient data

**Implementation:**
```javascript
const inputData = $json.body;

// Validation schema
const requiredFields = ['patientId', 'sessionId', 'chiefComplaint'];
const validation = {
  isValid: true,
  errors: [],
  sanitizedData: {}
};

// Check required fields
for (const field of requiredFields) {
  if (!inputData[field]) {
    validation.isValid = false;
    validation.errors.push(`Missing required field: ${field}`);
  }
}

// Sanitize text inputs
if (inputData.chiefComplaint) {
  validation.sanitizedData.chiefComplaint = inputData.chiefComplaint
    .trim()
    .substring(0, 1000) // Limit length
    .replace(/[<>]/g, ''); // Basic XSS prevention
}

// Validate severity score
if (inputData.symptoms?.severity) {
  const severity = parseInt(inputData.symptoms.severity);
  if (severity < 1 || severity > 10) {
    validation.errors.push('Severity must be between 1 and 10');
  } else {
    validation.sanitizedData.severity = severity;
  }
}

// Validate age
if (inputData.demographics?.age) {
  const age = parseInt(inputData.demographics.age);
  if (age < 0 || age > 150) {
    validation.errors.push('Invalid age provided');
  } else {
    validation.sanitizedData.age = age;
  }
}

return {
  validation,
  inputData: validation.isValid ? { ...inputData, ...validation.sanitizedData } : null,
  timestamp: new Date().toISOString()
};
```

### Node 3: Patient Authentication & Context
**Type**: HTTP Request
**Purpose**: Verify patient identity and retrieve session context

**Configuration:**
```json
{
  "method": "POST",
  "url": "https://auth.dextechlabs.com/verify-patient",
  "authentication": "genericCredentialType",
  "headers": {
    "Content-Type": "application/json",
    "Authorization": "Bearer {{ $credentials.authService.apiKey }}"
  },
  "body": {
    "patientId": "={{ $json.inputData.patientId }}",
    "sessionId": "={{ $json.inputData.sessionId }}"
  }
}
```

### Node 4: Emergency Screening
**Type**: Code (JavaScript)
**Purpose**: Immediate screening for emergency conditions

**Implementation:**
```javascript
const patientData = $json.inputData;
const emergencyKeywords = [
  'chest pain', 'can\'t breathe', 'unconscious', 'bleeding heavily',
  'severe pain', 'suicidal', 'overdose', 'allergic reaction',
  'stroke symptoms', 'heart attack', 'choking', 'severe burns'
];

const emergencyVitals = {
  temperature: { critical: temp => temp > 104 || temp < 95 },
  heartRate: { critical: hr => hr > 120 || hr < 50 },
  systolicBP: { critical: sbp => sbp > 180 || sbp < 90 },
  respiratoryRate: { critical: rr => rr > 30 || rr < 8 }
};

let emergencyFlags = {
  isEmergency: false,
  urgencyLevel: 'routine', // routine, urgent, emergency
  triggers: [],
  recommendedAction: 'continue_assessment'
};

// Check chief complaint for emergency keywords
const complaint = patientData.chiefComplaint?.toLowerCase() || '';
for (const keyword of emergencyKeywords) {
  if (complaint.includes(keyword)) {
    emergencyFlags.isEmergency = true;
    emergencyFlags.urgencyLevel = 'emergency';
    emergencyFlags.triggers.push(`Emergency keyword: ${keyword}`);
    emergencyFlags.recommendedAction = 'immediate_escalation';
  }
}

// Check severity score
if (patientData.symptoms?.severity >= 8) {
  emergencyFlags.urgencyLevel = emergencyFlags.urgencyLevel === 'routine' ? 'urgent' : emergencyFlags.urgencyLevel;
  emergencyFlags.triggers.push(`High severity score: ${patientData.symptoms.severity}`);
}

// Check vital signs if provided
if (patientData.vitals) {
  for (const [vital, value] of Object.entries(patientData.vitals)) {
    if (emergencyVitals[vital]?.critical(value)) {
      emergencyFlags.isEmergency = true;
      emergencyFlags.urgencyLevel = 'emergency';
      emergencyFlags.triggers.push(`Critical vital sign: ${vital} = ${value}`);
      emergencyFlags.recommendedAction = 'immediate_escalation';
    }
  }
}

return {
  ...patientData,
  emergencyAssessment: emergencyFlags,
  timestamp: new Date().toISOString()
};
```

### Node 5: Memory MCP - Patient History Retrieval
**Type**: MCP Client Tool
**Purpose**: Retrieve patient's medical history and previous interactions

**Configuration:**
```json
{
  "server": "memory",
  "tool": "search_nodes",
  "parameters": {
    "query": "={{ $json.inputData.patientId }}"
  }
}
```

### Node 6: Sequential Thinking - Clinical Reasoning
**Type**: MCP Client Tool
**Purpose**: Apply structured clinical reasoning to patient presentation

**Configuration:**
```json
{
  "server": "sequential-thinking",
  "tool": "sequentialthinking",
  "parameters": {
    "thought": "Analyzing patient presentation: Chief complaint is '{{ $json.inputData.chiefComplaint }}' with severity {{ $json.symptoms?.severity || 'not specified' }}. Patient is {{ $json.demographics?.age || 'age unknown' }} years old. Need to consider differential diagnosis, red flags, and appropriate triage level.",
    "thoughtNumber": 1,
    "totalThoughts": 5,
    "nextThoughtNeeded": true
  }
}
```

### Node 7: BioMCP - Symptom Analysis
**Type**: MCP Client Tool
**Purpose**: Look up medical information related to patient's symptoms

**Configuration:**
```json
{
  "server": "biomcp",
  "tool": "search",
  "parameters": {
    "query": "symptoms: {{ $json.inputData.chiefComplaint }}",
    "domain": "article",
    "page_size": 10,
    "call_benefit": "Analyzing patient symptoms for evidence-based triage and care recommendations"
  }
}
```

### Node 8: Risk Stratification
**Type**: Code (JavaScript)
**Purpose**: Calculate patient risk score and determine care pathway

**Implementation:**
```javascript
const patientData = $json.inputData;
const emergencyAssessment = $json.emergencyAssessment;
const clinicalReasoning = $json.sequentialThinking;
const medicalEvidence = $json.bioMCPResults;

// Calculate risk score (0-100)
let riskScore = 0;
let riskFactors = [];

// Age-based risk
if (patientData.demographics?.age) {
  if (patientData.demographics.age > 65) {
    riskScore += 15;
    riskFactors.push('Advanced age (>65)');
  } else if (patientData.demographics.age < 2) {
    riskScore += 20;
    riskFactors.push('Pediatric patient (<2 years)');
  }
}

// Severity-based risk
if (patientData.symptoms?.severity) {
  riskScore += patientData.symptoms.severity * 5;
  if (patientData.symptoms.severity >= 7) {
    riskFactors.push(`High pain/severity score: ${patientData.symptoms.severity}`);
  }
}

// Emergency flags
if (emergencyAssessment.isEmergency) {
  riskScore += 40;
  riskFactors.push('Emergency condition identified');
}

// Vital sign abnormalities
if (patientData.vitals) {
  let vitalAbnormalities = 0;
  if (patientData.vitals.temperature > 101.3 || patientData.vitals.temperature < 96.8) {
    vitalAbnormalities++;
    riskFactors.push('Abnormal temperature');
  }
  if (patientData.vitals.heartRate > 100 || patientData.vitals.heartRate < 60) {
    vitalAbnormalities++;
    riskFactors.push('Abnormal heart rate');
  }
  riskScore += vitalAbnormalities * 10;
}

// Determine care pathway
let carePathway = 'self-care';
let recommendedTimeframe = '1-2 weeks';
let providerType = 'primary-care';

if (riskScore >= 70 || emergencyAssessment.isEmergency) {
  carePathway = 'emergency';
  recommendedTimeframe = 'immediately';
  providerType = 'emergency-department';
} else if (riskScore >= 40) {
  carePathway = 'urgent';
  recommendedTimeframe = 'within 24 hours';
  providerType = 'urgent-care';
} else if (riskScore >= 20) {
  carePathway = 'routine';
  recommendedTimeframe = 'within 1-3 days';
  providerType = 'primary-care';
}

return {
  ...patientData,
  riskAssessment: {
    riskScore,
    riskFactors,
    carePathway,
    recommendedTimeframe,
    providerType,
    assessmentDate: new Date().toISOString()
  },
  clinicalSummary: {
    chiefComplaint: patientData.chiefComplaint,
    urgencyLevel: emergencyAssessment.urgencyLevel,
    recommendedAction: carePathway,
    keyFindings: riskFactors
  }
};
```

### Node 9: Response Generation
**Type**: Code (JavaScript)
**Purpose**: Generate patient-friendly response with care recommendations

**Implementation:**
```javascript
const assessment = $json.riskAssessment;
const patientData = $json.inputData;
const emergencyFlags = $json.emergencyAssessment;

let response = {
  success: true,
  patientId: patientData.patientId,
  sessionId: patientData.sessionId,
  assessmentComplete: true,
  assessment: {
    urgencyLevel: assessment.carePathway,
    riskScore: assessment.riskScore,
    timeframe: assessment.recommendedTimeframe
  },
  recommendations: [],
  nextSteps: [],
  emergencyInstructions: null,
  disclaimers: [
    "This assessment is for informational purposes only and does not replace professional medical advice.",
    "If you are experiencing a medical emergency, call 911 immediately.",
    "Please consult with a healthcare provider for proper medical evaluation and treatment."
  ]
};

// Generate care recommendations based on pathway
switch (assessment.carePathway) {
  case 'emergency':
    response.recommendations = [
      "Seek immediate emergency medical attention",
      "Call 911 or go to the nearest emergency department",
      "Do not drive yourself - call emergency services or have someone drive you"
    ];
    response.emergencyInstructions = {
      callEmergency: true,
      findNearestER: true,
      doNotDelay: true
    };
    break;
    
  case 'urgent':
    response.recommendations = [
      "Seek medical attention within 24 hours",
      "Contact your primary care provider or urgent care center",
      "Monitor symptoms and seek emergency care if they worsen"
    ];
    response.nextSteps = [
      "Schedule appointment with healthcare provider",
      "Monitor symptoms closely",
      "Prepare list of current medications and allergies"
    ];
    break;
    
  case 'routine':
    response.recommendations = [
      "Schedule routine appointment with your healthcare provider",
      "Continue monitoring your symptoms",
      "Consider appropriate self-care measures"
    ];
    response.nextSteps = [
      "Schedule non-urgent medical appointment",
      "Track symptoms in health diary",
      "Follow up in 1-2 weeks if symptoms persist"
    ];
    break;
    
  case 'self-care':
    response.recommendations = [
      "Your symptoms appear manageable with self-care",
      "Monitor symptoms and seek care if they worsen",
      "Consider over-the-counter remedies if appropriate"
    ];
    response.nextSteps = [
      "Continue self-monitoring",
      "Schedule routine check-up if due",
      "Contact healthcare provider if symptoms change"
    ];
    break;
}

// Add specific instructions based on symptoms
if (patientData.symptoms?.severity >= 7) {
  response.nextSteps.unshift("Pain management consultation may be beneficial");
}

return response;
```

### Node 10: Memory MCP - Session Storage
**Type**: MCP Client Tool
**Purpose**: Store assessment results and patient interaction history

**Configuration:**
```json
{
  "server": "memory",
  "tool": "create_entities",
  "parameters": {
    "entities": [
      {
        "name": "{{ $json.patientId }}_assessment_{{ $json.sessionId }}",
        "entityType": "clinical_assessment",
        "observations": [
          "Chief complaint: {{ $json.inputData.chiefComplaint }}",
          "Risk score: {{ $json.riskAssessment.riskScore }}",
          "Care pathway: {{ $json.riskAssessment.carePathway }}",
          "Assessment date: {{ $json.riskAssessment.assessmentDate }}",
          "Recommended timeframe: {{ $json.riskAssessment.recommendedTimeframe }}"
        ]
      }
    ]
  }
}
```

## Error Handling & Recovery

### Error Scenarios
1. **Invalid Input**: Return structured error response with validation details
2. **MCP Server Unavailable**: Fallback to basic assessment without external data
3. **Authentication Failure**: Redirect to login with session preservation
4. **Emergency Detection Error**: Default to urgent care recommendation

### Retry Logic
```javascript
// Retry configuration for MCP calls
const retryConfig = {
  maxAttempts: 3,
  backoffStrategy: 'exponential',
  initialDelay: 1000,
  maxDelay: 5000
};
```

## Security Measures

### Data Encryption
- All patient data encrypted in transit (TLS 1.3)
- Sensitive fields encrypted at rest
- Session tokens with 30-minute expiration

### Access Control
- Patient authentication required
- Rate limiting: 10 requests per minute per patient
- Geographic IP restrictions (configurable)

### Audit Logging
```json
{
  "eventType": "patient_intake",
  "patientId": "hashed_patient_id",
  "sessionId": "session_uuid",
  "timestamp": "ISO_8601_timestamp",
  "riskScore": "calculated_score",
  "carePathway": "determined_pathway",
  "emergencyFlags": "any_emergency_triggers"
}
```

## Testing Scenarios

### Test Case 1: Emergency Scenario
**Input**: Chest pain, severity 9, sweating, shortness of breath
**Expected**: Emergency pathway, <30 second response, 911 recommendation

### Test Case 2: Urgent Scenario  
**Input**: Severe headache, severity 8, visual changes
**Expected**: Urgent pathway, same-day care recommendation

### Test Case 3: Routine Scenario
**Input**: Mild cough, 3 days duration, no fever
**Expected**: Routine pathway, primary care recommendation

### Test Case 4: Self-Care Scenario
**Input**: Minor cuts, first aid applied, healing well
**Expected**: Self-care pathway, monitoring instructions

## Performance Metrics

### Target Response Times
- Total workflow execution: <2 minutes
- Emergency detection: <30 seconds
- MCP tool responses: <10 seconds each
- Memory operations: <5 seconds

### Success Criteria
- 99.9% uptime during business hours
- <1% false positive emergency detection
- >95% patient satisfaction with recommendations
- Zero data breaches or privacy violations

## Deployment Instructions

1. **Import Workflow**: Import JSON file into N8N instance
2. **Configure Credentials**: Set up MCP server authentication
3. **Test Endpoints**: Validate webhook connectivity
4. **Configure Monitoring**: Set up alerting for failures
5. **Clinical Validation**: Test with healthcare professionals
6. **Go-Live Checklist**: Complete pre-deployment verification

This patient intake workflow ensures safe, efficient, and clinically appropriate initial assessment of all patients while maintaining the highest standards of healthcare quality and patient safety.