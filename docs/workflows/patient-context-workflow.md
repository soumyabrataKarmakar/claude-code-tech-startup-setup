# Patient Context Management Workflow Specification

## Purpose & Scope

The Patient Context Management Workflow provides comprehensive patient data management, enabling personalized healthcare interactions through persistent context storage, medical history tracking, and intelligent data retrieval. This workflow leverages Memory MCP to maintain continuous patient relationships and provide contextual healthcare assistance.

## Clinical Objectives

- **Comprehensive Patient Profiles**: Maintain complete medical histories and preferences
- **Contextual Care**: Provide personalized recommendations based on patient history
- **Care Continuity**: Track patient interactions across multiple sessions and providers
- **Data Integration**: Unify patient data from multiple sources and interactions
- **Privacy Compliance**: Secure handling of protected health information (PHI)

## Architecture Diagram

```
[Patient Data Input]
           ↓
    [Context Management Webhook]
           ↓
    [Data Classification & Validation]
           ↓
    [Patient Identity Resolution]
           ↓
    [Memory MCP - Historical Data Retrieval]
           ↓
    [Data Processing & Enrichment]
    ├── [Medical History Analysis]
    ├── [Medication Timeline Tracking]
    ├── [Symptom Pattern Recognition]
    └── [Care Plan Integration]
           ↓
    [Context Synthesis & Update]
           ↓
    [Memory MCP - Data Storage]
           ↓
    [Relationship Mapping]
           ↓
    [Intelligent Insights Generation]
           ↓
    [Context Response & Recommendations]
```

## Data Model Structure

### Core Patient Entity
```json
{
  "entityType": "patient_profile",
  "name": "patient_{uuid}",
  "demographics": {
    "age": "number",
    "gender": "string",
    "dateOfBirth": "string",
    "location": "object"
  },
  "medicalIdentifiers": {
    "internalId": "string",
    "mrn": "string (optional)",
    "insuranceId": "string (optional)"
  }
}
```

### Medical History Entities
```json
{
  "entityType": "medical_condition",
  "name": "condition_{uuid}",
  "conditionDetails": {
    "diagnosis": "string",
    "icd10Code": "string",
    "onsetDate": "string",
    "status": "active|resolved|chronic",
    "severity": "mild|moderate|severe"
  }
}
```

### Medication Entities
```json
{
  "entityType": "medication_record",
  "name": "medication_{uuid}",
  "medicationDetails": {
    "name": "string",
    "genericName": "string",
    "dosage": "string",
    "frequency": "string",
    "startDate": "string",
    "endDate": "string (optional)",
    "prescribingProvider": "string"
  }
}
```

### Interaction History
```json
{
  "entityType": "healthcare_interaction",
  "name": "interaction_{uuid}",
  "interactionDetails": {
    "type": "assessment|consultation|emergency|follow_up",
    "chiefComplaint": "string",
    "assessment": "object",
    "recommendations": "array",
    "provider": "string",
    "timestamp": "string"
  }
}
```

## N8N Workflow Configuration

### Node 1: Context Management Webhook
**Type**: Webhook
**Purpose**: Receive patient context management requests

