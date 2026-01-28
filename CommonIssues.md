# Common Issues
## 1. Istio Pods Not Starting

**Symptom**: `istiod` or gateway pods in `CrashLoopBackOff`

**Diagnosis**:
```bash
oc logs -n istio-system deployment/istiod
oc describe pod -n istio-system <pod-name>
```

**Solutions**:
- Check node resources (CPU/memory)
- Verify ServiceMeshControlPlane is in Ready state
- Restart control plane:
  ```bash
  oc delete pod -n istio-system -l app=istiod
  ```

## 2. KServe InferenceService Not Ready

**Symptom**: InferenceService stuck in `NotReady` state

**Diagnosis**:
```bash
oc get inferenceservice <name> -n <namespace> -o yaml
oc get ksvc -n <namespace>
oc describe ksvc <name> -n <namespace>
```

**Common causes**:
- Image pull failures
- Insufficient resources
- Storage mount issues

**Solutions**:
```bash
# Check pod logs
oc logs -n <namespace> <pod-name> -c kserve-container

# Check events
oc get events -n <namespace> --sort-by='.lastTimestamp'

# Verify storage
oc get pvc -n <namespace>
```

## 3. Token Generation Fails

**Symptom**: 401 or 403 errors when requesting tokens

**Diagnosis**:
```bash
# Check MaaS API logs
oc logs -n maas deployment/maas-api

# Check Authorino logs
oc logs -n authorino deployment/authorino
```

**Solutions**:
- Verify OpenShift token is valid: `oc whoami -t`
- Check user has proper groups: `oc get user <username> -o yaml`
- Verify tier mapping: `oc get configmap tier-to-group-mapping -n maas`
- Restart MaaS API: `oc rollout restart deployment/maas-api -n maas`

## 4. Rate Limiting Not Working

**Symptom**: Requests not throttled even when exceeding limits

**Diagnosis**:
```bash
# Check Limitador logs
oc logs -n limitador-system deployment/limitador

# Check RateLimitPolicy
oc get ratelimitpolicy -A -o yaml
```

**Solutions**:
- Verify Limitador is running
- Check rate limit policy is attached to HTTPRoute
- Verify tier label on service account:
  ```bash
  oc get serviceaccount <sa-name> -n maas -o yaml | grep tier
  ```

## 5. Gateway Not Accessible

**Symptom**: Cannot reach gateway URL externally

**Diagnosis**:
```bash
# Check route
oc get route -n maas

# Check gateway status
oc get gateway maas-default-gateway -n maas -o yaml

# Check listener status
oc describe gateway maas-default-gateway -n maas
```

**Solutions**:
- Verify route is created and has host
- Check gateway listeners are programmed
- Test internal access:
  ```bash
  oc run test --image=curlimages/curl -it --rm -- \
    curl http://maas-default-gateway.maas.svc.cluster.local
  ```

## 6. Model Inference Returns Errors

**Symptom**: 500, 503, or timeout errors during inference

**Diagnosis**:
```bash
# Check model pod status
oc get pods -n <model-namespace>

# Check model logs
oc logs -n <model-namespace> <pod-name> -c kserve-container

# Check Knative service
oc get ksvc -n <model-namespace>

# Check revision status
oc get revision -n <model-namespace>
```

**Solutions**:
- Verify model pod is Running
- Check model loaded successfully (check logs)
- Verify sufficient resources:
  ```bash
  oc describe pod -n <model-namespace> <pod-name>
  ```
- Check autoscaler:
  ```bash
  oc get podautoscaler -n <model-namespace>
  ```
