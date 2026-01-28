# Debugging Commands

```bash
# View all MaaS resources
oc get all -n maas

# Check all InferenceServices
oc get inferenceservice -A

# View Gateway configuration
oc get gateway maas-default-gateway -n maas -o yaml

# View HTTPRoutes
oc get httproute -A -o yaml

# Check authentication policies
oc get authorizationpolicy -A

# Check rate limit policies
oc get ratelimitpolicy -A

# View all events
oc get events -A --sort-by='.lastTimestamp' | tail -50

# Check Prometheus metrics
oc port-forward -n openshift-monitoring prometheus-k8s-0 9090:9090
# Then access http://localhost:9090
```