**Configuration:**
```json
{
  "httpMethod": "POST",
  "path": "/webhook/patient-context",
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
  "requestType": "string (create|update|retrieve|merge|analyze)",
  "timestamp": "string (ISO 8601)",
  
  "patientData": {
    "demographics": {
      "age": "number",
      "gender": "string",
      "dateOfBirth": "string (optional)",
      "location": {
        "city": "string",
        "state": "string",
        "zipCode": "string"
      }
    },
    "identifiers": {
      "mrn": "string (optional)",
      "insuranceId": "string (optional)",
      "externalIds": ["string"]
    }
  },
  
  "medicalData": {
    "conditions": [
      {
        "diagnosis": "string",
        "icd10Code": "string (optional)",
        "onsetDate": "string",
        "status": "string",
        "severity": "string",
        "notes": "string (optional)"
      }
    ],
    "medications": [
      {
        "name": "string",
        "genericName": "string (optional)",
        "dosage": "string",
        "frequency": "string",
        "startDate": "string",
        "endDate": "string (optional)",
        "prescribingProvider": "string (optional)",
        "adherence": "string (optional)"
      }
    ],
    "allergies": [
      {
        "allergen": "string",
        "reaction": "string",
        "severity": "string",
        "onsetDate": "string (optional)"
      }
    ],
    "procedures": [
      {
        "name": "string",
        "cptCode": "string (optional)",
        "date": "string",
        "provider": "string",
        "outcome": "string (optional)"
      }
    ],
    "labResults": [
      {
        "testName": "string",
        "value": "string",
        "unit": "string",
        "referenceRange": "string",
        "date": "string",
        "abnormalFlag": "boolean"
      }
    ]
  },
  
  "interactionHistory": {
    "currentSession": {
      "sessionId": "string",
      "type": "string",
      "chiefComplaint": "string",
      "symptoms": ["string"],
      "assessment": "object",
      "recommendations": ["string"]
    },
    "previousInteractions": [
      {
        "date": "string",
        "type": "string",
        "summary": "string",
        "provider": "string",
        "outcome": "string"
      }
    ]
  },
  
  "preferences": {
    "communicationPreferences": {
      "language": "string",
      "contactMethods": ["string"],
      "timezone": "string"
    },
    "healthGoals": ["string"],
    "carePreferences": {
      "preferredProviders": ["string"],
      "preferredHospitals": ["string"],
      "treatmentPreferences": ["string"]
    }
  }
}
```

### Node 2: Data Classification & Validation
**Type**: Code (JavaScript)
**Purpose**: Classify and validate incoming patient data

**Implementation:**
```javascript
const contextRequest = $json.body;

const validation = {
  isValid: true,
  errors: [],
  warnings: [],
  dataQuality: {
    completeness: 0,
    accuracy: 0,
    timeliness: 0
  },
  classification: {
    phi: [],
    medicalData: [],
    preferences: [],
    interactions: []
  }
};

// Required fields validation
const requiredFields = ['requestId', 'patientId', 'requestType'];
for (const field of requiredFields) {
  if (!contextRequest[field]) {
    validation.isValid = false;
    validation.errors.push(`Missing required field: ${field}`);
  }
}

// Validate patient demographics
if (contextRequest.patientData?.demographics) {
  const demographics = contextRequest.patientData.demographics;
  
  if (demographics.age && (demographics.age < 0 || demographics.age > 150)) {
    validation.errors.push('Invalid age value');
  } else if (demographics.age) {
    validation.dataQuality.completeness += 20;
    validation.classification.phi.push('age');
  }
  
  if (demographics.dateOfBirth) {
    const birthDate = new Date(demographics.dateOfBirth);
    if (isNaN(birthDate.getTime())) {
      validation.errors.push('Invalid date of birth format');
    } else {
      validation.dataQuality.accuracy += 15;
      validation.classification.phi.push('dateOfBirth');
    }
  }
  
  if (demographics.location) {
    validation.dataQuality.completeness += 10;
    validation.classification.phi.push('location');
  }
}

// Validate medical conditions
if (contextRequest.medicalData?.conditions) {
  contextRequest.medicalData.conditions.forEach((condition, index) => {
    if (!condition.diagnosis) {
      validation.errors.push(`Condition ${index + 1} missing diagnosis`);
    } else {
      validation.classification.medicalData.push(`condition_${condition.diagnosis}`);
      validation.dataQuality.completeness += 5;
    }
    
    if (condition.onsetDate) {
      const onsetDate = new Date(condition.onsetDate);
      if (isNaN(onsetDate.getTime())) {
        validation.warnings.push(`Invalid onset date for condition: ${condition.diagnosis}`);
      } else {
        validation.dataQuality.timeliness += 5;
      }
    }
  });
}

// Validate medications
if (contextRequest.medicalData?.medications) {
  contextRequest.medicalData.medications.forEach((medication, index) => {
    if (!medication.name) {
      validation.errors.push(`Medication ${index + 1} missing name`);
    } else {
      validation.classification.medicalData.push(`medication_${medication.name}`);
      validation.dataQuality.completeness += 5;
    }
    
    if (medication.dosage && medication.frequency) {
      validation.dataQuality.accuracy += 5;
    }
  });
}

// Validate allergies
if (contextRequest.medicalData?.allergies) {
  contextRequest.medicalData.allergies.forEach((allergy, index) => {
    if (!allergy.allergen) {
      validation.errors.push(`Allergy ${index + 1} missing allergen`);
    } else {
      validation.classification.medicalData.push(`allergy_${allergy.allergen}`);
      validation.dataQuality.completeness += 3;
    }
  });
}

// Validate interaction history
if (contextRequest.interactionHistory?.currentSession) {
  const session = contextRequest.interactionHistory.currentSession;
  if (session.chiefComplaint) {
    validation.classification.interactions.push('current_session');
    validation.dataQuality.timeliness += 10;
  }
}

// Calculate overall data quality scores
const maxCompleteness = 100;
const maxAccuracy = 100;
const maxTimeliness = 100;

validation.dataQuality.completeness = Math.min(validation.dataQuality.completeness, maxCompleteness);
validation.dataQuality.accuracy = Math.min(validation.dataQuality.accuracy, maxAccuracy);
validation.dataQuality.timeliness = Math.min(validation.dataQuality.timeliness, maxTimeliness);

// Data sanitization
const sanitizedData = {
  ...contextRequest,
  processedAt: new Date().toISOString(),
  dataFingerprint: `${contextRequest.patientId}_${Date.now()}`
};

// Remove any potential XSS or injection attempts
const sanitizeText = (text) => {
  if (typeof text === 'string') {
    return text.trim().replace(/[<>\"']/g, '').substring(0, 1000);
  }
  return text;
};

// Sanitize text fields recursively
const sanitizeObject = (obj) => {
  if (Array.isArray(obj)) {
    return obj.map(sanitizeObject);
  } else if (obj && typeof obj === 'object') {
    const sanitized = {};
    for (const [key, value] of Object.entries(obj)) {
      sanitized[key] = sanitizeObject(value);
    }
    return sanitized;
  } else if (typeof obj === 'string') {
    return sanitizeText(obj);
  }
  return obj;
};

sanitizedData.medicalData = sanitizeObject(sanitizedData.medicalData);
sanitizedData.interactionHistory = sanitizeObject(sanitizedData.interactionHistory);

return {
  validation,
  contextRequest: validation.isValid ? sanitizedData : null,
  timestamp: new Date().toISOString()
};
```

