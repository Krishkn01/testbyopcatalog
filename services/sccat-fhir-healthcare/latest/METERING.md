# Metering Integration Guide

## Overview

The FHIR Healthcare Service includes comprehensive metering integration with IBM Sovereign Core's metering platform. This enables usage tracking, compliance reporting, and flexible billing models for healthcare workloads.

## Architecture

### Metering Pattern: Autonomous Pre-Aggregator (Approach 3)

We use a dedicated **metering collector** deployment that runs independently and collects metrics from all service components:

```
┌─────────────────────────────────────────────────────────────┐
│                    FHIR Healthcare Instance                  │
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                    │
│  │  FHIR Server │      │  PostgreSQL  │                    │
│  │   (Metrics)  │      │  (Metrics)   │                    │
│  └──────┬───────┘      └──────┬───────┘                    │
│         │                     │                              │
│         └──────────┬──────────┘                             │
│                    ▼                                         │
│         ┌─────────────────────┐                             │
│         │ Metering Collector  │                             │
│         │  (Node.js Sidecar)  │                             │
│         └──────────┬──────────┘                             │
│                    │                                         │
└────────────────────┼─────────────────────────────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ Sovereign Core       │
          │ Metering API (XPM)   │
          └──────────────────────┘
```

**Benefits**:
- Complete separation from service logic
- Centralized metric collection
- Independent scaling and updates
- No impact on FHIR server performance
- Resilient to service restarts

---

## Metrics Collected

### 1. FHIR API Calls (`fhir_api_calls`)

**Type**: Cumulative  
**Unit**: requests  
**Description**: Total number of FHIR API requests across all resource types  
**Collection**: Hourly aggregation  
**Billing**: $0.001 per request

**Examples**:
- `GET /fhir/Patient/123`
- `POST /fhir/Claim`
- `PUT /fhir/Coverage/456`
- `DELETE /fhir/Observation/789`

### 2. FHIR Resources Stored (`fhir_resources_stored`)

**Type**: High-Watermark  
**Unit**: resources  
**Description**: Peak count of FHIR resources in database during the hour  
**Collection**: Hourly snapshot  
**Billing**: $0.0001 per resource

**Includes**:
- Patient resources
- Practitioner resources
- Organization resources
- Claim resources
- ClaimResponse resources
- Coverage resources
- All other FHIR resource types

### 3. Claims Processed (`claims_processed`)

**Type**: Cumulative  
**Unit**: claims  
**Description**: Number of healthcare claims submitted and processed  
**Collection**: Hourly aggregation  
**Billing**: $0.05 per claim

**Workflow**:
1. Claim created via FHIR API
2. CQL validation executed
3. Claim adjudication workflow
4. ClaimResponse generated

### 4. EDI X12 Transactions (`edi_transactions`)

**Type**: Cumulative  
**Unit**: transactions  
**Description**: EDI X12 transactions processed via webMethods  
**Collection**: Hourly aggregation  
**Billing**: $0.10 per transaction

**Transaction Types**:
- EDI X12 837 Professional Claim (outbound)
- EDI X12 835 Payment/Remittance (inbound)

### 5. Storage Bytes (`storage_bytes`)

**Type**: High-Watermark  
**Unit**: bytes  
**Description**: Peak storage consumed by FHIR resources  
**Collection**: Hourly snapshot  
**Billing**: $0.00000001 per byte (~$0.01 per GB)

**Includes**:
- PostgreSQL database size
- FHIR resource JSON storage
- Binary attachments
- Audit logs

### 6. Active Users (`active_users`)

**Type**: Point-in-Time  
**Unit**: users  
**Description**: Unique authenticated users at snapshot time  
**Collection**: Hourly snapshot  
**Billing**: $1.00 per user per hour

**User Types**:
- Healthcare providers
- Insurance payers
- System administrators
- Integration users

### 7. CQL Validations (`cql_validations`)

**Type**: Cumulative  
**Unit**: validations  
**Description**: Clinical Quality Language rule executions  
**Collection**: Hourly aggregation  
**Billing**: $0.002 per validation

