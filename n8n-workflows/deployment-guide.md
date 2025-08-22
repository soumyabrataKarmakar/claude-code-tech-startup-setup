# Master Chatbot Orchestration Workflow - Deployment Guide

## 🏥 Overview

This deployment guide provides comprehensive instructions for setting up the Master Chatbot Orchestration Workflow that integrates Claude 3.5 Sonnet with all 7 healthcare workflows in a HIPAA-compliant environment.

## 📋 Prerequisites

### Required Software
- **N8N**: Version 1.0+ (self-hosted for HIPAA compliance)
- **Node.js**: Version 18+ 
- **Docker**: For containerized deployment
- **SSL Certificate**: Required for HTTPS enforcement
- **Database**: PostgreSQL 14+ for audit logging

### Required API Keys & Credentials
```bash
# Anthropic API Configuration
ANTHROPIC_API_KEY=your_anthropic_api_key_here

# N8N Configuration
N8N_API_URL=https://your-n8n-instance.com
N8N_API_KEY=your_n8n_api_key
N8N_WEBHOOK_BASE_URL=https://your-n8n-instance.com

# Memory MCP Configuration
MEMORY_MCP_ENDPOINT=https://your-n8n-instance.com/webhook/memory-mcp

# HIPAA Compliance Settings
HIPAA_AUDIT_ENABLED=true
AUDIT_LOG_RETENTION_DAYS=2555  # 7 years required
AUDIT_DATABASE_URL=postgresql://user:pass@host:port/audit_db

# Security Configuration
JWT_SECRET=your_jwt_secret_key
ENCRYPTION_KEY=your_32_character_encryption_key
API_RATE_LIMIT=1000  # requests per minute per IP
```

## 🔧 Step-by-Step Deployment

### Step 1: Environment Setup

1. **Clone the Repository**
```bash
git clone <your-repo-url>
cd claude-code-tech-startup-setup/n8n-workflows
```

2. **Set Environment Variables**
```bash
# Create .env file
cp .env.example .env

# Edit .env with your actual credentials
nano .env
```

3. **Install Dependencies**
```bash
npm install
# or if using Docker
docker-compose up -d
```

### Step 2: Anthropic API Configuration

1. **Create Anthropic API Credentials in N8N**
   - Navigate to N8N → Credentials
   - Create new credential of type "HTTP Header Auth"
   - Name: "Anthropic API Key"
   - Header Name: `x-api-key`
   - Header Value: `your_anthropic_api_key`

2. **Verify API Access**
```bash
curl -H "x-api-key: your_api_key" \
     -H "anthropic-version: 2023-06-01" \
     -H "content-type: application/json" \
     https://api.anthropic.com/v1/messages
```

### Step 3: Import Healthcare Workflows

1. **Import Individual Workflows** (in this order)
   - `emergency-escalation-workflow.json`
   - `patient-intake-workflow.json`
   - `clinical-decision-support-workflow.json`
   - `patient-context-workflow.json`
   - `provider-integration-workflow.json`
   - `multi-modal-interface-workflow.json`
   - `dual-oversight-integration-workflow.json`

2. **Import Master Orchestration Workflow**
   - `master-chatbot-orchestration-workflow.json`

3. **Activate All Workflows**
```bash
curl -X PATCH "http://localhost:5678/api/v1/workflows/{workflow_id}/activate" \
     -H "X-N8N-API-KEY: your_api_key"
```

### Step 4: Memory MCP Integration Setup

1. **Configure Memory MCP Server**
```bash
# Install Memory MCP
npx @modelcontextprotocol/server-memory

# Create webhook endpoint for Memory MCP
# This should be configured in your MCP setup
```

2. **Test Memory Integration**
```bash
curl -X POST "http://localhost:5678/webhook/memory-mcp" \
     -H "Content-Type: application/json" \
     -d '{
       "action": "create_entities",
       "entities": [{
         "name": "test_conversation",
         "entityType": "conversation",
         "observations": ["Test message"]
       }]
     }'
```

### Step 5: HIPAA Compliance Configuration

1. **Enable HTTPS/TLS**
```yaml
# docker-compose.yml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n
    environment:
      - N8N_PROTOCOL=https
      - N8N_SSL_KEY=/certs/private.key
      - N8N_SSL_CERT=/certs/certificate.crt
    volumes:
      - ./certs:/certs:ro
```

2. **Configure Audit Database**
```sql
-- Create HIPAA audit table
CREATE TABLE hipaa_audit_logs (
  audit_id UUID PRIMARY KEY,
  timestamp TIMESTAMPTZ NOT NULL,
  session_id_hash VARCHAR(64) NOT NULL,
  patient_id_hash VARCHAR(64) NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  workflow_type VARCHAR(100) NOT NULL,
  intent_classified VARCHAR(50),
  urgency_level VARCHAR(20),
  compliance_status VARCHAR(50) NOT NULL,
  retention_until TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Create retention policy
CREATE INDEX idx_retention_until ON hipaa_audit_logs(retention_until);
```