### Node 3: Patient Identity Resolution
**Type**: Memory MCP
**Purpose**: Resolve patient identity and check for existing records

**Configuration:**
```json
{
  "server": "memory",
  "tool": "search_nodes",
  "parameters": {
    "query": "{{ $json.contextRequest.patientId }} patient_profile demographics identifiers"
  }
}
```

### Node 4: Historical Data Retrieval
**Type**: Memory MCP
**Purpose**: Retrieve comprehensive patient history

**Configuration:**
```json
{
  "server": "memory",
  "tool": "open_nodes",
  "parameters": {
    "names": [
      "{{ $json.contextRequest.patientId }}_profile",
      "{{ $json.contextRequest.patientId }}_conditions",
      "{{ $json.contextRequest.patientId }}_medications",
      "{{ $json.contextRequest.patientId }}_allergies",
      "{{ $json.contextRequest.patientId }}_interactions"
    ]
  }
}
```

### Node 5: Data Processing & Enrichment
**Type**: Code (JavaScript)
**Purpose**: Process and enrich patient data with historical context

**Implementation:**
```javascript
const contextRequest = $json.contextRequest;
const existingProfile = $json.patientIdentity || {};
const historicalData = $json.historicalData || {};

const enrichedData = {
  patientProfile: {
    id: contextRequest.patientId,
    lastUpdated: new Date().toISOString(),
    dataQuality: $json.validation.dataQuality,
    completeness: 0
  },
  medicalHistory: {
    conditions: [],
    medications: [],
    allergies: [],
    procedures: [],
    labResults: []
  },
  interactions: [],
  insights: {
    trends: [],
    risks: [],
    recommendations: []
  },
  timeline: []
};

// Merge existing profile with new data
if (existingProfile.results && existingProfile.results.length > 0) {
  const existingData = existingProfile.results[0];
  enrichedData.patientProfile.existing = true;
  enrichedData.patientProfile.firstCreated = existingData.created || new Date().toISOString();
} else {
  enrichedData.patientProfile.existing = false;
  enrichedData.patientProfile.firstCreated = new Date().toISOString();
}

// Process medical conditions
if (contextRequest.medicalData?.conditions) {
  contextRequest.medicalData.conditions.forEach(condition => {
    const enrichedCondition = {
      ...condition,
      id: `condition_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      enteredDate: new Date().toISOString(),
      status: condition.status || 'active'
    };
    
    // Check for condition interactions or complications
    if (condition.diagnosis.toLowerCase().includes('diabetes')) {
      enrichedCondition.riskFactors = ['cardiovascular disease', 'kidney disease', 'neuropathy'];
      enrichedData.insights.risks.push('Diabetes complications monitoring recommended');
    }
    
    if (condition.diagnosis.toLowerCase().includes('hypertension')) {
      enrichedCondition.riskFactors = ['stroke', 'heart attack', 'kidney disease'];
      enrichedData.insights.risks.push('Blood pressure monitoring critical');
    }
    
    enrichedData.medicalHistory.conditions.push(enrichedCondition);
    enrichedData.timeline.push({
      date: condition.onsetDate || new Date().toISOString(),
      type: 'condition_onset',
      description: `Diagnosed with ${condition.diagnosis}`,
      severity: condition.severity
    });
  });
}

