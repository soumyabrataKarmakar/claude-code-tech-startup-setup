# Clinical Decision Support Workflow Specification

## Purpose & Scope

The Clinical Decision Support Workflow provides evidence-based medical guidance by integrating multiple MCP tools to analyze patient symptoms, research medical literature, and generate clinically appropriate recommendations. This workflow supports healthcare providers and patients with reliable, evidence-based medical information.

## Clinical Objectives

- **Evidence-Based Guidance**: Provide recommendations backed by peer-reviewed research
- **Differential Diagnosis Support**: Consider multiple diagnostic possibilities
- **Treatment Protocol Integration**: Reference established clinical guidelines
- **Medication Safety**: Include drug interaction checking and safety information
- **Continuous Learning**: Update recommendations based on latest medical evidence

## Architecture Diagram

```
[Patient Assessment Data]
           ↓
    [Webhook Trigger]
           ↓
    [Input Processing & Validation]
           ↓
    [Sequential Thinking - Clinical Reasoning] ←→ [Memory MCP - Patient History]
           ↓
    [BioMCP - Medical Literature Search]
           ↓
    [PubMed MCP - Research Evidence]
           ↓
    [Brave Search - Current Guidelines]
           ↓
    [Sequential Thinking - Evidence Synthesis]
           ↓
    [Clinical Recommendation Engine]
           ↓
    [Drug Interaction Analysis] (if medications involved)
           ↓
    [Safety Review & Validation]
           ↓
    [Structured Response Generation]
           ↓
    [Memory MCP - Decision Storage]
           ↓
    [Provider/Patient Response]
```

## N8N Workflow Configuration

### Node 1: Clinical Decision Support Webhook
**Type**: Webhook
**Purpose**: Receive clinical decision support requests

**Configuration:**
```json
{
  "httpMethod": "POST",
  "path": "/webhook/clinical-decision-support",
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
  "sessionId": "string (UUID)",
  "requestType": "string (symptom_analysis|medication_review|condition_information|treatment_options)",
  "clinicalData": {
    "symptoms": ["string"],
    "duration": "string",
    "severity": "number (1-10)",
    "associatedFindings": ["string"],
    "aggravatingFactors": ["string"],
    "relievingFactors": ["string"]
  },
  "medicalHistory": {
    "conditions": ["string"],
    "medications": ["string"],
    "allergies": ["string"],
    "surgeries": ["string"]
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
    "respiratoryRate": "number (optional)",
    "oxygenSaturation": "number (optional)"
  },
  "labResults": {
    "recent": ["object (optional)"],
    "pending": ["string (optional)"]
  },
  "imaging": {
    "studies": ["object (optional)"],
    "findings": ["string (optional)"]
  },
  "priorities": {
    "urgency": "string (low|medium|high|emergency)",
    "focusAreas": ["string"],
    "providerSpecialty": "string (optional)"
  }
}
```

### Node 2: Input Processing & Clinical Validation
**Type**: Code (JavaScript)
**Purpose**: Process and validate clinical input data

**Implementation:**
```javascript
const clinicalRequest = $json.body;

// Clinical data validation
const validation = {
  isValid: true,
  errors: [],
  warnings: [],
  sanitizedData: {},
  clinicalFlags: []
};

// Required fields validation
const requiredFields = ['requestId', 'patientId', 'requestType'];
for (const field of requiredFields) {
  if (!clinicalRequest[field]) {
    validation.isValid = false;
    validation.errors.push(`Missing required field: ${field}`);
  }
}

// Validate severity score
if (clinicalRequest.clinicalData?.severity) {
  const severity = parseInt(clinicalRequest.clinicalData.severity);
  if (severity < 1 || severity > 10) {
    validation.errors.push('Severity must be between 1 and 10');
  } else if (severity >= 8) {
    validation.clinicalFlags.push('High severity score requires urgent attention');
  }
}

// Validate age for age-specific protocols
if (clinicalRequest.demographics?.age) {
  const age = parseInt(clinicalRequest.demographics.age);
  if (age < 0 || age > 150) {
    validation.errors.push('Invalid age provided');
  } else {
    if (age < 18) validation.clinicalFlags.push('Pediatric patient - age-specific protocols apply');
    if (age > 65) validation.clinicalFlags.push('Geriatric patient - medication adjustments may be needed');
  }
}

// Validate vital signs for abnormal values
if (clinicalRequest.vitals) {
  const vitals = clinicalRequest.vitals;
  if (vitals.temperature > 101.3) validation.clinicalFlags.push('Fever present');
  if (vitals.heartRate > 100) validation.clinicalFlags.push('Tachycardia present');
  if (vitals.heartRate < 60) validation.clinicalFlags.push('Bradycardia present');
  if (vitals.oxygenSaturation < 95) validation.clinicalFlags.push('Low oxygen saturation');
}

// Check for drug allergies and interactions potential
if (clinicalRequest.medicalHistory?.allergies?.length > 0) {
  validation.clinicalFlags.push('Patient has known allergies - check compatibility');
}

if (clinicalRequest.medicalHistory?.medications?.length > 5) {
  validation.clinicalFlags.push('Polypharmacy - check for drug interactions');
}

// Sanitize text inputs
const textFields = ['symptoms', 'associatedFindings', 'conditions'];
textFields.forEach(field => {
  if (clinicalRequest.clinicalData?.[field]) {
    validation.sanitizedData[field] = clinicalRequest.clinicalData[field]
      .map(item => item.trim().substring(0, 500).replace(/[<>]/g, ''));
  }
});

return {
  validation,
  clinicalRequest: validation.isValid ? { ...clinicalRequest, ...validation.sanitizedData } : null,
  timestamp: new Date().toISOString(),
  processingFlags: validation.clinicalFlags
};
```

