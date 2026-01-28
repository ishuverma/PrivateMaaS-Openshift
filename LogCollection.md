# Log Collection

```bash
# Collect logs from all MaaS components
mkdir -p maas-logs

# MaaS API
oc logs -n maas deployment/maas-api > maas-logs/maas-api.log

# Authorino
oc logs -n authorino deployment/authorino > maas-logs/authorino.log

# Limitador
oc logs -n limitador-system deployment/limitador > maas-logs/limitador.log

# Gateway (Envoy)
oc logs -n maas deployment/maas-default-gateway > maas-logs/gateway.log

# KServe controller
oc logs -n opendatahub deployment/kserve-controller-manager > maas-logs/kserve.log
```