// Process medications
if (contextRequest.medicalData?.medications) {
  contextRequest.medicalData.medications.forEach(medication => {
    const enrichedMedication = {
      ...medication,
      id: `medication_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      enteredDate: new Date().toISOString(),
      status: medication.endDate ? 'discontinued' : 'active'
    };
    
    // Check for drug interactions with existing medications
    const existingMedications = enrichedData.medicalHistory.medications;
    const interactionChecks = [];
    
    // Common interaction patterns
    if (medication.name.toLowerCase().includes('warfarin')) {
      interactionChecks.push('Monitor INR levels closely');
      enrichedData.insights.risks.push('Warfarin requires careful monitoring and drug interaction screening');
    }
    
    if (medication.name.toLowerCase().includes('insulin')) {
      interactionChecks.push('Monitor blood glucose levels');
      enrichedData.insights.recommendations.push('Regular glucose monitoring and carbohydrate counting essential');
    }
    
    enrichedMedication.interactionChecks = interactionChecks;
    enrichedData.medicalHistory.medications.push(enrichedMedication);
    
    enrichedData.timeline.push({
      date: medication.startDate || new Date().toISOString(),
      type: 'medication_start',
      description: `Started ${medication.name} ${medication.dosage}`,
      prescriber: medication.prescribingProvider
    });
  });
}

// Process allergies
if (contextRequest.medicalData?.allergies) {
  contextRequest.medicalData.allergies.forEach(allergy => {
    const enrichedAllergy = {
      ...allergy,
      id: `allergy_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      enteredDate: new Date().toISOString(),
      crossReactivity: []
    };
    
    // Check for cross-reactivity patterns
    if (allergy.allergen.toLowerCase().includes('penicillin')) {
      enrichedAllergy.crossReactivity = ['amoxicillin', 'ampicillin', 'cephalosporins'];
      enrichedData.insights.risks.push('Penicillin allergy - avoid beta-lactam antibiotics');
    }
    
    if (allergy.allergen.toLowerCase().includes('sulfa')) {
      enrichedAllergy.crossReactivity = ['sulfamethoxazole', 'sulfadiazine'];
      enrichedData.insights.risks.push('Sulfa allergy - screen sulfonamide medications');
    }
    
    enrichedData.medicalHistory.allergies.push(enrichedAllergy);
    
    enrichedData.timeline.push({
      date: allergy.onsetDate || new Date().toISOString(),
      type: 'allergy_identified',
      description: `Allergy to ${allergy.allergen} identified`,
      severity: allergy.severity
    });
  });
}