### Node 3: Sequential Thinking - Initial Clinical Reasoning
**Type**: MCP Client Tool
**Purpose**: Begin structured clinical reasoning process

**Configuration:**
```json
{
  "server": "sequential-thinking",
  "tool": "sequentialthinking",
  "parameters": {
    "thought": "Beginning clinical analysis for {{ $json.clinicalRequest.requestType }}. Patient presents with: {{ $json.clinicalRequest.clinicalData.symptoms.join(', ') }}. Age: {{ $json.clinicalRequest.demographics?.age || 'unknown' }}, Severity: {{ $json.clinicalRequest.clinicalData?.severity || 'not specified' }}. Medical history includes: {{ $json.clinicalRequest.medicalHistory?.conditions?.join(', ') || 'none reported' }}. Need to consider differential diagnosis, evidence-based guidelines, and patient-specific factors.",
    "thoughtNumber": 1,
    "totalThoughts": 8,
    "nextThoughtNeeded": true
  }
}
```

### Node 4: Memory MCP - Patient History Integration
**Type**: MCP Client Tool
**Purpose**: Retrieve comprehensive patient medical history

**Configuration:**
```json
{
  "server": "memory",
  "tool": "search_nodes",
  "parameters": {
    "query": "{{ $json.clinicalRequest.patientId }} medical history conditions medications"
  }
}
```

### Node 5: BioMCP - Medical Literature Search
**Type**: MCP Client Tool
**Purpose**: Search medical databases for condition-specific information

**Configuration:**
```json
{
  "server": "biomcp",
  "tool": "search",
  "parameters": {
    "query": "disease: {{ $json.clinicalRequest.clinicalData.symptoms[0] }} AND symptoms: {{ $json.clinicalRequest.clinicalData.symptoms.join(' OR ') }}",
    "domain": "article",
    "page_size": 15,
    "call_benefit": "Searching for evidence-based information on patient symptoms for clinical decision support"
  }
}
```

### Node 6: PubMed MCP - Research Evidence
**Type**: MCP Client Tool
**Purpose**: Access recent peer-reviewed research

**Configuration:**
```json
{
  "server": "pubmed-mcp-server",
  "tool": "search_pubmed_key_words",
  "parameters": {
    "key_words": "{{ $json.clinicalRequest.clinicalData.symptoms.join(' AND ') }} diagnosis treatment",
    "num_results": 10
  }
}
```

### Node 7: Brave Search - Current Clinical Guidelines
**Type**: MCP Client Tool
**Purpose**: Search for current clinical practice guidelines

**Configuration:**
```json
{
  "server": "brave-search",
  "tool": "brave_web_search",
  "parameters": {
    "query": "{{ $json.clinicalRequest.clinicalData.symptoms[0] }} clinical practice guidelines 2024 2025 evidence-based treatment",
    "count": 8
  }
}
```

### Node 8: Drug Information Analysis (Conditional)
**Type**: MCP Client Tool
**Purpose**: Analyze medications if present in request

**Configuration:**
```json
{
  "server": "biomcp",
  "tool": "search",
  "parameters": {
    "query": "drug: {{ $json.clinicalRequest.medicalHistory?.medications?.join(' OR drug: ') || 'none' }}",
    "domain": "drug",
    "page_size": 10,
    "call_benefit": "Analyzing patient medications for safety, interactions, and therapeutic considerations"
  }
}
```

