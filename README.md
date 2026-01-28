# Model as a Service (MaaS) - Deployment Guide for Fresh OpenShift Cluster

## Table of Contents
1. [Overview](#overview)
2. [What is Model as a Service](#what-is-model-as-a-service)
3. [Architecture](#architecture)
4. [Components](#components)
5. [Prerequisites](#prerequisites)
6. [Deployment Workflow](#deployment-workflow)
7. [Step-by-Step Deployment](#step-by-step-deployment)
8. [Verification](#verification)
9. [Post-Deployment Configuration](#post-deployment-configuration)
10. [Troubleshooting](#troubleshooting)
11. [References](#references)

---

## Overview

This guide provides complete instructions for deploying **Model as a Service (MaaS)** on a fresh OpenShift cluster. MaaS is a cloud-native platform built on Open Data Hub that provides policy-based access control, rate limiting, and tier-based subscriptions for AI model serving.

### Key Capabilities

- **Self-Service Access Control**: Users can request and manage their own API tokens
- **Tier-Based Subscriptions**: Different service levels (free, premium, enterprise) with distinct quotas
- **Rate Limiting**: Usage limits enforced per tier to manage costs
- **Centralized Gateway**: Single entry point for all model inference requests
- **Multi-Model Support**: Deploy and serve multiple LLM models (Qwen, Granite, Llama, etc.)
- **Cost Management**: Monitor and control inference costs through observability dashboards

---

## What is Model as a Service

**Model as a Service (MaaS)** extends Open Data Hub's capabilities by adding a management layer for self-service access control, rate limiting, and tier-based subscriptions. It enables organizations to:

- **Deliver AI models as shared resources** that users can access on demand
- **Provide standardized API endpoints** compatible with OpenAI API format
- **Share and access private AI models** at scale across the organization
- **Enforce governance policies** in real-time during inference requests
- **Monitor usage and costs** through comprehensive observability

### Status

MaaS is under active development and provided as a **developer preview**. It is not yet recommended for production workloads.

---

## Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                           External Clients                           │
│                     (Users, Applications, Tools)                     │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 │ HTTPS
                                 │
┌────────────────────────────────▼─────────────────────────────────────┐
│                      OpenShift Route / Ingress                       │
│                    (maas-default-gateway route)                      │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────┐
│                    GATEWAY & POLICY LAYER                            │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │           Envoy Gateway (maas-default-gateway)                 │ │
│  │                                                                │ │
│  │  • Single entry point for all traffic                         │ │
│  │  • Path-based routing                                         │ │
│  │  • TLS termination                                            │ │
│  └─────────────────┬──────────────────────────────────────────────┘ │
│                    │                                                 │
│         ┌──────────┴──────────┐                                      │
│         │                     │                                      │
│  ┌──────▼──────┐      ┌──────▼──────┐                              │
│  │  HTTPRoute  │      │  HTTPRoute  │                              │
│  │  /maas-api/*│      │  /v1/*      │                              │
│  └──────┬──────┘      └──────┬──────┘                              │
│         │                     │                                      │
│  ┌──────▼──────────────────────▼──────────────────────────────────┐ │
│  │      RHCL (Red Hat Connectivity Link) - Policy Engine         │ │
│  │                                                                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │ │
│  │  │  Authorino   │  │  Limitador   │  │ Cert Manager     │    │ │
│  │  │              │  │              │  │                  │    │ │
│  │  │ Authentication│  │ Rate Limiting│  │ TLS Certificates │    │ │
│  │  │ & AuthZ      │  │ Per Tier     │  │                  │    │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘    │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
                         │                     │
         ┌───────────────┴──────┐   ┌──────────▼─────────────┐
         │                      │   │                        │
┌────────▼──────────┐  ┌────────▼──────────┐  ┌─────────────▼──────────┐
│   MaaS API        │  │  RHOAI/ODH        │  │   Prometheus           │
│   Service         │  │  Model Serving    │  │   (Observability)      │
│                   │  │                   │  │                        │
│ • Token Gen       │  │ ┌───────────────┐ │  │ • Usage Metrics        │
│ • Tier Mapping    │  │ │ KServe        │ │  │ • Dashboards           │
│ • User Management │  │ │               │ │  │ • Monitoring           │
│                   │  │ │ InferenceServ │ │  └────────────────────────┘
│ ┌───────────────┐ │  │ │ ┌───────────┐ │ │
│ │ ConfigMap:    │ │  │ │ │Model:     │ │ │
│ │ tier-to-group │ │  │ │ │ Qwen      │ │ │
│ │ mapping       │ │  │ │ │ Granite   │ │ │
│ └───────────────┘ │  │ │ │ Llama     │ │ │
│                   │  │ │ │ Custom... │ │ │
│ Creates:          │  │ │ └───────────┘ │ │
│ • ServiceAccounts │  │ │               │ │
│ │ with tier labels│  │ │ Serverless    │ │
│ • Tokens (K8s API)│  │ │ Auto-scaling  │ │
└───────────────────┘  │ └───────────────┘ │
                       │                   │
                       │ ┌───────────────┐ │
                       │ │ Knative       │ │
                       │ │ Serving       │ │
                       │ └───────────────┘ │
                       │                   │
                       │ ┌───────────────┐ │
                       │ │ Istio Service │ │
                       │ │ Mesh          │ │
                       │ └───────────────┘ │
                       └───────────────────┘
```

### Request Flow Architecture

#### Token Generation Flow

```
┌────────────┐
│   User     │
│ (OpenShift)│
└─────┬──────┘
      │
      │ 1. POST /maas-api/token
      │    Authorization: Bearer <OpenShift Token>
      │    X-Forwarded-Access-Token: <OpenShift Token>
      ▼
┌─────────────────────┐
│ maas-default-gateway│
└─────────┬───────────┘
          │
          │ 2. Route to MaaS API
          ▼
┌─────────────────────┐
│   Authorino         │
│ (Authentication)    │
└─────────┬───────────┘
          │
          │ 3. Validate OpenShift token
          ▼
┌─────────────────────┐
│   MaaS API          │
│                     │
│ a. Extract user     │
│ b. Get K8s groups   │──────> tier-to-group-mapping
│ c. Determine tier   │        ConfigMap
│ d. Create SA        │
│ e. Request token    │──────> Kubernetes API
│                     │        (TokenRequest)
└─────────┬───────────┘
          │
          │ 4. Return service account token
          ▼
┌────────────┐
│   User     │
│ (Receives  │
│  SA token) │
└────────────┘
```

#### Inference Request Flow

```
┌────────────┐
│   Client   │
└─────┬──────┘
      │
      │ 1. POST /v1/chat/completions
      │    Authorization: Bearer <Service Account Token>
      ▼
┌─────────────────────┐
│ maas-default-gateway│
└─────────┬───────────┘
          │
          │ 2. Route to model serving
          ▼
┌─────────────────────┐
│   Authorino         │
│ (Authentication)    │
└─────────┬───────────┘
          │
          │ 3. Validate SA token
          │    Extract tier label
          ▼
┌─────────────────────┐
│   Limitador         │
│ (Rate Limiting)     │
└─────────┬───────────┘
          │
          │ 4. Check rate limit for tier
          │    (free: 10 req/min)
          │    (premium: 100 req/min)
          │    (enterprise: 1000 req/min)
          ▼
      [Allowed?]
          │
          ├─ No ──> 429 Too Many Requests
          │
          ├─ Yes
          ▼
┌─────────────────────┐
│ Kubernetes RBAC     │
│ Authorization       │
└─────────┬───────────┘
          │
          │ 5. Check permissions on
          │    LLMInferenceService
          ▼
┌─────────────────────┐
│   KServe            │
│ InferenceService    │
└─────────┬───────────┘
          │
          │ 6. Forward to model pod
          ▼
┌─────────────────────┐
│   Model Container   │
│   (vLLM/TGI/etc)    │
└─────────┬───────────┘
          │
          │ 7. Generate response
          ▼
┌────────────┐
│   Client   │
│ (Response) │
└────────────┘

    │
    └──> Metrics sent to Prometheus
```

### Component Interaction Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OPERATOR LAYER                               │
│                                                                     │
│  ┌───────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ OpenShift AI  │  │ Serverless   │  │ Service Mesh           │  │
│  │ Operator      │  │ Operator     │  │ Operator               │  │
│  └───────┬───────┘  └──────┬───────┘  └────────┬───────────────┘  │
│          │                  │                    │                  │
│          │ Manages          │ Manages            │ Manages          │
│          ▼                  ▼                    ▼                  │
│  ┌───────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │DataScienceClus│  │Knative Serving│  │ServiceMeshControlPlane │  │
│  │ter CR         │  │               │  │                        │  │
│  └───────┬───────┘  └──────┬───────┘  └────────┬───────────────┘  │
└──────────┼──────────────────┼──────────────────┼──────────────────┘
           │                  │                   │
           │ Creates          │ Creates           │ Creates
           ▼                  ▼                   ▼
┌──────────────────┐  ┌─────────────┐  ┌────────────────────────┐
│ KServe           │  │ Knative     │  │ Istio Pods             │
│ Controller       │  │ Pods        │  │ • istiod               │
│                  │  │ • activator │  │ • ingressgateway       │
│ Manages:         │  │ • autoscaler│  │ • egressgateway        │
│ InferenceService │  │ • controller│  │                        │
│ ServingRuntime   │  │ • webhook   │  │ Manages:               │
│ PredictorSpec    │  └─────────────┘  │ • Service mesh network │
└──────────────────┘                   │ • mTLS                 │
                                       │ • Traffic management   │
                                       └────────────────────────┘
```

---

## Components

### 1. OpenShift AI / Open Data Hub (Core Platform)

**Purpose**: Provides the foundational platform for data science and AI workloads

**Operator**: `opendatahub-operator` or `rhods-operator`

**Key Resources**:
- `DataScienceCluster`: Main CR that manages all components
- `DataScienceClusterInitialization`: Initialization configuration

**Managed Components**:
- KServe (model serving)
- Model Registry
- Data Science Pipelines
- Workbenches
- Model Mesh (optional)

**Namespace**: `opendatahub` or `redhat-ods-operator`

### 2. KServe (Model Serving)

**Purpose**: Serverless model inference platform with auto-scaling

**Version**: Integrated with OpenShift AI 3.0+

**Key Features**:
- **Serverless architecture**: Auto-scales to zero when idle
- **Multi-framework support**: PyTorch, TensorFlow, SKLearn, XGBoost, vLLM, Hugging Face
- **InferenceService**: Custom resource for deploying models
- **Predictor/Transformer/Explainer**: Model pipeline components

**Key Resources**:
- `InferenceService`: Defines model deployment
- `ServingRuntime`: Defines model server configuration
- `ClusterServingRuntime`: Cluster-wide runtime templates

**Dependencies**:
- Knative Serving (serverless runtime)
- Istio Service Mesh (networking)

### 3. Red Hat OpenShift Serverless (Knative)

**Purpose**: Provides serverless runtime for KServe

**Operator**: `serverless-operator`

**Components**:
- **Knative Serving**: Serverless application runtime
- **Knative Eventing** (optional): Event-driven architecture

**Key Resources**:
- `KnativeServing`: CR to deploy Knative Serving
- `Service`: Knative Service (auto-scaling workload)
- `Route`: Traffic routing
- `Revision`: Immutable snapshots of service configuration

**Namespaces**:
- `knative-serving`: Control plane
- Model namespaces: Application workloads

**Features**:
- Auto-scaling (scale to zero)
- Traffic splitting (blue/green, canary)
- Revision management

### 4. Red Hat OpenShift Service Mesh (Istio)

**Purpose**: Service mesh for network management and observability

**Operator**: `servicemeshoperator`

**Components**:
- **istiod**: Control plane for configuration and policies
- **istio-ingressgateway**: Ingress traffic management
- **istio-egressgateway**: Egress traffic management

**Key Resources**:
- `ServiceMeshControlPlane`: Main control plane CR
- `ServiceMeshMemberRoll`: Namespaces in the mesh
- `Gateway`: Istio gateway for ingress/egress
- `VirtualService`: Traffic routing rules
- `DestinationRule`: Load balancing policies

**Namespace**: `istio-system`

**Features**:
- mTLS (mutual TLS) between services
- Traffic management
- Observability (metrics, traces)
- Security policies

### 5. Red Hat Connectivity Link (RHCL) - Policy Engine

**Purpose**: Provides policy enforcement layer for authentication, authorization, and rate limiting

**Version**: 1.2+

**Components**:
- **Authorino**: Authentication and authorization service
- **Limitador**: Rate limiting service
- **Cert-manager**: Certificate management

**Key Resources**:
- `AuthorizationPolicy`: Defines authentication rules
- `RateLimitPolicy`: Defines rate limiting rules
- `Certificate`: TLS certificate requests

**Features**:
- **Token validation**: Validates OpenShift and Service Account tokens
- **RBAC integration**: Uses Kubernetes RBAC for authorization
- **Rate limiting**: Per-tier usage limits
- **Metrics**: Usage data sent to Prometheus

### 6. Envoy Gateway

**Purpose**: API gateway using Envoy proxy

**Component**: `maas-default-gateway`

**Key Resources**:
- `Gateway`: Main gateway resource
- `HTTPRoute`: HTTP routing rules
- `GRPCRoute`: gRPC routing rules (optional)

**Features**:
- Path-based routing (`/maas-api/*`, `/v1/*`)
- TLS termination
- Request/response transformation
- Policy attachment points

### 7. MaaS API Service

**Purpose**: Backend service for token management and user tier assignment

**Language**: Go

**Key Functions**:
- **Token generation**: Creates service accounts and requests tokens
- **Tier mapping**: Maps Kubernetes groups to service tiers
- **User management**: Tracks token issuance and usage

**Key Resources**:
- `Deployment`: MaaS API application
- `Service`: Internal service endpoint
- `ConfigMap`: `tier-to-group-mapping` for tier configuration
- `ServiceAccount`: For API pod authentication

**Endpoints**:
- `POST /maas-api/token`: Request new token
- `GET /maas-api/tokens`: List user's tokens

### 8. Observability Stack

**Purpose**: Monitoring and metrics collection

**Components**:
- **Prometheus**: Metrics collection and storage
- **Grafana**: Dashboards and visualization
- **Limitador metrics exporter**: Rate limit usage metrics

**Metrics Collected**:
- Request counts per tier
- Rate limit usage
- Model inference latency
- Token generation events
- Gateway traffic metrics

**Dashboards**:
- Tier usage overview
- Model performance
- Rate limit enforcement
- Cost tracking

---

## Prerequisites

### Infrastructure Requirements

1. **OpenShift Cluster**
   - Version: **4.19.9 or higher**
   - Cluster admin access
   - Internet connectivity for pulling images

2. **Resource Requirements** (Recommended)
   - **vCPUs**: 16+
   - **RAM**: 32 GB+
   - **Storage**: 100 GB+
   - **Worker nodes**: 3+ (for HA)

3. **Storage**
   - Dynamic storage provisioner configured
   - Default storage class set
   - Persistent volume support

4. **Network Requirements**
   - External route access (for gateway)
   - Internal service-to-service communication
   - Container image registry access

### Software Requirements

1. **CLI Tools**
   - `oc` (OpenShift CLI)
   - `kubectl` (Kubernetes CLI)
   - `kustomize` (v5.7.0+)
   - `jq` (JSON processor)
   - `gsed` (macOS only - GNU sed)

2. **Permissions**
   - Cluster admin role
   - Ability to create projects/namespaces
   - Ability to install operators via OperatorHub

### Pre-Installation Checks

Run these commands to verify prerequisites:

```bash
# Check cluster version
oc version

# Check node resources
oc describe nodes | grep -A 5 "Allocatable"

# Check storage classes
oc get storageclass

# Verify cluster admin access
oc auth can-i create namespaces --all-namespaces

# Check CLI tools
kustomize version
jq --version
```

---

## Deployment Workflow

### Overview

The deployment follows this sequence:

```
1. Install Base Operators (Manual via OperatorHub)
   └─> OpenShift Serverless
   └─> OpenShift Service Mesh
   └─> OpenShift AI / Open Data Hub

2. Configure Service Mesh Control Plane
   └─> Deploy Istio components
   └─> Verify istiod, ingress/egress gateways

3. Configure Knative Serving
   └─> Deploy Knative serving components
   └─> Configure integration with Istio

4. Enable KServe
   └─> Update DataScienceCluster CR
   └─> Deploy KServe controller

5. Deploy MaaS Platform (Automated Script)
   └─> Install RHCL (Cert-manager, Authorino, Limitador)
   └─> Create Gateway and HTTPRoutes
   └─> Deploy MaaS API service
   └─> Configure policies (AuthZ, RateLimit)
   └─> Deploy sample models (optional)

6. Verify Deployment
   └─> Check all pods running
   └─> Test token generation
   └─> Test model inference

7. Configure Tiers and Models
   └─> Customize tier mappings
   └─> Deploy production models
   └─> Set up monitoring dashboards
```

### Deployment Timeline

| Stage | Component | Duration | Dependencies |
|-------|-----------|----------|--------------|
| 1 | Operator Installation | 5-10 min | None |
| 2 | Service Mesh Setup | 10-15 min | Serverless Operator |
| 3 | Knative Serving | 5-10 min | Service Mesh |
| 4 | KServe Enable | 5 min | Knative, Service Mesh |
| 5 | MaaS Deployment | 10-20 min | All above |
| 6 | Verification | 5-10 min | MaaS deployed |
| 7 | Configuration | 10-30 min | User preferences |

**Total**: 50-100 minutes for complete deployment

---

## Step-by-Step Deployment

### Phase 1: Install Base Operators

#### Step 1.1: Install OpenShift Serverless Operator

1. **Access OperatorHub**
   ```bash
   oc get packagemanifests -n openshift-marketplace | grep serverless
   ```

2. **Install via Web Console**:
   - Navigate to: Operators → OperatorHub
   - Search: "Red Hat OpenShift Serverless"
   - Click "Install"
   - Accept defaults:
     - Installation mode: All namespaces
     - Update channel: stable
     - Approval strategy: Automatic
   - Click "Install"

3. **Verify Installation**:
   ```bash
   oc get csv -n openshift-serverless
   ```

   Expected output: `serverless-operator.vX.Y.Z` with phase `Succeeded`

#### Step 1.2: Install OpenShift Service Mesh Operator

1. **Install via Web Console**:
   - Navigate to: Operators → OperatorHub
   - Search: "Red Hat OpenShift Service Mesh"
   - Click "Install"
   - Accept defaults
   - Click "Install"

2. **Verify Installation**:
   ```bash
   oc get csv -n openshift-operators | grep servicemesh
   ```

   Expected: `servicemeshoperator.vX.Y.Z` with phase `Succeeded`

#### Step 1.3: Install OpenShift AI Operator

For **Red Hat OpenShift AI** (RHOAI):
1. **Install via Web Console**:
   - Navigate to: Operators → OperatorHub
   - Search: "Red Hat OpenShift AI"
   - Click "Install"
   - Accept defaults
   - Click "Install"

2. **Verify Installation**:
   ```bash
   oc get csv -n redhat-ods-operator
   ```

For **Open Data Hub** (ODH):
1. **Install via Web Console**:
   - Search: "Open Data Hub Operator"
   - Install similarly

2. **Verify**:
   ```bash
   oc get csv -n opendatahub
   ```

### Phase 2: Configure Service Mesh

#### Step 2.1: Create Service Mesh Control Plane

OpenShift AI automatically creates `DataScienceClusterInitialization`, which triggers Service Mesh deployment.

1. **Verify DataScienceClusterInitialization**:
   ```bash
   oc get datascienceclusterinitializations -A
   ```

2. **Check ServiceMeshControlPlane Creation**:
   ```bash
   oc get servicemeshcontrolplane -n istio-system
   ```

3. **Wait for Istio Components**:
   ```bash
   # Wait for istiod
   oc wait --for=condition=ready pod -l app=istiod -n istio-system --timeout=300s

   # Wait for ingress gateway
   oc wait --for=condition=ready pod -l app=istio-ingressgateway -n istio-system --timeout=300s

   # Wait for egress gateway
   oc wait --for=condition=ready pod -l app=istio-egressgateway -n istio-system --timeout=300s
   ```

4. **Verify All Istio Pods Running**:
   ```bash
   oc get pods -n istio-system
   ```

   Expected pods:
   - `istiod-*` (Running)
   - `istio-ingressgateway-*` (Running)
   - `istio-egressgateway-*` (Running)

### Phase 3: Configure Knative Serving

#### Step 3.1: Create DataScienceCluster

1. **Create DataScienceCluster CR**:

   Create file `dsc.yaml`:
   ```yaml
   apiVersion: datasciencecluster.opendatahub.io/v1
   kind: DataScienceCluster
   metadata:
     name: default-dsc
   spec:
     components:
       kserve:
         managementState: Managed
         serving:
           ingressGateway:
             certificate:
               type: SelfSigned
           managementState: Managed
       dashboard:
         managementState: Managed
       workbenches:
         managementState: Managed
       datasciencepipelines:
         managementState: Removed
       modelmeshserving:
         managementState: Removed
       codeflare:
         managementState: Removed
       ray:
         managementState: Removed
       kueue:
         managementState: Removed
   ```

2. **Apply the CR**:
   ```bash
   oc apply -f dsc.yaml
   ```

3. **Verify Knative Serving Deployment**:
   ```bash
   oc get pods -n knative-serving
   ```

   Expected pods:
   - `activator-*`
   - `autoscaler-*`
   - `autoscaler-hpa-*`
   - `controller-*`
   - `net-istio-controller-*`
   - `net-istio-webhook-*`
   - `webhook-*`

### Phase 4: Enable KServe

KServe is enabled in the DataScienceCluster CR above with `managementState: Managed`.

1. **Verify KServe Controller**:
   ```bash
   oc get pods -n opendatahub | grep kserve
   # OR for RHOAI
   oc get pods -n redhat-ods-operator | grep kserve
   ```

2. **Check KServe CRDs**:
   ```bash
   oc get crd | grep inferenceservice
   oc get crd | grep servingruntime
   ```

   Expected CRDs:
   - `inferenceservices.serving.kserve.io`
   - `servingruntimes.serving.kserve.io`
   - `clusterservingruntimes.serving.kserve.io`

### Phase 5: Deploy MaaS Platform

#### Step 5.1: Clone MaaS Repository

```bash
git clone https://github.com/opendatahub-io/models-as-a-service.git
cd models-as-a-service
```

#### Step 5.2: Run Deployment Script

```bash
# Set release version (use latest release tag)
export MAAS_REF="v0.0.2"

# Run deployment script
./scripts/deploy-rhoai-stable.sh
```

**What the script does**:
1. Installs **Cert-manager operator**
2. Installs **LWS (Limitador) operator**
3. Installs **Red Hat Connectivity Link** operator
4. Creates `maas-default-gateway` Gateway resource
5. Creates HTTPRoutes for `/maas-api/*` and `/v1/*`
6. Deploys **MaaS API** service
7. Creates authentication and rate limiting policies
8. Configures tier-to-group mappings

#### Step 5.3: Monitor Deployment

```bash
# Watch script output
# The script will display progress for each component

# In another terminal, monitor pods
watch oc get pods -A | grep -E "maas|limitador|authorino|cert-manager"
```

**Expected namespaces created**:
- `cert-manager`: Certificate management
- `limitador-system`: Rate limiting
- `authorino`: Authentication service
- `maas`: MaaS API and gateway

#### Step 5.4: Verify MaaS Components

```bash
# Check Gateway
oc get gateway -A

# Check HTTPRoutes
oc get httproute -A

# Check MaaS API deployment
oc get deployment -n maas

# Check all MaaS pods
oc get pods -n maas
oc get pods -n limitador-system
oc get pods -n authorino
oc get pods -n cert-manager
```

### Phase 6: Deploy Sample Models (Optional)

The deployment includes sample models for testing.

#### Option A: Deploy Simulator Model (CPU only)

```bash
cd deployment/components/models
kustomize build simulator | oc apply -f -
```

#### Option B: Deploy Facebook OPT-125M (CPU)

```bash
kustomize build facebook-opt-125m | oc apply -f -
```

#### Option C: Deploy Qwen3 (Requires GPU)

```bash
kustomize build qwen3 | oc apply -f -
```

**Verify Model Deployment**:
```bash
# Check InferenceService
oc get inferenceservice -A

# Check model pods
oc get pods -n <model-namespace>

# Wait for model to be ready
oc get inferenceservice <model-name> -n <namespace> -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

---

## Verification

### Verify All Components

#### 1. Check Operators

```bash
# All operators should show Succeeded
oc get csv -A | grep -E "serverless|servicemesh|opendatahub|rhods"
```

#### 2. Check Core Services

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

#### 3. Check MaaS Platform

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

#### 4. Check Gateway Route

```bash
# Get external URL
oc get route -n maas

# Should show route like:
# maas-default-gateway-<hash>.apps.<cluster-domain>
```

### Test Token Generation

#### 1. Login to OpenShift

```bash
oc login --server=https://api.<cluster-domain>:6443 --username=<user>
```

#### 2. Get OpenShift Token

```bash
TOKEN=$(oc whoami -t)
echo $TOKEN
```

#### 3. Request MaaS Token

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

### Test Model Inference

#### 1. Use MaaS Token

```bash
# Save the token from previous step
MAAS_TOKEN="<token-from-response>"

# Get gateway URL
GATEWAY_URL=$(oc get route -n maas -o jsonpath='{.items[0].spec.host}')
```

#### 2. Send Inference Request

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

### Test Rate Limiting

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

## Post-Deployment Configuration

### Configure Tier Mappings

#### 1. View Current Tier Configuration

```bash
oc get configmap tier-to-group-mapping -n maas -o yaml
```

#### 2. Edit Tier Mappings

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

#### 3. Restart MaaS API

```bash
oc rollout restart deployment/maas-api -n maas
```

### Add User Groups

```bash
# Add user to premium group
oc adm groups new premium-users
oc adm groups add-users premium-users user1

# Add user to enterprise group
oc adm groups new enterprise-users
oc adm groups add-users enterprise-users admin-user
```

### Deploy Production Models

#### 1. Create Model Namespace

```bash
oc new-project production-models
```

#### 2. Add Namespace to Service Mesh

```bash
oc label namespace production-models istio-injection=enabled
```

#### 3. Create ServingRuntime

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

#### 4. Create InferenceService

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

#### 5. Configure RBAC for Model Access

```bash
# Allow premium tier to access model
oc create role model-access --verb=get --resource=inferenceservices -n production-models
oc create rolebinding premium-model-access \
  --role=model-access \
  --serviceaccount=maas:premium-sa-* \
  -n production-models
```

### Configure Monitoring

#### 1. Access Grafana

```bash
# Get Grafana route
oc get route grafana -n openshift-monitoring

# Login with OpenShift credentials
```

#### 2. Import MaaS Dashboards

The deployment includes pre-built dashboards for:
- Tier usage overview
- Rate limit metrics
- Model performance
- Gateway traffic

Import from: `deployment/observability/dashboards/`

### Configure TLS Certificates

#### Option A: Self-Signed (Default)

Already configured during deployment.

#### Option B: Custom Certificate

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

## Troubleshooting

### [Common Issues](https://github.com/ishuverma/PrivateMaaS-Openshift/blob/main/CommonIssues.md)
### [Debugging Commands](https://github.com/ishuverma/PrivateMaaS-Openshift/blob/main/Debugging.md)



### Log Collection

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

---

## References

### Official Documentation

- **OpenDataHub MaaS Project**: https://github.com/opendatahub-io/models-as-a-service
  - Complete source code and deployment manifests

- **OpenDataHub Documentation**: https://opendatahub.io/docs
  - Getting Started: https://opendatahub.io/docs/getting-started-with-open-data-hub/
  - Deploying Models: https://opendatahub.io/docs/deploying-models/
  - Model Serving Platform: https://opendatahub.io/docs/configuring-your-model-serving-platform/
  - Architecture: https://opendatahub.io/docs/architecture/

- **Red Hat Developer**:
  - Models-as-a-Service in OpenShift AI: https://developers.redhat.com/articles/2025/11/25/introducing-models-service-openshift-ai
  - Installing KServe with ODH: https://developers.redhat.com/articles/2024/06/27/how-install-kserve-using-open-data-hub

### Component Documentation

- **KServe**: https://kserve.github.io/website/
- **Knative**: https://knative.dev/docs/
- **Istio**: https://istio.io/latest/docs/
- **Kuadrant**: https://docs.kuadrant.io/
- **Authorino**: https://docs.kuadrant.io/authorino/
- **Limitador**: https://docs.kuadrant.io/limitador/

### OpenShift Documentation

- **OpenShift Serverless**: https://docs.openshift.com/serverless/
- **OpenShift Service Mesh**: https://docs.openshift.com/service-mesh/
- **OpenShift AI**: https://access.redhat.com/documentation/en-us/red_hat_openshift_ai/

### GitHub Repositories

- **models-as-a-service**: https://github.com/opendatahub-io/models-as-a-service
- **opendatahub-operator**: https://github.com/opendatahub-io/opendatahub-operator
- **kserve**: https://github.com/opendatahub-io/kserve

### Version Information

- **OpenShift**: 4.19.9+
- **OpenShift AI / ODH**: 3.0+
- **Red Hat Connectivity Link**: 1.2+
- **KServe**: Integrated with OpenShift AI
- **MaaS Platform**: v0.0.2 (as of this guide)

---

## Appendix: Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL USERS                                 │
│                                                                             │
│    ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐     │
│    │  User 1  │      │  User 2  │      │  App 1   │      │  Tool    │     │
│    │  (Free)  │      │(Premium) │      │(Enterpr.)│      │          │     │
│    └────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘     │
└─────────┼──────────────────┼──────────────────┼──────────────────┼─────────┘
          │                  │                  │                  │
          │ 1. Request Token │                  │ 2. Use Token    │
          │ (OpenShift Auth) │                  │ (Inference)     │
          └──────────────────┴──────────────────┴──────────────────┘
                                      │
                                      │ HTTPS
                                      ▼
          ┌──────────────────────────────────────────────────────┐
          │          OpenShift Route (External DNS)              │
          │      maas-gateway.apps.cluster.example.com           │
          └──────────────────────┬───────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────────────┐
│                        GATEWAY & POLICY LAYER                                │
│                          (Namespace: maas)                                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              Envoy Gateway (maas-default-gateway)                   │    │
│  │                                                                     │    │
│  │  ┌───────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐    │    │
│  │  │ Listener  │  │  TLS     │  │  Routing │  │  Policy        │    │    │
│  │  │ :443      │─>│ Terminate│─>│  Engine  │─>│  Attachment    │    │    │
│  │  └───────────┘  └──────────┘  └──────────┘  └────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                             │                    │                          │
│                  ┌──────────┴────────┐  ┌────────▼──────────┐               │
│                  │                   │  │                   │               │
│  ┌───────────────▼────────┐  ┌───────▼──────────┐  ┌───────▼──────────┐    │
│  │   HTTPRoute            │  │   HTTPRoute      │  │   HTTPRoute      │    │
│  │   /maas-api/*          │  │   /v1/models     │  │   /v1/chat/*     │    │
│  │                        │  │                  │  │                  │    │
│  │ ┌────────────────────┐ │  │ ┌──────────────┐ │  │ ┌──────────────┐ │    │
│  │ │AuthorizationPolicy │ │  │ │AuthzPolicy   │ │  │ │AuthzPolicy   │ │    │
│  │ │- OpenShift Token   │ │  │ │- SA Token    │ │  │ │- SA Token    │ │    │
│  │ └────────────────────┘ │  │ └──────────────┘ │  │ └──────────────┘ │    │
│  │                        │  │                  │  │                  │    │
│  │                        │  │ ┌──────────────┐ │  │ ┌──────────────┐ │    │
│  │                        │  │ │RateLimitPol. │ │  │ │RateLimitPol. │ │    │
│  │                        │  │ │- Per Tier    │ │  │ │- Per Tier    │ │    │
│  │                        │  │ └──────────────┘ │  │ └──────────────┘ │    │
│  └────────────┬───────────┘  └────────┬─────────┘  └────────┬─────────┘    │
│               │                       │                      │              │
│               │                       │                      │              │
│  ┌────────────▼────────────────┐  ┌───▼──────────────────────▼──────────┐  │
│  │   RHCL Components           │  │   RHCL Components                   │  │
│  │                             │  │                                     │  │
│  │  ┌─────────────────────┐    │  │  ┌─────────────────────┐            │  │
│  │  │    Authorino        │    │  │  │    Authorino        │            │  │
│  │  │  (Authentication)   │    │  │  │  (Authentication)   │            │  │
│  │  │                     │    │  │  │                     │            │  │
│  │  │ • Validate Tokens   │    │  │  │ • Validate SA Token │            │  │
│  │  │ • Extract Groups    │    │  │  │ • Extract Tier Label│            │  │
│  │  └─────────────────────┘    │  │  └─────────────────────┘            │  │
│  │                             │  │                                     │  │
│  │                             │  │  ┌─────────────────────┐            │  │
│  │                             │  │  │    Limitador        │            │  │
│  │                             │  │  │  (Rate Limiting)    │            │  │
│  │                             │  │  │                     │            │  │
│  │                             │  │  │ • Check Quotas      │            │  │
│  │                             │  │  │ • Enforce Limits    │            │  │
│  │                             │  │  │ • Send Metrics ────>│──────┐     │  │
│  └─────────────┬───────────────┘  │  └─────────────────────┘      │     │  │
└────────────────┼──────────────────┴──────────────┬─────────────────┼─────────┘
                 │                                 │                 │
                 │                                 │                 │
┌────────────────▼───────────────┐  ┌──────────────▼─────────────┐   │
│       MaaS API Service         │  │    KServe / Model Serving  │   │
│     (Namespace: maas)          │  │   (Multiple Namespaces)    │   │
│                                │  │                            │   │
│  ┌──────────────────────────┐  │  │  ┌──────────────────────┐  │   │
│  │  Go Backend Service      │  │  │  │  InferenceService 1  │  │   │
│  │                          │  │  │  │  (Qwen3)             │  │   │
│  │  Endpoints:              │  │  │  │                      │  │   │
│  │  • POST /token           │  │  │  │  ┌────────────────┐  │  │   │
│  │  • GET /tokens           │  │  │  │  │ ServingRuntime │  │  │   │
│  │                          │  │  │  │  │ (vLLM)         │  │  │   │
│  │  Logic:                  │  │  │  │  └────────────────┘  │  │   │
│  │  1. Validate user token  │  │  │  └──────────────────────┘  │   │
│  │  2. Get user groups ────>│──┐│  │                            │   │
│  │  3. Map to tier          │  ││  │  ┌──────────────────────┐  │   │
│  │  4. Create SA with label │  ││  │  │  InferenceService 2  │  │   │
│  │  5. Request K8s token    │  ││  │  │  (Granite)           │  │   │
│  │  6. Return token + tier  │  ││  │  │                      │  │   │
│  └──────────────────────────┘  ││  │  │  ┌────────────────┐  │  │   │
│                                │││  │  │  │ ServingRuntime │  │  │   │
│  ┌──────────────────────────┐  │││  │  │  │ (TGI)          │  │  │   │
│  │  ConfigMap               │  │││  │  │  └────────────────┘  │  │   │
│  │  tier-to-group-mapping   │<─┘││  │  └──────────────────────┘  │   │
│  │                          │   ││  │                            │   │
│  │  free:                   │   ││  │  All backed by:            │   │
│  │    - system:auth'd       │   ││  │  ┌──────────────────────┐  │   │
│  │    - 10 req/min          │   ││  │  │  Knative Serving     │  │   │
│  │  premium:                │   ││  │  │  • Auto-scaling      │  │   │
│  │    - premium-users       │   ││  │  │  • Scale to zero     │  │   │
│  │    - 100 req/min         │   ││  │  └──────────────────────┘  │   │
│  │  enterprise:             │   ││  │                            │   │
│  │    - enterprise-users    │   ││  │  Networking by:            │   │
│  │    - 1000 req/min        │   ││  │  ┌──────────────────────┐  │   │
│  └──────────────────────────┘   ││  │  │  Istio Service Mesh  │  │   │
│                                  ││  │  │  • mTLS              │  │   │
│  Service Account Created:        ││  │  │  • Routing           │  │   │
│  ┌──────────────────────────┐   ││  │  │  • Observability     │  │   │
│  │ SA: user1-sa-<timestamp> │   ││  │  └──────────────────────┘  │   │
│  │ Labels:                  │   ││  └────────────────────────────┘   │
│  │   tier: free             │   ││                                   │
│  │ Token: JWT (K8s signed)  │   ││  ┌────────────────────────────┐   │
│  │ Expires: 24h             │   ││  │  Kubernetes RBAC           │   │
│  └──────────────────────────┘   ││  │                            │   │
└──────────────────────────────────┘│  │  RoleBindings per tier:    │   │
                                    │  │  • free -> read-only       │   │
                                    │  │  • premium -> full access  │   │
                                    └──│  • enterprise -> all models│   │
                                       └────────────────────────────┘   │
                                                                        │
┌───────────────────────────────────────────────────────────────────────▼─────┐
│                         OBSERVABILITY STACK                                 │
│                                                                             │
│  ┌────────────────┐     ┌────────────────┐     ┌─────────────────────┐    │
│  │  Prometheus    │     │    Grafana     │     │  Limitador Metrics  │    │
│  │                │────>│                │     │  Exporter           │    │
│  │  • Metrics     │     │  • Dashboards  │     │                     │    │
│  │  • Alerts      │     │  • Tier Usage  │     │  • Request Counts   │    │
│  │  • Storage     │     │  • Model Perf  │     │  • Rate Limits      │    │
│  └────────────────┘     └────────────────┘     └─────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                           OPERATOR LAYER                                    │
│                                                                             │
│  ┌───────────────────┐  ┌──────────────────┐  ┌────────────────────────┐   │
│  │ OpenShift AI      │  │ Serverless       │  │ Service Mesh           │   │
│  │ Operator          │  │ Operator         │  │ Operator               │   │
│  │                   │  │                  │  │                        │   │
│  │ Manages:          │  │ Manages:         │  │ Manages:               │   │
│  │ • DSC             │  │ • KnativeServing │  │ • SMCP                 │   │
│  │ • DSCI            │  │ • Eventing       │  │ • SMMR                 │   │
│  │ • KServe          │  │                  │  │ • Gateways             │   │
│  └───────────────────┘  └──────────────────┘  └────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

This guide provides a complete reference for deploying **Model as a Service (MaaS)** on a fresh OpenShift cluster. The deployment includes:

### Components Deployed
1. **OpenShift Serverless** - Serverless runtime for auto-scaling models
2. **OpenShift Service Mesh** - Network management and observability
3. **OpenShift AI / Open Data Hub** - AI platform foundation
4. **KServe** - Model serving with auto-scaling
5. **Red Hat Connectivity Link (RHCL)** - Policy enforcement (Authorino, Limitador, Cert-manager)
6. **MaaS Platform** - API gateway, token management, tier-based access control

### Key Features
- **Self-service token generation** with tier-based access
- **Rate limiting** per subscription tier (free, premium, enterprise)
- **Centralized API gateway** for all model access
- **OpenAI-compatible API** for easy integration
- **Comprehensive observability** with Prometheus and Grafana
- **Multi-model support** with serverless auto-scaling

### Deployment Time
Approximately **50-100 minutes** for complete deployment on a fresh cluster.

### Next Steps
After deployment:
1. Configure tier mappings for your organization
2. Deploy production models
3. Set up monitoring dashboards
4. Distribute gateway URL to users
5. Configure custom TLS certificates
6. Set up backup and disaster recovery

For issues or questions, refer to the Troubleshooting section or consult the official documentation linked in the References section.