**Use Cases**:
- Claim validation rules
- Coverage eligibility checks
- Clinical decision support
- Quality measure calculations

---

## Billing Models

### Model 1: Pay-As-You-Go (Recommended)

**Type**: Pure Usage-Based  
**Base Fee**: $0  
**Formula**: `Cost = Σ(Metric_Quantity × Unit_Price)`

**Pricing**:
- FHIR API Calls: $0.001 per request
- Claims Processed: $0.05 per claim
- EDI Transactions: $0.10 per transaction
- Storage: $0.01 per GB per hour
- CQL Validations: $0.002 per validation

**Best For**: Variable workloads, development/testing, cost optimization

**Example Monthly Cost** (10,000 patients, 1,000 claims):
```
FHIR API Calls:    1,000,000 × $0.001  = $1,000
Claims Processed:      1,000 × $0.05   = $50
EDI Transactions:      1,000 × $0.10   = $100
Storage:              50 GB × $0.01    = $50
CQL Validations:      5,000 × $0.002   = $10
─────────────────────────────────────────────
Total:                                  $1,210
```

### Model 2: Tiered with Overages

**Type**: Fixed Fee + Overages  
**Tiers**: Starter, Professional, Enterprise

#### Starter Tier
**Base Fee**: $100/month  
**Includes**:
- 100,000 FHIR API calls
- 1,000 claims processed
- 10 GB storage
- 10 active users

**Overage Rates**:
- FHIR API Calls: $0.001 per request
- Claims: $0.05 per claim
- Storage: $0.01 per GB
- Users: $1.00 per user

**Best For**: Small clinics, pilot projects

#### Professional Tier
**Base Fee**: $500/month  
**Includes**:
- 1,000,000 FHIR API calls
- 10,000 claims processed
- 100 GB storage
- 50 active users

**Overage Rates**:
- FHIR API Calls: $0.0008 per request (20% discount)
- Claims: $0.04 per claim (20% discount)
- Storage: $0.008 per GB (20% discount)
- Users: $0.80 per user (20% discount)

**Best For**: Medium healthcare organizations, regional payers

#### Enterprise Tier
**Base Fee**: $2,000/month  
**Includes**:
- 10,000,000 FHIR API calls
- 100,000 claims processed
- 1 TB storage
- 500 active users

**Overage Rates**:
- FHIR API Calls: $0.0005 per request (50% discount)
- Claims: $0.03 per claim (40% discount)
- Storage: $0.005 per GB (50% discount)
- Users: $0.50 per user (50% discount)

**Best For**: Large health systems, national payers

### Model 3: Hybrid Subscription

**Type**: Platform Fee + Usage  
**Base Fee**: $250/month  
**Includes**:
- Platform access
- Standard support
- 99.5% SLA
- Security updates

**Usage Charges**:
- All metrics billed at standard rates
- No included usage bundle
- Predictable base cost + variable usage

**Best For**: Enterprises requiring guaranteed support and SLA

---

## Metering Configuration

### Prerequisites

Before enabling metering, you must:

1. **Obtain Metering API Key**
   - Contact Sovereign Core administrators
   - Request product metering API key for `sccat-fhir-healthcare`
   - Exchange API key for metering token

2. **Create Metering Secret**
   ```bash
   kubectl create secret generic metering-credentials \
     --namespace=fhir-healthcare \
     --from-literal=metering-token='YOUR_METERING_TOKEN' \
     --from-literal=subaccount-id='YOUR_SUBACCOUNT_ID'
   ```

3. **Verify Metering Domain**
   - Default: `https://xpm.apps.cluster.local`
   - Update in metering-sidecar-deployment.yaml if different

### Deployment

The metering collector is automatically deployed with the service:

```yaml
# Included in base manifests
resources:
  - metering-sidecar-deployment.yaml
```

**Resource Requirements**:
- CPU: 100m request, 200m limit
- Memory: 128Mi request, 256Mi limit
- Replicas: 1 (single collector per instance)