### Node 9: Sequential Thinking - Evidence Synthesis
**Type**: MCP Client Tool
**Purpose**: Synthesize evidence from multiple sources

**Configuration:**
```json
{
  "server": "sequential-thinking",
  "tool": "sequentialthinking",
  "parameters": {
    "thought": "Synthesizing evidence from medical literature, research papers, and clinical guidelines. Key findings from BioMCP: {{ $json.bioMCPResults?.results?.length || 0 }} relevant articles found. PubMed research: {{ $json.pubMedResults?.length || 0 }} papers reviewed. Clinical guidelines: {{ $json.braveSearchResults?.results?.length || 0 }} current guidelines identified. Need to integrate this evidence with patient-specific factors to generate appropriate recommendations.",
    "thoughtNumber": 4,
    "totalThoughts": 8,
    "nextThoughtNeeded": true
  }
}
```

### Node 10: Clinical Recommendation Engine
**Type**: Code (JavaScript)
**Purpose**: Generate evidence-based clinical recommendations

**Implementation:**
```javascript
const clinicalData = $json.clinicalRequest;
const bioMCPResults = $json.bioMCPResults || {};
const pubMedResults = $json.pubMedResults || [];
const guidelinesResults = $json.braveSearchResults || {};
const drugInfo = $json.drugAnalysis || {};
const patientHistory = $json.patientHistory || {};

// Initialize recommendation structure
const recommendations = {
  differentialDiagnosis: [],
  recommendedActions: [],
  diagnosticTests: [],
  treatmentOptions: [],
  medicationConsiderations: [],
  followUpCare: [],
  redFlags: [],
  patientEducation: [],
  evidenceSources: [],
  confidenceLevel: 'moderate'
};

// Process symptoms for differential diagnosis
const primarySymptoms = clinicalData.clinicalData.symptoms;
let diagnosisConfidence = 0;

// Analyze BioMCP results for diagnostic information
if (bioMCPResults.results && bioMCPResults.results.length > 0) {
  bioMCPResults.results.slice(0, 5).forEach(result => {
    recommendations.differentialDiagnosis.push({
      condition: result.title,
      likelihood: 'possible',
      evidenceSource: result.url
    });
    recommendations.evidenceSources.push({
      type: 'medical_database',
      title: result.title,
      url: result.url
    });
  });
  diagnosisConfidence += 20;
}

// Process PubMed research evidence
if (pubMedResults.length > 0) {
  pubMedResults.slice(0, 3).forEach(study => {
    recommendations.evidenceSources.push({
      type: 'peer_reviewed_research',
      title: study.title,
      pmid: study.pmid,
      relevance: 'high'
    });
  });
  diagnosisConfidence += 30;
}

// Process clinical guidelines
if (guidelinesResults.results && guidelinesResults.results.length > 0) {
  guidelinesResults.results.slice(0, 3).forEach(guideline => {
    recommendations.treatmentOptions.push({
      approach: 'Evidence-based guideline recommendation',
      source: guideline.title,
      url: guideline.url
    });
    recommendations.evidenceSources.push({
      type: 'clinical_guideline',
      title: guideline.title,
      url: guideline.url
    });
  });
  diagnosisConfidence += 25;
}

// Generate recommended actions based on severity and findings
const severity = clinicalData.clinicalData.severity || 0;
if (severity >= 8) {
  recommendations.recommendedActions.push('Urgent medical evaluation recommended within 24 hours');
  recommendations.redFlags.push('High severity score requires prompt medical attention');
} else if (severity >= 6) {
  recommendations.recommendedActions.push('Medical consultation recommended within 2-3 days');
} else {
  recommendations.recommendedActions.push('Routine medical follow-up as appropriate');
}

// Age-specific considerations
const age = clinicalData.demographics?.age;
if (age && age > 65) {
  recommendations.medicationConsiderations.push('Geriatric dosing adjustments may be required');
  recommendations.diagnosticTests.push('Consider age-related organ function assessment');
} else if (age && age < 18) {
  recommendations.medicationConsiderations.push('Pediatric dosing and formulations required');
  recommendations.treatmentOptions.push('Age-appropriate treatment protocols');
}

// Medication analysis if drugs present
if (clinicalData.medicalHistory?.medications?.length > 0) {
  recommendations.medicationConsiderations.push('Review current medications for interactions');
  recommendations.medicationConsiderations.push('Assess medication adherence and effectiveness');
  
  if (drugInfo.results && drugInfo.results.length > 0) {
    drugInfo.results.forEach(drug => {
      recommendations.medicationConsiderations.push(`Consider ${drug.title} interactions and contraindications`);
    });
  }
}

// Allergy considerations
if (clinicalData.medicalHistory?.allergies?.length > 0) {
  recommendations.redFlags.push('Patient has known allergies - verify medication and treatment compatibility');
  recommendations.medicationConsiderations.push('Avoid medications with known allergen cross-reactivity');
}

// Generate follow-up recommendations
recommendations.followUpCare = [
  'Monitor symptom progression and response to treatment',
  'Schedule appropriate follow-up based on condition severity',
  'Return for evaluation if symptoms worsen or new symptoms develop'
];

// Patient education recommendations
recommendations.patientEducation = [
  'Understand warning signs that require immediate medical attention',
  'Learn about condition management and self-care measures',
  'Maintain medication compliance and understand potential side effects'
];

// Set confidence level based on available evidence
if (diagnosisConfidence >= 70) {
  recommendations.confidenceLevel = 'high';
} else if (diagnosisConfidence >= 40) {
  recommendations.confidenceLevel = 'moderate';
} else {
  recommendations.confidenceLevel = 'low';
  recommendations.redFlags.push('Limited evidence available - professional medical consultation strongly recommended');
}

return {
  ...clinicalData,
  clinicalRecommendations: recommendations,
  evidenceQuality: {
    totalSources: recommendations.evidenceSources.length,
    confidenceScore: diagnosisConfidence,
    recommendationStrength: recommendations.confidenceLevel
  },
  timestamp: new Date().toISOString()
};
```

