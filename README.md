# Personal Health Assistant

A comprehensive healthcare assistant platform combining mobile application with N8N-powered intelligent workflows, leveraging Model Context Protocol (MCP) tools for medical intelligence and patient care.

## Overview

This Personal Health Assistant provides:
- **Intelligent Patient Triage**: AI-powered symptom assessment and clinical decision support
- **Emergency Detection**: Real-time identification of critical health conditions
- **Evidence-Based Care**: Integration with medical literature and clinical guidelines
- **Provider Integration**: Seamless connection with healthcare systems via FHIR/HL7
- **Accessible Interface**: Multi-modal support including voice, text, and accessibility features

## Architecture

### Mobile Application
- **Frontend**: React Native/Flutter cross-platform development
- **Interface**: Chat-based interaction with voice and accessibility support
- **Integration**: Real-time communication with N8N workflows via webhooks
- **Security**: End-to-end encryption and HIPAA compliance

### N8N Backend Workflows
- **Patient Intake**: Symptom collection and risk stratification
- **Clinical Decision Support**: Evidence-based medical guidance using BioMCP/PubMed
- **Emergency Protocols**: <30 second response for critical conditions
- **Patient Context**: Persistent history and personalized care using Memory MCP
- **Provider Integration**: Healthcare system connectivity and appointment scheduling
- **Interface Orchestration**: Multi-modal conversation management

### MCP Tool Integration
- **BioMCP**: Medical literature, clinical trials, drug information
- **PubMed MCP**: Research papers and evidence-based medicine
- **Sequential Thinking**: Complex medical reasoning and decision trees
- **Memory MCP**: Patient history and context persistence
- **Brave Search**: Current health information and guidelines
- **GitHub MCP**: Documentation and protocol management

## Project Structure

```
├── docs/                      # Documentation
│   ├── framework/            # Patient experience and clinical protocols
│   ├── workflows/            # N8N workflow specifications
│   ├── mobile-app/           # Mobile application architecture
│   ├── security/             # Security and compliance documentation
│   └── deployment/           # Deployment and monitoring guides
├── n8n-workflows/            # Production N8N workflow files
├── config/                   # Configuration files and settings
├── assets/                   # Images, diagrams, and resources
└── .mcp.json                # MCP server configurations
```

## Key Features

### Clinical Intelligence
- **Symptom Assessment**: Structured clinical interviews with decision trees
- **Risk Stratification**: Evidence-based triage protocols
- **Emergency Detection**: Real-time critical condition identification
- **Treatment Guidance**: Personalized care recommendations

### Healthcare Integration
- **FHIR/HL7 Compliance**: Standard healthcare data exchange
- **EHR Connectivity**: Integration with major healthcare systems
- **Provider Communication**: Automated referrals and care coordination
- **Appointment Management**: Scheduling and availability systems

### Patient Experience
- **Conversational Interface**: Natural language interaction
- **Voice Support**: Hands-free accessibility
- **Multi-language**: Translation and localization support
- **Privacy First**: HIPAA-compliant data handling

### Safety & Compliance
- **Medical Disclaimers**: Clear AI limitations and professional consultation requirements
- **Emergency Protocols**: Immediate escalation for critical conditions
- **Audit Trails**: Comprehensive logging for compliance
- **Clinical Validation**: Evidence-based responses with source citations

## Getting Started

1. **Prerequisites**: N8N instance with MCP support, mobile development environment
2. **MCP Configuration**: Ensure all MCP servers are properly configured (see .mcp.json)
3. **N8N Setup**: Import and configure the 6 core workflows
4. **Mobile App**: Set up React Native/Flutter development environment
5. **Testing**: Use provided test scenarios for clinical validation

## Documentation

- [Patient Experience Framework](docs/framework/patient-experience-framework.md)
- [Clinical Decision Support](docs/framework/clinical-decision-support.md)
- [N8N Workflow Specifications](docs/workflows/README.md)
- [Mobile App Architecture](docs/mobile-app/architecture.md)
- [Security & Compliance](docs/security/hipaa-compliance.md)
- [Deployment Guide](docs/deployment/setup-guide.md)

## Safety Notice

This Personal Health Assistant is designed to support, not replace, professional medical advice. Users should always consult healthcare providers for medical decisions. Emergency situations require immediate professional medical attention.

## License

[To be determined based on healthcare compliance requirements]