### Configuration Parameters

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| Metering Domain | `METERING_DOMAIN` | `https://xpm.apps.cluster.local` | XPM API endpoint |
| Metering Token | `METERING_TOKEN` | (from secret) | Authentication token |
| Product ID | `PRODUCT_ID` | `sccat-fhir-healthcare` | Service identifier |
| Instance ID | `INSTANCE_ID` | (from labels) | Instance identifier |
| Subaccount ID | `SUBACCOUNT_ID` | (from secret) | Tenant subaccount |
| Collection Interval | `COLLECTION_INTERVAL` | `3600000` (1 hour) | Metric collection frequency (ms) |
| Submission Interval | `SUBMISSION_INTERVAL` | `3600000` (1 hour) | Metric submission frequency (ms) |

---

## Monitoring Metering

### View Collector Logs

```bash
# View metering collector logs
kubectl logs deployment/metering-collector -n fhir-healthcare

# Follow logs in real-time
kubectl logs -f deployment/metering-collector -n fhir-healthcare
```

**Expected Output**:
```
=== FHIR Healthcare Metering Collector ===
Product ID: sccat-fhir-healthcare
Instance ID: 019d070d-6a17-705d-b6a7-a00d75ed716f
Collection Interval: 3600000 ms
Submission Interval: 3600000 ms
==========================================
[Collector] Starting metrics collection...
[Collector] Collecting FHIR metrics...
[Collector] Total FHIR resources: 15234
[Collector] Collecting database metrics...
[Collector] Estimated storage: 31199232 bytes
[Collector] Metrics collected: {
  fhir_api_calls: 0,
  claims_processed: 1523,
  edi_transactions: 0,
  fhir_resources_stored: 15234,
  storage_bytes: 31199232,
  active_users: 0,
  cql_validations: 0
}
[Submitter] Starting metrics submission...
[Submitter] Submitting metrics: [...]
[Submitter] Metrics submitted successfully. Transaction ID: abc123...
```

### Verify Submissions