### Node 11: Safety Review & Clinical Validation
**Type**: Code (JavaScript)
**Purpose**: Final safety check and clinical validation

**Implementation:**
```javascript
const clinicalData = $json;
const recommendations = clinicalData.clinicalRecommendations;

const safetyReview = {
  safetyFlags: [],
  validationChecks: [],
  approvalStatus: 'pending',
  requiredDisclaimer: true
};

// Safety flag checks
const severity = clinicalData.clinicalRequest.clinicalData.severity || 0;
const age = clinicalData.clinicalRequest.demographics?.age;

// High severity safety checks
if (severity >= 8) {
  safetyReview.safetyFlags.push('High severity symptoms require urgent medical evaluation');
  safetyReview.validationChecks.push('Emergency escalation protocol reviewed');
}

// Age-specific safety considerations
if (age < 18) {
  safetyReview.safetyFlags.push('Pediatric patient - specialized protocols apply');
  safetyReview.validationChecks.push('Age-appropriate recommendations verified');
} else if (age > 75) {
  safetyReview.safetyFlags.push('Elderly patient - increased medication sensitivity');
  safetyReview.validationChecks.push('Geriatric considerations included');
}

// Medication safety checks
if (clinicalData.clinicalRequest.medicalHistory?.medications?.length > 5) {
  safetyReview.safetyFlags.push('Polypharmacy - drug interaction risk');
  safetyReview.validationChecks.push('Medication interaction analysis completed');
}

// Allergy safety checks
if (clinicalData.clinicalRequest.medicalHistory?.allergies?.length > 0) {
  safetyReview.safetyFlags.push('Known allergies - treatment compatibility required');
  safetyReview.validationChecks.push('Allergy considerations documented');
}

// Evidence quality validation
if (recommendations.evidenceSources.length < 3) {
  safetyReview.safetyFlags.push('Limited evidence - professional consultation strongly recommended');
}

// Final approval based on safety review
if (safetyReview.safetyFlags.length === 0) {
  safetyReview.approvalStatus = 'approved';
} else if (severity < 8 && safetyReview.safetyFlags.length <= 2) {
  safetyReview.approvalStatus = 'approved_with_caution';
} else {
  safetyReview.approvalStatus = 'requires_professional_review';
}

return {
  ...clinicalData,
  safetyReview,
  finalValidation: {
    timestamp: new Date().toISOString(),
    reviewComplete: true,
    safetyStatus: safetyReview.approvalStatus
  }
};
```

### Node 12: Structured Response Generation
**Type**: Code (JavaScript)
**Purpose**: Generate structured clinical decision support response