// Process current interaction
if (contextRequest.interactionHistory?.currentSession) {
  const session = contextRequest.interactionHistory.currentSession;
  const enrichedInteraction = {
    ...session,
    id: `interaction_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
    timestamp: new Date().toISOString(),
    context: {
      relatedConditions: [],
      medicationRelevance: [],
      allergyConsiderations: []
    }
  };
  
  // Analyze interaction context
  if (session.symptoms) {
    session.symptoms.forEach(symptom => {
      // Check symptom relevance to existing conditions
      enrichedData.medicalHistory.conditions.forEach(condition => {
        if (symptom.toLowerCase().includes(condition.diagnosis.split(' ')[0].toLowerCase())) {
          enrichedInteraction.context.relatedConditions.push(condition.diagnosis);
        }
      });
    });
  }
  
  enrichedData.interactions.push(enrichedInteraction);
}

// Generate insights and trends
enrichedData.insights.trends = [
  `Patient has ${enrichedData.medicalHistory.conditions.length} active conditions`,
  `Currently on ${enrichedData.medicalHistory.medications.filter(m => m.status === 'active').length} medications`,
  `${enrichedData.medicalHistory.allergies.length} known allergies documented`
];

// Calculate completeness score
let completenessScore = 0;
if (contextRequest.patientData?.demographics) completenessScore += 20;
if (enrichedData.medicalHistory.conditions.length > 0) completenessScore += 25;
if (enrichedData.medicalHistory.medications.length > 0) completenessScore += 25;
if (enrichedData.medicalHistory.allergies.length > 0) completenessScore += 15;
if (enrichedData.interactions.length > 0) completenessScore += 15;

enrichedData.patientProfile.completeness = completenessScore;

// Sort timeline chronologically
enrichedData.timeline.sort((a, b) => new Date(a.date) - new Date(b.date));

return {
  ...contextRequest,
  enrichedPatientData: enrichedData,
  processingComplete: true
};
```

### Node 6: Context Synthesis & Update
**Type**: Sequential Thinking MCP
**Purpose**: Apply clinical reasoning to synthesize patient context

**Configuration:**
```json
{
  "server": "sequential-thinking",
  "tool": "sequentialthinking",
  "parameters": {
    "thought": "Analyzing comprehensive patient context for {{ $json.contextRequest.patientId }}. Patient profile completeness: {{ $json.enrichedPatientData.patientProfile.completeness }}%. Active conditions: {{ $json.enrichedPatientData.medicalHistory.conditions.length }}. Active medications: {{ $json.enrichedPatientData.medicalHistory.medications.filter(m => m.status === 'active').length }}. Known allergies: {{ $json.enrichedPatientData.medicalHistory.allergies.length }}. Need to identify care patterns, potential risks, drug interactions, and personalized care opportunities.",
    "thoughtNumber": 1,
    "totalThoughts": 4,
    "nextThoughtNeeded": true
  }
}
```

### Node 7: Memory MCP - Data Storage
**Type**: Memory MCP
**Purpose**: Store enriched patient data and relationships

**Configuration:**
```json
{
  "server": "memory",
  "tool": "create_entities",
  "parameters": {
    "entities": [
      {
        "name": "{{ $json.contextRequest.patientId }}_profile_updated",
        "entityType": "patient_profile",
        "observations": [
          "Patient ID: {{ $json.contextRequest.patientId }}",
          "Age: {{ $json.contextRequest.patientData?.demographics?.age || 'unknown' }}",
          "Gender: {{ $json.contextRequest.patientData?.demographics?.gender || 'unknown' }}",
          "Profile completeness: {{ $json.enrichedPatientData.patientProfile.completeness }}%",
          "Active conditions: {{ $json.enrichedPatientData.medicalHistory.conditions.length }}",
          "Active medications: {{ $json.enrichedPatientData.medicalHistory.medications.filter(m => m.status === 'active').length }}",
          "Known allergies: {{ $json.enrichedPatientData.medicalHistory.allergies.length }}",
          "Last updated: {{ $json.enrichedPatientData.patientProfile.lastUpdated }}",
          "Timeline events: {{ $json.enrichedPatientData.timeline.length }}"
        ]
      }
    ]
  }
}
```

### Node 8: Relationship Mapping
**Type**: Memory MCP
**Purpose**: Create relationships between patient data entities

**Configuration:**
```json
{
  "server": "memory",
  "tool": "create_relations",
  "parameters": {
    "relations": [
      {
        "from": "{{ $json.contextRequest.patientId }}_profile_updated",
        "to": "{{ $json.contextRequest.patientId }}_conditions",
        "relationType": "has_medical_conditions"
      },
      {
        "from": "{{ $json.contextRequest.patientId }}_profile_updated",
        "to": "{{ $json.contextRequest.patientId }}_medications",
        "relationType": "takes_medications"
      },
      {
        "from": "{{ $json.contextRequest.patientId }}_profile_updated",
        "to": "{{ $json.contextRequest.patientId }}_allergies",
        "relationType": "has_allergies"
      }
    ]
  }
}
```

### Node 9: Intelligent Insights Generation
**Type**: Code (JavaScript)
**Purpose**: Generate personalized insights and recommendations

**Implementation:**
```javascript
const enrichedData = $json.enrichedPatientData;
const clinicalReasoning = $json.sequentialThinking || {};

const intelligentInsights = {
  personalizedRecommendations: [],
  riskAssessments: [],
  careGaps: [],
  medicationOptimization: [],
  preventiveCare: [],
  followUpPriorities: []
};

// Risk assessment based on conditions
const conditions = enrichedData.medicalHistory.conditions;
const medications = enrichedData.medicalHistory.medications;
const allergies = enrichedData.medicalHistory.allergies;

// Diabetes management insights
const diabetesConditions = conditions.filter(c => 
  c.diagnosis.toLowerCase().includes('diabetes')
);
if (diabetesConditions.length > 0) {
  intelligentInsights.personalizedRecommendations.push({
    category: 'Diabetes Management',
    priority: 'high',
    recommendations: [
      'Regular HbA1c monitoring every 3-6 months',
      'Annual eye examination for diabetic retinopathy',
      'Foot care assessment and daily inspection',
      'Blood pressure monitoring and control'
    ]
  });
  
  intelligentInsights.riskAssessments.push({
    risk: 'Diabetic Complications',
    level: 'moderate_to_high',
    factors: ['Duration of diabetes', 'Glycemic control', 'Comorbidities'],
    monitoring: 'Regular screening for cardiovascular, renal, and ocular complications'
  });
}

// Hypertension management
const hypertensionConditions = conditions.filter(c => 
  c.diagnosis.toLowerCase().includes('hypertension')
);
if (hypertensionConditions.length > 0) {
  intelligentInsights.personalizedRecommendations.push({
    category: 'Hypertension Management',
    priority: 'high',
    recommendations: [
      'Home blood pressure monitoring',
      'DASH diet implementation',
      'Regular cardiovascular exercise',
      'Sodium intake reduction'
    ]
  });
}

// Medication optimization
if (medications.length > 5) {
  intelligentInsights.medicationOptimization.push({
    issue: 'Polypharmacy Risk',
    priority: 'medium',
    recommendations: [
      'Comprehensive medication review with pharmacist',
      'Drug interaction screening',
      'Consider deprescribing inappropriate medications',
      'Medication adherence assessment'
    ]
  });
}

// Allergy considerations
if (allergies.length > 0) {
  intelligentInsights.riskAssessments.push({
    risk: 'Adverse Drug Reactions',
    level: 'medium',
    factors: allergies.map(a => a.allergen),
    monitoring: 'Always verify allergy status before prescribing new medications'
  });
}

// Age-based preventive care
const age = $json.contextRequest.patientData?.demographics?.age;
if (age) {
  if (age >= 50) {
    intelligentInsights.preventiveCare.push({
      category: 'Cancer Screening',
      priority: 'medium',
      recommendations: [
        'Colorectal cancer screening (colonoscopy or FIT)',
        'Mammography (if applicable)',
        'Lung cancer screening (if smoking history)'
      ]
    });
  }
  
  if (age >= 65) {
    intelligentInsights.preventiveCare.push({
      category: 'Geriatric Care',
      priority: 'medium',
      recommendations: [
        'Annual influenza vaccination',
        'Pneumococcal vaccination',
        'Fall risk assessment',
        'Cognitive screening',
        'Osteoporosis screening'
      ]
    });
  }
}

// Care gaps identification
let careGaps = [];
if (!medications.some(m => m.name.toLowerCase().includes('statin')) && 
    conditions.some(c => c.diagnosis.toLowerCase().includes('diabetes'))) {
  careGaps.push('Consider statin therapy for cardiovascular protection in diabetes');
}

if (!medications.some(m => m.name.toLowerCase().includes('ace') || m.name.toLowerCase().includes('arb')) && 
    conditions.some(c => c.diagnosis.toLowerCase().includes('hypertension'))) {
  careGaps.push('Consider ACE inhibitor or ARB for hypertension management');
}

intelligentInsights.careGaps = careGaps;

// Follow-up priorities
intelligentInsights.followUpPriorities = [
  {
    priority: 'high',
    item: 'Medication adherence assessment',
    timeframe: '1-2 weeks'
  },
  {
    priority: 'medium',
    item: 'Vital signs monitoring',
    timeframe: '1 month'
  },
  {
    priority: 'low',
    item: 'Preventive care review',
    timeframe: '3-6 months'
  }
];

return {
  ...enrichedData,
  intelligentInsights,
  contextComplete: true,
  generatedAt: new Date().toISOString()
};
```

### Node 10: Context Response & Recommendations
**Type**: Webhook Response
**Purpose**: Return comprehensive patient context and recommendations

**Configuration:**
```json
{
  "respondWith": "json",
  "responseBody": {
    "success": true,
    "requestId": "{{ $json.contextRequest.requestId }}",
    "patientId": "{{ $json.contextRequest.patientId }}",
    "contextProcessed": true,
    
    "patientProfile": {
      "completeness": "{{ $json.enrichedPatientData.patientProfile.completeness }}%",
      "lastUpdated": "{{ $json.enrichedPatientData.patientProfile.lastUpdated }}",
      "dataQuality": "{{ $json.validation.dataQuality }}"
    },
    
    "medicalSummary": {
      "activeConditions": "{{ $json.enrichedPatientData.medicalHistory.conditions.length }}",
      "activeMedications": "{{ $json.enrichedPatientData.medicalHistory.medications.filter(m => m.status === 'active').length }}",
      "knownAllergies": "{{ $json.enrichedPatientData.medicalHistory.allergies.length }}",
      "timelineEvents": "{{ $json.enrichedPatientData.timeline.length }}"
    },
    
    "personalizedInsights": {
      "recommendations": "{{ $json.intelligentInsights.personalizedRecommendations }}",
      "riskAssessments": "{{ $json.intelligentInsights.riskAssessments }}",
      "careGaps": "{{ $json.intelligentInsights.careGaps }}",
      "preventiveCare": "{{ $json.intelligentInsights.preventiveCare }}"
    },
    
    "actionItems": {
      "followUpPriorities": "{{ $json.intelligentInsights.followUpPriorities }}",
      "medicationOptimization": "{{ $json.intelligentInsights.medicationOptimization }}"
    },
    
    "contextMetadata": {
      "processingTime": "calculated_in_seconds",
      "dataClassification": "{{ $json.validation.classification }}",
      "generatedAt": "{{ $json.generatedAt }}"
    }
  },
  "responseCode": 200
}
```

## Data Privacy & Security

### PHI Protection
- All patient data encrypted at rest and in transit
- Role-based access controls for data retrieval
- Audit logging for all data access and modifications
- Automatic data retention and purging policies

### Memory MCP Security
- Encrypted storage of all patient entities
- Secure relationship mapping without exposing sensitive data
- Regular backup and disaster recovery procedures
- Compliance with HIPAA privacy and security rules

## Performance Optimization

### Response Time Targets
- Context retrieval: <3 seconds
- Data processing: <5 seconds
- Insight generation: <2 seconds
- Complete workflow: <10 seconds

### Scalability Features
- Parallel processing of patient data components
- Caching for frequently accessed patient profiles
- Optimized Memory MCP queries
- Batch processing for bulk updates

This patient context management workflow ensures comprehensive, secure, and intelligent handling of patient data while providing personalized healthcare insights and maintaining the highest standards of data privacy and security.