3. **Set Up Data Retention**
```sql
-- Automated cleanup job (run daily)
DELETE FROM hipaa_audit_logs 
WHERE retention_until < NOW();
```

### Step 6: Authentication & Security

1. **Configure Webhook Authentication**
```javascript
// N8N Webhook Authentication
{
  "authentication": "headerAuth",
  "credentials": "api-key-credential",
  "options": {
    "allowedOrigins": "https://your-healthcare-app.com"
  }
}
```

2. **Set Rate Limiting**
```yaml
# nginx.conf or equivalent
location /webhook/ {
    limit_req zone=api burst=10 nodelay;
    proxy_pass http://n8n-backend;
}
```

## 🧪 Testing & Validation

### Run Test Scenarios

1. **Execute Test Suite**
```bash
# Use the test-scenarios.json file
node test-runner.js test-scenarios.json
```

2. **Manual Testing Examples**

**Emergency Test:**
```bash
curl -X POST "https://your-n8n.com/webhook/chatbot" \
     -H "Authorization: Bearer test-token" \
     -H "Content-Type: application/json" \
     -d '{
       "patient_id": "test-123",
       "session_id": "sess-456",
       "message": "I am having severe chest pain and trouble breathing"
     }'
```

**General Health Test:**
```bash
curl -X POST "https://your-n8n.com/webhook/chatbot" \
     -H "Authorization: Bearer test-token" \
     -H "Content-Type: application/json" \
     -d '{
       "patient_id": "test-456", 
       "session_id": "sess-789",
       "message": "What are some tips for heart health?"
     }'
```

### Validation Checklist

- [ ] All 7 healthcare workflows are active
- [ ] Master orchestration workflow routes correctly
- [ ] Claude 3.5 Sonnet API responds properly
- [ ] Memory MCP creates conversation entities
- [ ] HIPAA audit logs are generated
- [ ] Authentication validates properly
- [ ] Error handling works for timeouts
- [ ] Response synthesis generates proper outputs

## 📊 Monitoring & Maintenance

### Health Checks

1. **N8N Health Endpoint**
```bash
curl "https://your-n8n.com/healthz"
```

2. **Claude API Health**
```bash
curl -H "x-api-key: your_key" \
     -H "anthropic-version: 2023-06-01" \
     https://api.anthropic.com/v1/messages
```

3. **Memory MCP Health**
```bash
curl -X POST "https://your-n8n.com/webhook/memory-mcp" \
     -d '{"action": "health_check"}'
```

### Performance Monitoring

1. **Key Metrics to Track**
   - Response times by workflow type
   - Error rates and retry attempts  
   - Claude API token usage
   - Memory MCP entity creation rates
   - HIPAA audit log volume

2. **Alerting Setup**
```yaml
# Example Prometheus alerts
- alert: ChatbotResponseTime
  expr: avg(chatbot_response_time) > 30
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Chatbot response time exceeding 30 seconds"

- alert: HIpaaAuditFailure  
  expr: rate(hipaa_audit_failures[5m]) > 0.01
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "HIPAA audit logging failures detected"
```

## 🔒 Security Best Practices

### Data Protection
- All PHI is encrypted in transit and at rest
- Patient IDs are hashed in audit logs
- Session data expires after 24 hours
- API keys rotate every 90 days

### Access Control
- Role-based access to N8N workflows
- IP whitelisting for webhook endpoints
- Multi-factor authentication for admin access
- Regular security audits and penetration testing

### Compliance Monitoring
- Daily audit log reviews
- Monthly HIPAA compliance assessments
- Quarterly security vulnerability scans
- Annual third-party security audits

## 🚨 Troubleshooting

### Common Issues

1. **Claude API Rate Limits**
   - Implement exponential backoff
   - Monitor token usage
   - Consider multiple API keys

2. **Workflow Timeouts**
   - Check network connectivity
   - Review timeout settings
   - Analyze workflow complexity

3. **Memory MCP Failures**
   - Verify webhook endpoints
   - Check MCP server health
   - Review entity creation logs

4. **HIPAA Audit Issues**
   - Verify database connectivity
   - Check retention policies
   - Review log formatting

### Support Contacts
- **Technical Issues**: tech-support@dextechlabs.com
- **HIPAA Compliance**: hipaa-officer@dextechlabs.com
- **Emergency Support**: emergency@dextechlabs.com (24/7)

## 📈 Scaling Considerations

### Horizontal Scaling
- Deploy multiple N8N instances behind load balancer
- Use Redis for session state management
- Implement database read replicas

### Performance Optimization
- Cache frequently used data
- Optimize Claude prompt lengths
- Use async processing where possible
- Monitor and tune database queries

This deployment guide ensures your Master Chatbot Orchestration Workflow is production-ready, HIPAA-compliant, and scalable for healthcare applications.