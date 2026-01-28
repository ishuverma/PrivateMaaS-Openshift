# Post-Deployment Configuration
## Configure Tier Mappings

### 1. View Current Tier Configuration

```bash
oc get configmap tier-to-group-mapping -n maas -o yaml
```

### 2. Edit Tier Mappings

```bash
oc edit configmap tier-to-group-mapping -n maas
```

**Example configuration**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tier-to-group-mapping
  namespace: maas
data:
  mapping.yaml: |
    tiers:
      free:
        groups:
          - "system:authenticated"
        rateLimit:
          requestsPerMinute: 10
      premium:
        groups:
          - "premium-users"
        rateLimit:
          requestsPerMinute: 100
      enterprise:
        groups:
          - "enterprise-users"
        rateLimit:
          requestsPerMinute: 1000
```

### 3. Restart MaaS API

```bash
oc rollout restart deployment/maas-api -n maas
```

## Add User Groups

```bash
# Add user to premium group
oc adm groups new premium-users
oc adm groups add-users premium-users user1

# Add user to enterprise group
oc adm groups new enterprise-users
oc adm groups add-users enterprise-users admin-user
```

## Deploy Production Models

### 1. Create Model Namespace

```bash
oc new-project production-models
```

### 2. Add Namespace to Service Mesh

```bash
oc label namespace production-models istio-injection=enabled
```

### 3. Create ServingRuntime

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: vllm-runtime
  namespace: production-models
spec:
  supportedModelFormats:
    - name: pytorch
      version: "1"
      autoSelect: true
  containers:
    - name: kserve-container
      image: quay.io/opendatahub/vllm:latest
      args:
        - --model
        - /mnt/models
        - --tensor-parallel-size
        - "1"
      resources:
        requests:
          cpu: "2"
          memory: 8Gi
        limits:
          cpu: "4"
          memory: 16Gi
```

### 4. Create InferenceService

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: granite-7b
  namespace: production-models
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      runtime: vllm-runtime
      storage:
        path: s3://models/granite-7b
        key: aws-secret
```

### 5. Configure RBAC for Model Access

```bash
# Allow premium tier to access model
oc create role model-access --verb=get --resource=inferenceservices -n production-models
oc create rolebinding premium-model-access \
  --role=model-access \
  --serviceaccount=maas:premium-sa-* \
  -n production-models
```

## Configure Monitoring

### 1. Access Grafana

```bash
# Get Grafana route
oc get route grafana -n openshift-monitoring

# Login with OpenShift credentials
```

### 2. Import MaaS Dashboards

The deployment includes pre-built dashboards for:
- Tier usage overview
- Rate limit metrics
- Model performance
- Gateway traffic

Import from: `deployment/observability/dashboards/`

## Configure TLS Certificates

### Option A: Self-Signed (Default)

Already configured during deployment.

### Option B: Custom Certificate

```bash
# Create TLS secret
oc create secret tls custom-gateway-cert \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key \
  -n maas

# Update Gateway to use custom cert
oc edit gateway maas-default-gateway -n maas
```

Update certificate reference:
```yaml
spec:
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: custom-gateway-cert
```

---