```bash
# Check recent submissions via Sovereign Core API
export METERING_DOMAIN="https://xpm.apps.cluster.local"
export METERING_TOKEN="your-token-here"
export INSTANCE_CRN="crn:v1:ibm-sc:private:sccat-fhir-healthcare:earth-1:sub/subaccount-id:instance-id::"
export CRN_ENCODED=$(printf %s "$INSTANCE_CRN" | jq -sRr @uri)

# Fetch last 24 hours of usage
export START_TIME=$(($(date +%s%3N) - 86400000))
export END_TIME=$(date +%s%3N)

curl --request GET \
  --url "$METERING_DOMAIN/api/v1/metering/usage/crns/$CRN_ENCODED?start=$START_TIME&end=$END_TIME" \
  --header "Authorization: Bearer $METERING_TOKEN" | jq
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No metrics submitted | Missing metering token | Create metering-credentials secret |
| 401 Unauthorized | Invalid/expired token | Regenerate token from API key |
| 403 Forbidden | Product ID mismatch | Verify PRODUCT_ID matches registration |
| 422 Unprocessable | Time outside 24h window | Check system clock synchronization |
| Collector crash loop | Missing FHIR server | Verify FHIR deployment is running |

---

## Testing Metering

### Local Testing (Without Sovereign Core)

For development and testing without Sovereign Core metering API:

1. **Disable Metering Token**
   ```bash
   # Collector will log metrics but skip submission
   kubectl delete secret metering-credentials -n fhir-healthcare
   ```

2. **View Collected Metrics**
   ```bash
   kubectl logs deployment/metering-collector -n fhir-healthcare | grep "Metrics collected"
   ```

3. **Simulate Metering API**
   ```bash
   # Run dummy metering endpoint (from catalogathon-meteringproxy)
   kubectl port-forward svc/metering-proxy 8080:8080 -n metering
   
   # Update METERING_DOMAIN
   kubectl set env deployment/metering-collector \
     METERING_DOMAIN=http://metering-proxy.metering.svc.cluster.local:8080 \
     -n fhir-healthcare
   ```

### Integration Testing

1. **Generate Test Load**
   ```bash
   # Create test patients
   for i in {1..100}; do
     curl -X POST http://fhir-server:8080/fhir/Patient \
       -H "Content-Type: application/json" \
       -d '{"resourceType":"Patient","name":[{"family":"Test'$i'"}]}'
   done
   
   # Create test claims
   for i in {1..50}; do
     curl -X POST http://fhir-server:8080/fhir/Claim \
       -H "Content-Type: application/json" \
       -d '{"resourceType":"Claim","status":"active","type":{"coding":[{"code":"professional"}]}}'
   done
   ```

2. **Wait for Collection Cycle**
   ```bash
   # Default: 1 hour, or force restart to trigger immediate collection
   kubectl rollout restart deployment/metering-collector -n fhir-healthcare
   ```

3. **Verify Metrics**
   ```bash
   kubectl logs deployment/metering-collector -n fhir-healthcare --tail=50
   ```

---

## Best Practices

### 1. Metric Accuracy
- Ensure system clocks are synchronized (NTP)
- Monitor collector health and restart on failures
- Validate metric calculations against known test data
- Store submission records for audit trail

### 2. Performance
- Default 1-hour intervals balance accuracy and overhead
- Adjust intervals based on workload characteristics
- Monitor collector resource usage
- Scale FHIR server independently of metering

### 3. Cost Optimization
- Monitor usage trends to select optimal billing model
- Set up alerts for unexpected usage spikes
- Review metrics monthly for optimization opportunities
- Consider tiered plans for predictable workloads

### 4. Compliance
- Retain metering logs for audit requirements
- Document metric definitions for stakeholders
- Implement dashboards for usage visibility
- Regular reconciliation with billing statements

### 5. Troubleshooting
- Enable debug logging for investigation
- Test metering in non-production first
- Implement retry logic with exponential backoff
- Monitor submission success rates

---

## Metering API Reference

### Submit Usage

**Endpoint**: `POST /api/v1/metering/resources/{instance_id}-subscription/usage`

**Headers**:
```
Authorization: Bearer {metering_token}
Content-Type: application/json
```

**Payload**:
```json
[{
  "crn": "crn:v1:ibm-sc:private:sccat-fhir-healthcare:earth-1:sub/{subaccount_id}:{instance_id}::",
  "meteringPlan": "metered",
  "start": 1773878400000,
  "end": 1773964740000,
  "measured_usage": [
    {
      "quantity": 1000,
      "measure": "fhir_api_calls",
      "meteringModel": "cumulative"
    },
    {
      "quantity": 15234,
      "measure": "fhir_resources_stored",
      "meteringModel": "high-watermark"
    }
  ]
}]
```

**Response**:
```json
{
  "transactionId": "abc123...",
  "status": "successful"
}
```

### Fetch Usage

**Endpoint**: `GET /api/v1/metering/usage/crns/{crn_encoded}?start={start}&end={end}`

**Headers**:
```
Authorization: Bearer {metering_token}
Content-Type: application/json
```

**Response**:
```json
{
  "crn": "crn:v1:ibm-sc:private:sccat-fhir-healthcare:...",
  "usage": [{
    "start": 1773878400000,
    "end": 1773964740000,
    "measures": [
      {
        "measure": "fhir_api_calls",
        "quantity": 1000,
        "meteringModel": "cumulative"
      }
    ]
  }]
}
```

---

## Support

For metering issues or questions:
- **Metering Logs**: `kubectl logs deployment/metering-collector -n fhir-healthcare`
- **Sovereign Core Support**: metering-support@sovereign-core.ibm.com
- **Documentation**: [reference-metering.md](../../../catalogathon-guide/reference/reference-metering.md)

---

**Metering Integration Version**: 1.0  
**Last Updated**: 2026-04-26  
**Made with Bob** 🤖