**Implementation:**
```javascript
const clinicalAnalysis = $json;
const recommendations = clinicalAnalysis.clinicalRecommendations;
const safetyReview = clinicalAnalysis.safetyReview;

const response = {
  success: true,
  requestId: clinicalAnalysis.clinicalRequest.requestId,
  patientId: clinicalAnalysis.clinicalRequest.patientId,
  sessionId: clinicalAnalysis.clinicalRequest.sessionId,
  analysisComplete: true,
  
  clinicalSummary: {
    primarySymptoms: clinicalAnalysis.clinicalRequest.clinicalData.symptoms,
    severity: clinicalAnalysis.clinicalRequest.clinicalData.severity,
    urgencyLevel: safetyReview.approvalStatus,
    confidenceLevel: recommendations.confidenceLevel
  },
  
  differentialDiagnosis: recommendations.differentialDiagnosis,
  
  recommendedActions: {
    immediate: recommendations.recommendedActions.filter(action => 
      action.includes('urgent') || action.includes('immediate')),
    shortTerm: recommendations.recommendedActions.filter(action => 
      action.includes('within') || action.includes('24 hours')),
    longTerm: recommendations.followUpCare
  },
  
  diagnosticRecommendations: recommendations.diagnosticTests,
  
  treatmentOptions: recommendations.treatmentOptions,
  
  medicationGuidance: {
    considerations: recommendations.medicationConsiderations,
    safetyAlerts: safetyReview.safetyFlags.filter(flag => 
      flag.includes('medication') || flag.includes('drug'))
  },
  
  patientEducation: recommendations.patientEducation,
  
  redFlags: recommendations.redFlags,
  
  evidenceSupport: {
    totalSources: recommendations.evidenceSources.length,
    qualityRating: recommendations.confidenceLevel,
    keyReferences: recommendations.evidenceSources.slice(0, 5)
  },
  
  safetyInformation: {
    flags: safetyReview.safetyFlags,
    approvalStatus: safetyReview.approvalStatus,
    requiresProfessionalReview: safetyReview.approvalStatus === 'requires_professional_review'
  },
  
  disclaimers: [
    "This clinical decision support is for informational purposes only",
    "Does not replace professional medical judgment or patient evaluation",
    "Healthcare providers should use clinical judgment in applying recommendations",
    "Patients should consult healthcare providers for medical decisions",
    "Emergency conditions require immediate professional medical care"
  ],
  
  metadata: {
    analysisDate: new Date().toISOString(),
    evidenceLastUpdated: new Date().toISOString(),
    systemVersion: "clinical-decision-support-v1.0.0",
    processingTime: "calculated_in_seconds"
  }
};

return response;
```

### Node 13: Memory MCP - Decision Storage
**Type**: MCP Client Tool
**Purpose**: Store clinical decision support results

**Configuration:**
```json
{
  "server": "memory",
  "tool": "create_entities",
  "parameters": {
    "entities": [
      {
        "name": "{{ $json.patientId }}_clinical_decision_{{ $json.requestId }}",
        "entityType": "clinical_decision_support",
        "observations": [
          "Request type: {{ $json.clinicalRequest.requestType }}",
          "Primary symptoms: {{ $json.clinicalRequest.clinicalData.symptoms.join(', ') }}",
          "Confidence level: {{ $json.clinicalRecommendations.confidenceLevel }}",
          "Safety status: {{ $json.safetyReview.approvalStatus }}",
          "Evidence sources: {{ $json.clinicalRecommendations.evidenceSources.length }}",
          "Decision date: {{ $json.metadata.analysisDate }}"
        ]
      }
    ]
  }
}
```

## Error Handling & Recovery

### Error Scenarios
1. **MCP Server Unavailable**: Fallback to cached guidelines and basic recommendations
2. **Insufficient Clinical Data**: Request additional information before proceeding
3. **Evidence Quality Too Low**: Escalate to professional review requirement
4. **Safety Flag Triggered**: Automatic escalation to emergency protocols

### Retry Configuration
```javascript
const retryConfig = {
  mcpTools: {
    maxAttempts: 3,
    backoffStrategy: 'exponential',
    initialDelay: 2000,
    maxDelay: 10000
  },
  evidenceSearch: {
    fallbackToLocal: true,
    cacheTimeout: 3600000 // 1 hour
  }
};
```

## Performance Metrics

### Target Response Times
- Complete analysis: <5 minutes
- Evidence synthesis: <2 minutes
- Safety validation: <30 seconds
- Response generation: <1 minute

### Quality Metrics
- Evidence source diversity: >3 different types
- Clinical accuracy: >95% agreement with specialists
- Safety flag detection: >99% sensitivity
- Patient satisfaction: >90% helpful rating

This clinical decision support workflow ensures evidence-based, safe, and comprehensive medical guidance while maintaining appropriate clinical boundaries and safety protocols.