# Verify All Components

## 1. Check Operators

```bash
# All operators should show Succeeded
oc get csv -A | grep -E "serverless|servicemesh|opendatahub|rhods"
```

## 2. Check Core Services

```bash
# Istio
oc get pods -n istio-system
# Expected: istiod, ingress/egress gateways (all Running)

# Knative
oc get pods -n knative-serving
# Expected: activator, autoscaler, controller, net-istio-* (all Running)

# KServe
oc get pods -n opendatahub | grep kserve
# OR
oc get pods -n redhat-ods-operator | grep kserve
# Expected: kserve-controller-manager (Running)
```

## 3. Check MaaS Platform

```bash
# MaaS components
oc get pods -n maas
oc get pods -n limitador-system
oc get pods -n authorino
oc get pods -n cert-manager

# Gateway
oc get gateway maas-default-gateway -n maas

# HTTPRoutes
oc get httproute -n maas

# Policies
oc get authorizationpolicy -A
oc get ratelimitpolicy -A
```

## 4. Check Gateway Route

```bash
# Get external URL
oc get route -n maas

# Should show route like:
# maas-default-gateway-<hash>.apps.<cluster-domain>
```

## Test Token Generation

### 1. Login to OpenShift

```bash
oc login --server=https://api.<cluster-domain>:6443 --username=<user>
```

### 2. Get OpenShift Token

```bash
TOKEN=$(oc whoami -t)
echo $TOKEN
```

### 3. Request MaaS Token

```bash
# Get gateway URL
GATEWAY_URL=$(oc get route -n maas -o jsonpath='{.items[0].spec.host}')

# Request token
curl -X POST https://${GATEWAY_URL}/maas-api/token \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "X-Forwarded-Access-Token: ${TOKEN}" \
  -k | jq
```

**Expected response**:
```json
{
  "token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjQ4...",
  "tier": "free",
  "expiresAt": "2026-01-28T00:00:00Z"
}
```

## Test Model Inference

### 1. Use MaaS Token

```bash
# Save the token from previous step
MAAS_TOKEN="<token-from-response>"

# Get gateway URL
GATEWAY_URL=$(oc get route -n maas -o jsonpath='{.items[0].spec.host}')
```

### 2. Send Inference Request

```bash
curl -X POST https://${GATEWAY_URL}/v1/chat/completions \
  -H "Authorization: Bearer ${MAAS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3",
    "messages": [
      {
        "role": "user",
        "content": "Hello, how are you?"
      }
    ]
  }' \
  -k | jq
```

**Expected response**:
```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1706400000,
  "model": "qwen3",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! I'm doing well, thank you for asking..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 25,
    "total_tokens": 35
  }
}
```

## Test Rate Limiting

```bash
# Send multiple requests rapidly to trigger rate limit
for i in {1..15}; do
  curl -X POST https://${GATEWAY_URL}/v1/chat/completions \
    -H "Authorization: Bearer ${MAAS_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{"model":"qwen3","messages":[{"role":"user","content":"Test"}]}' \
    -k
  echo ""
done
```

**Expected**: After 10 requests (free tier limit), you should receive:
```
HTTP/1.1 429 Too Many Requests
```

---
