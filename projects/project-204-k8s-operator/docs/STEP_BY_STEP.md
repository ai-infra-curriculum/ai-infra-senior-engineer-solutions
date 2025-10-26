# TrainingJob Kubernetes Operator - Step-by-Step Implementation Guide

**Project 204: Custom Kubernetes Operator for ML Training Jobs**

This comprehensive guide walks you through building a production-ready Kubernetes operator from scratch using Python and the Kopf framework. You'll learn how to extend Kubernetes with custom resources, implement controller patterns, manage distributed ML training workloads, and build enterprise-grade infrastructure automation.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Learning Objectives](#learning-objectives)
3. [Prerequisites](#prerequisites)
4. [Architecture Deep Dive](#architecture-deep-dive)
5. [Phase 1: Project Setup and CRD](#phase-1-project-setup-and-crd)
6. [Phase 2: Basic Operator with Kopf](#phase-2-basic-operator-with-kopf)
7. [Phase 3: Resource Builders](#phase-3-resource-builders)
8. [Phase 4: Controllers Implementation](#phase-4-controllers-implementation)
9. [Phase 5: Status Management and Monitoring](#phase-5-status-management-and-monitoring)
10. [Phase 6: Checkpoint Management](#phase-6-checkpoint-management)
11. [Phase 7: Fault Tolerance and Retry Logic](#phase-7-fault-tolerance-and-retry-logic)
12. [Phase 8: Production Deployment](#phase-8-production-deployment)
13. [Phase 9: Testing Strategy](#phase-9-testing-strategy)
14. [Phase 10: Monitoring and Observability](#phase-10-monitoring-and-observability)
15. [Troubleshooting Guide](#troubleshooting-guide)
16. [Best Practices](#best-practices)
17. [Advanced Topics](#advanced-topics)

---

## Overview

### What You'll Build

A production-ready Kubernetes operator that:
- Extends Kubernetes API with a `TrainingJob` custom resource
- Automates deployment of distributed ML training workloads
- Manages GPU allocation across multiple nodes
- Handles fault tolerance with automatic checkpoint recovery
- Provides real-time status updates and metrics
- Integrates with MLflow for experiment tracking

### Why This Matters

Kubernetes operators are the gold standard for automating complex stateful applications. Building an ML training operator teaches you:

**Technical Skills:**
- Kubernetes API programming and controller patterns
- Custom Resource Definitions (CRDs)
- Operator frameworks (Kopf)
- Distributed systems coordination
- State machine implementation
- Error handling in asynchronous systems

**ML Infrastructure Skills:**
- Automating training job lifecycle
- GPU resource management
- Distributed training coordination
- Checkpoint management patterns
- Observability for ML workloads

**Production Engineering:**
- Building resilient automation
- Implementing retry logic and backoff
- Monitoring and alerting
- Production deployment patterns

---

## Learning Objectives

By completing this implementation, you will:

### Core Competencies
- ✅ Design and implement custom Kubernetes operators
- ✅ Create and manage Custom Resource Definitions (CRDs)
- ✅ Implement controller reconciliation loops
- ✅ Handle distributed ML training orchestration
- ✅ Manage GPU resources in Kubernetes
- ✅ Build fault-tolerant systems with checkpointing

### Advanced Skills
- ✅ Use Kopf framework for operator development
- ✅ Implement complex state machines
- ✅ Handle Kubernetes API interactions programmatically
- ✅ Design resource builders for declarative infrastructure
- ✅ Implement production-grade error handling
- ✅ Add comprehensive observability

### Production Patterns
- ✅ RBAC and security best practices
- ✅ Resource cleanup with finalizers
- ✅ Status subresources and conditions
- ✅ Event recording and logging
- ✅ Prometheus metrics integration
- ✅ Testing strategies for operators

---

## Prerequisites

### Required Knowledge

**Kubernetes (Advanced):**
- CRDs and custom resources
- Controllers and reconciliation loops
- RBAC, ServiceAccounts, Roles
- Jobs, Pods, Services, ConfigMaps
- GPU scheduling with device plugins

**Python (Intermediate to Advanced):**
- Async/await patterns
- Type hints and dataclasses
- Exception handling
- Testing with pytest
- Logging and structured logs

**ML Training (Intermediate):**
- Distributed training concepts
- PyTorch/TensorFlow basics
- GPU training fundamentals
- Checkpoint/restore patterns

### Required Tools

```bash
# Kubernetes cluster
minikube start --cpus=4 --memory=8192 --gpus=all
# OR use GKE/EKS/AKS with GPU node pools

# Python 3.11+
python --version  # >= 3.11

# kubectl
kubectl version --client

# Docker
docker --version

# GPU support (optional for full testing)
nvidia-smi  # Verify NVIDIA GPU access
```

### Required Libraries

```bash
pip install kopf kubernetes pydantic prometheus-client
```

---

## Architecture Deep Dive

### System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Control Plane                      │
│                                                                   │
│  API Server ←→ etcd (stores TrainingJob resources)              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ Watch/Update
                             │
┌────────────────────────────▼────────────────────────────────────┐
│               TrainingJob Operator (Our Code)                    │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Kopf Framework                          │  │
│  │  - Event Loop                                             │  │
│  │  - Handler Registration                                   │  │
│  │  - Retry/Backoff Logic                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │             Event Handlers (our code)                     │  │
│  │                                                            │  │
│  │  @kopf.on.create()  ──► create_handler()                 │  │
│  │  @kopf.on.update()  ──► update_handler()                 │  │
│  │  @kopf.on.delete()  ──► delete_handler()                 │  │
│  │  @kopf.timer()      ──► reconcile_handler() [every 30s]  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────┐ ┌─────────────────┐ ┌───────────────────┐  │
│  │JobController  │ │StatusController │ │CheckpointCtrl    │  │
│  │- Create Jobs  │ │- Monitor Status │ │- Manage Checkpts │  │
│  │- Manage Svcs  │ │- Update Metrics │ │- Handle Recovery │  │
│  └───────────────┘ └─────────────────┘ └───────────────────┘  │
│                                                                   │
│  ┌───────────────┐ ┌─────────────────┐ ┌───────────────────┐  │
│  │JobBuilder     │ │ServiceBuilder   │ │ConfigMapBuilder  │  │
│  │- Build Job    │ │- Build Service  │ │- Build ConfigMap │  │
│  │  Specs        │ │  Specs          │ │  Specs           │  │
│  └───────────────┘ └─────────────────┘ └───────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ Create/Update/Delete Resources
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  Created Kubernetes Resources                    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Job (parallelism=4, for 4-worker training)             │   │
│  │    ├─ Pod: worker-0 [GPU: 2, Rank: 0, MASTER_ADDR]     │   │
│  │    ├─ Pod: worker-1 [GPU: 2, Rank: 1]                  │   │
│  │    ├─ Pod: worker-2 [GPU: 2, Rank: 2]                  │   │
│  │    └─ Pod: worker-3 [GPU: 2, Rank: 3]                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Headless Service (training-job-workers)                │   │
│  │    - Stable DNS for worker-0, worker-1, worker-2, ...   │   │
│  │    - Enables distributed training communication         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ConfigMap (training-config)                            │   │
│  │    - Hyperparameters                                     │   │
│  │    - Network settings (MASTER_ADDR, MASTER_PORT)        │   │
│  │    - Framework config (NCCL, GLOO)                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  PersistentVolumeClaim (checkpoints)                    │   │
│  │    - Shared storage for model checkpoints               │   │
│  │    - Enables resume on failure                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Controller Pattern

The operator implements the standard Kubernetes controller pattern:

1. **Watch**: Monitor TrainingJob resources for changes
2. **Compare**: Compare desired state (spec) with actual state (status)
3. **Act**: Create/update/delete Kubernetes resources to match desired state
4. **Update Status**: Reflect actual state in TrainingJob status field
5. **Repeat**: Continuously reconcile until desired state achieved

### State Machine

```
┌─────────┐
│ Created │  User creates TrainingJob resource
└────┬────┘
     │
     ▼
┌─────────┐
│ Pending │  Operator validates spec, allocates resources
└────┬────┘
     │
     ├──► [Validation Failed] ──► Failed
     │
     ▼
┌──────────────┐
│ Initializing │  Creating K8s Job, Service, ConfigMap
└──────┬───────┘
       │
       ├──► [Resource Creation Failed] ──► Failed
       │
       ▼
┌─────────┐
│ Running │  All workers started, training in progress
└────┬────┘
     │
     ├──► [Pod Failure, retries < backoffLimit] ──► Restarting
     ├──► [Pod Failure, retries >= backoffLimit] ──► Failed
     ├──► [User Suspension] ──► Suspended
     │
     ▼
┌───────────┐
│ Completed │  All epochs finished successfully
└───────────┘
```

---

## Phase 1: Project Setup and CRD

### Step 1.1: Project Structure

Create the directory structure:

```bash
project-204-k8s-operator/
├── src/
│   ├── operator/
│   │   └── main.py                # Kopf operator entry point
│   ├── controllers/
│   │   ├── __init__.py
│   │   ├── job_controller.py      # Job lifecycle management
│   │   ├── status_controller.py   # Status updates
│   │   └── checkpoint_controller.py # Checkpoint management
│   ├── resources/
│   │   ├── __init__.py
│   │   ├── job_builder.py         # K8s Job spec builder
│   │   ├── service_builder.py     # Service spec builder
│   │   └── configmap_builder.py   # ConfigMap spec builder
│   ├── models/
│   │   ├── __init__.py
│   │   └── trainingjob.py         # Pydantic models for validation
│   └── utils/
│       ├── __init__.py
│       ├── k8s_client.py          # Kubernetes API wrapper
│       ├── logger.py              # Structured logging
│       └── metrics.py             # Prometheus metrics
├── kubernetes/
│   ├── base/
│   │   ├── trainingjob-crd.yaml   # Custom Resource Definition
│   │   ├── rbac.yaml              # ServiceAccount, Role, RoleBinding
│   │   ├── deployment.yaml        # Operator deployment
│   │   └── service.yaml           # Operator service (for metrics)
│   └── overlays/
│       └── with-monitoring/
│           └── servicemonitor.yaml # Prometheus ServiceMonitor
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── examples/
│   ├── trainingjob-simple.yaml
│   ├── trainingjob-distributed.yaml
│   └── trainingjob-gpu.yaml
├── Dockerfile
├── requirements.txt
└── README.md
```

### Step 1.2: Define the CRD

Create `kubernetes/base/trainingjob-crd.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: trainingjobs.ml.example.com
spec:
  group: ml.example.com
  names:
    kind: TrainingJob
    listKind: TrainingJobList
    plural: trainingjobs
    singular: trainingjob
    shortNames:
      - tj
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - model
                - dataset
              properties:
                model:
                  type: string
                  description: Model architecture to train
                dataset:
                  type: string
                  description: Dataset to use for training
                numWorkers:
                  type: integer
                  minimum: 1
                  maximum: 32
                  default: 1
                  description: Number of distributed training workers
                gpusPerWorker:
                  type: integer
                  minimum: 0
                  maximum: 8
                  default: 1
                  description: Number of GPUs per worker
                hyperparameters:
                  type: object
                  description: Training hyperparameters
                  properties:
                    learningRate:
                      type: number
                    batchSize:
                      type: integer
                    epochs:
                      type: integer
                  additionalProperties: true
                checkpoint:
                  type: object
                  description: Checkpoint configuration
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    frequency:
                      type: integer
                      default: 1
                      description: Checkpoint every N epochs
                    storage:
                      type: object
                      properties:
                        type:
                          type: string
                          enum: [pvc, s3, gcs]
                          default: pvc
                        path:
                          type: string
                resources:
                  type: object
                  description: Resource requests/limits per worker
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                    limits:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                backoffLimit:
                  type: integer
                  minimum: 0
                  maximum: 10
                  default: 3
                  description: Number of retries before marking as failed
            status:
              type: object
              properties:
                state:
                  type: string
                  enum:
                    - Pending
                    - Initializing
                    - Running
                    - Completed
                    - Failed
                    - Suspended
                progress:
                  type: string
                  description: Training progress (e.g., "45%")
                currentEpoch:
                  type: integer
                  description: Current training epoch
                totalEpochs:
                  type: integer
                  description: Total epochs to train
                metrics:
                  type: object
                  description: Latest training metrics
                  properties:
                    loss:
                      type: number
                    accuracy:
                      type: number
                  additionalProperties: true
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      reason:
                        type: string
                      message:
                        type: string
                      lastTransitionTime:
                        type: string
                        format: date-time
                startTime:
                  type: string
                  format: date-time
                completionTime:
                  type: string
                  format: date-time
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: State
          type: string
          jsonPath: .status.state
        - name: Progress
          type: string
          jsonPath: .status.progress
        - name: Epoch
          type: integer
          jsonPath: .status.currentEpoch
        - name: Workers
          type: integer
          jsonPath: .spec.numWorkers
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
```

**Key Points:**
- **openAPIV3Schema**: Validates TrainingJob specs at API level
- **subresources.status**: Enables separate updates to status field
- **additionalPrinterColumns**: Shows useful info in `kubectl get trainingjob`

### Step 1.3: Apply the CRD

```bash
kubectl apply -f kubernetes/base/trainingjob-crd.yaml

# Verify CRD is registered
kubectl get crd trainingjobs.ml.example.com

# Check it accepts resources
kubectl explain trainingjob.spec
```

---

## Phase 2: Basic Operator with Kopf

### Step 2.1: Understand Kopf Framework

Kopf (Kubernetes Operator Pythonic Framework) handles:
- Watching Kubernetes resources
- Invoking handlers on events (create, update, delete)
- Retry logic and error handling
- Status updates and patching
- Finalizers for cleanup

### Step 2.2: Create Operator Entry Point

Create `src/operator/main.py`:

```python
"""
TrainingJob Kubernetes Operator

This operator manages the lifecycle of ML training jobs on Kubernetes.
Built with Kopf framework for robust event handling and reconciliation.
"""

import kopf
import logging
from typing import Dict, Any
from kubernetes import client, config

# Import our controllers
from controllers.job_controller import JobController
from controllers.status_controller import StatusController
from controllers.checkpoint_controller import CheckpointController
from utils.logger import setup_logging
from utils.metrics import OperatorMetrics

# Setup logging
setup_logging()
logger = logging.getLogger(__name__)

# Initialize Kubernetes client
try:
    config.load_incluster_config()  # When running in cluster
except config.ConfigException:
    config.load_kube_config()  # When running locally

# Initialize controllers
job_controller = JobController()
status_controller = StatusController()
checkpoint_controller = CheckpointController()

# Initialize metrics
metrics = OperatorMetrics()


@kopf.on.startup()
def configure(settings: kopf.OperatorSettings, **_):
    """Configure operator startup settings"""
    settings.persistence.finalizer = 'trainingjob.ml.example.com/finalizer'
    settings.persistence.progress_storage = kopf.AnnotationsProgressStorage()
    settings.persistence.diffbase_storage = kopf.AnnotationsDiffBaseStorage()

    # Posting settings
    settings.posting.level = logging.INFO
    settings.posting.enabled = True

    # Watch settings
    settings.watching.server_timeout = 600
    settings.watching.client_timeout = 660

    logger.info("TrainingJob operator started")
    metrics.increment_operator_starts()


@kopf.on.create('ml.example.com', 'v1', 'trainingjobs')
def create_handler(
    spec: Dict[str, Any],
    name: str,
    namespace: str,
    logger: kopf.Logger,
    **kwargs
) -> Dict[str, Any]:
    """
    Handle TrainingJob creation.

    This is called when a new TrainingJob resource is created.
    We validate the spec and create necessary Kubernetes resources.
    """
    logger.info(f"Creating TrainingJob: {namespace}/{name}")

    try:
        # Validate spec
        _validate_spec(spec)

        # Create Kubernetes resources (Job, Service, ConfigMap)
        job_controller.create_training_job(
            name=name,
            namespace=namespace,
            spec=spec
        )

        # Update status to Initializing
        status_controller.update_state(
            name=name,
            namespace=namespace,
            state='Initializing',
            message='Training resources created, waiting for workers to start'
        )

        # Record Kubernetes event
        _record_event(
            name=name,
            namespace=namespace,
            reason='Created',
            message='TrainingJob resources created successfully'
        )

        metrics.increment_trainingjobs_created()

        return {'message': 'TrainingJob created successfully'}

    except Exception as e:
        logger.error(f"Failed to create TrainingJob: {e}")
        metrics.increment_trainingjobs_failed()

        # Update status to Failed
        status_controller.update_state(
            name=name,
            namespace=namespace,
            state='Failed',
            message=f'Creation failed: {str(e)}'
        )

        raise kopf.PermanentError(f"Failed to create TrainingJob: {e}")


@kopf.on.update('ml.example.com', 'v1', 'trainingjobs')
def update_handler(
    spec: Dict[str, Any],
    old: Dict[str, Any],
    new: Dict[str, Any],
    name: str,
    namespace: str,
    logger: kopf.Logger,
    **kwargs
):
    """
    Handle TrainingJob updates.

    Note: Most fields are immutable once training starts.
    We only allow updates to certain fields like resource limits.
    """
    logger.info(f"Updating TrainingJob: {namespace}/{name}")

    # Check what changed
    spec_diff = _get_spec_diff(old.get('spec', {}), new.get('spec', {}))

    if not spec_diff:
        logger.info("No spec changes detected")
        return

    # Validate allowed updates
    immutable_fields = {'model', 'dataset', 'numWorkers', 'gpusPerWorker'}
    changed_immutable = immutable_fields.intersection(spec_diff.keys())

    if changed_immutable:
        raise kopf.PermanentError(
            f"Cannot update immutable fields: {changed_immutable}"
        )

    # Handle allowed updates
    if 'checkpoint' in spec_diff:
        checkpoint_controller.update_checkpoint_config(
            name=name,
            namespace=namespace,
            config=spec['checkpoint']
        )

    logger.info(f"Updated TrainingJob fields: {list(spec_diff.keys())}")


@kopf.on.delete('ml.example.com', 'v1', 'trainingjobs')
def delete_handler(
    spec: Dict[str, Any],
    name: str,
    namespace: str,
    logger: kopf.Logger,
    **kwargs
):
    """
    Handle TrainingJob deletion.

    Clean up all created resources: Job, Service, ConfigMap, PVC (optional)
    """
    logger.info(f"Deleting TrainingJob: {namespace}/{name}")

    try:
        # Delete Kubernetes resources
        job_controller.delete_training_job(
            name=name,
            namespace=namespace
        )

        # Optionally delete checkpoints
        if spec.get('checkpoint', {}).get('deleteOnCompletion', False):
            checkpoint_controller.delete_checkpoints(
                name=name,
                namespace=namespace
            )

        metrics.increment_trainingjobs_deleted()

        logger.info(f"TrainingJob {namespace}/{name} deleted successfully")

    except Exception as e:
        logger.error(f"Error during deletion: {e}")
        # Allow deletion to proceed even if cleanup fails
        raise


@kopf.timer(
    'ml.example.com', 'v1', 'trainingjobs',
    interval=30.0,  # Run every 30 seconds
    idle=60.0       # Stop after 60s of no changes
)
def reconcile_handler(
    spec: Dict[str, Any],
    status: Dict[str, Any],
    name: str,
    namespace: str,
    logger: kopf.Logger,
    **kwargs
):
    """
    Periodic reconciliation loop.

    Runs every 30 seconds to:
    - Check actual state vs desired state
    - Update TrainingJob status
    - Handle failures and retries
    """
    current_state = status.get('state', 'Unknown')

    # Don't reconcile if already in terminal state
    if current_state in ('Completed', 'Failed'):
        return

    try:
        # Get actual Job status
        job_status = job_controller.get_job_status(
            name=name,
            namespace=namespace
        )

        # Update TrainingJob status based on Job status
        new_state = _determine_state(job_status, spec)

        if new_state != current_state:
            status_controller.update_state(
                name=name,
                namespace=namespace,
                state=new_state,
                message=f'Job status: {job_status}'
            )

        # Update metrics if training is running
        if new_state == 'Running':
            training_metrics = _fetch_training_metrics(name, namespace)
            if training_metrics:
                status_controller.update_metrics(
                    name=name,
                    namespace=namespace,
                    metrics=training_metrics
                )

    except Exception as e:
        logger.error(f"Reconciliation error: {e}")
        # Don't fail, will retry in next cycle


# Helper functions

def _validate_spec(spec: Dict[str, Any]):
    """Validate TrainingJob spec"""
    required_fields = ['model', 'dataset']
    for field in required_fields:
        if field not in spec:
            raise ValueError(f"Missing required field: {field}")

    num_workers = spec.get('numWorkers', 1)
    if num_workers < 1 or num_workers > 32:
        raise ValueError(f"numWorkers must be between 1 and 32, got {num_workers}")

    gpus_per_worker = spec.get('gpusPerWorker', 1)
    if gpus_per_worker < 0 or gpus_per_worker > 8:
        raise ValueError(f"gpusPerWorker must be between 0 and 8, got {gpus_per_worker}")


def _get_spec_diff(old_spec: Dict, new_spec: Dict) -> Dict:
    """Get differences between old and new spec"""
    diff = {}
    for key in new_spec:
        if key not in old_spec or old_spec[key] != new_spec[key]:
            diff[key] = new_spec[key]
    return diff


def _determine_state(job_status: Dict, spec: Dict) -> str:
    """Determine TrainingJob state from Job status"""
    if not job_status:
        return 'Pending'

    conditions = job_status.get('conditions', [])
    active = job_status.get('active', 0)
    succeeded = job_status.get('succeeded', 0)
    failed = job_status.get('failed', 0)

    # Check if completed
    total_workers = spec.get('numWorkers', 1)
    if succeeded >= total_workers:
        return 'Completed'

    # Check if failed
    backoff_limit = spec.get('backoffLimit', 3)
    if failed > backoff_limit:
        return 'Failed'

    # Check if running
    if active > 0:
        return 'Running'

    return 'Initializing'


def _fetch_training_metrics(name: str, namespace: str) -> Dict[str, float]:
    """Fetch training metrics from worker pods"""
    # This would query pod logs or metrics endpoint
    # Simplified implementation
    return {
        'loss': 0.0,
        'accuracy': 0.0
    }


def _record_event(
    name: str,
    namespace: str,
    reason: str,
    message: str,
    event_type: str = 'Normal'
):
    """Record Kubernetes event"""
    # Kubernetes event recording
    pass


if __name__ == '__main__':
    # Run the operator
    kopf.run()
```

**Key Concepts:**

1. **Handlers**: Decorated functions that respond to events
   - `@kopf.on.create`: Called when resource created
   - `@kopf.on.update`: Called when resource updated
   - `@kopf.on.delete`: Called when resource deleted
   - `@kopf.timer`: Periodic reconciliation

2. **Error Handling**:
   - `kopf.PermanentError`: Don't retry
   - `kopf.TemporaryError`: Retry with backoff
   - Regular exceptions: Retry automatically

3. **Finalizers**: Ensure cleanup before deletion

### Step 2.3: Setup Logging

Create `src/utils/logger.py`:

```python
"""Structured logging configuration"""

import logging
import sys
from pythonjsonlogger import jsonlogger


def setup_logging(level=logging.INFO):
    """Configure structured JSON logging"""

    # Create logger
    logger = logging.getLogger()
    logger.setLevel(level)

    # Clear existing handlers
    logger.handlers = []

    # Create console handler
    handler = logging.StreamHandler(sys.stdout)
    handler.setLevel(level)

    # JSON formatter
    formatter = jsonlogger.JsonFormatter(
        '%(asctime)s %(name)s %(levelname)s %(message)s',
        rename_fields={
            'asctime': 'timestamp',
            'levelname': 'level',
            'name': 'logger'
        }
    )

    handler.setFormatter(formatter)
    logger.addHandler(handler)

    return logger
```

### Step 2.4: Test Basic Operator

```bash
# Run operator locally
cd project-204-k8s-operator
python -m src.operator.main

# In another terminal, create a test TrainingJob
kubectl apply -f - <<EOF
apiVersion: ml.example.com/v1
kind: TrainingJob
metadata:
  name: test-job
  namespace: default
spec:
  model: resnet50
  dataset: imagenet
  numWorkers: 2
  gpusPerWorker: 1
  hyperparameters:
    learningRate: 0.001
    batchSize: 32
    epochs: 10
EOF

# Watch operator logs
# Should see "Creating TrainingJob: default/test-job"

# Check TrainingJob status
kubectl get trainingjob test-job
kubectl describe trainingjob test-job
```

---

## Phase 3: Resource Builders

Now we implement the builders that construct Kubernetes resource specifications.

### Step 3.1: Job Builder

Create `src/resources/job_builder.py`:

```python
"""
Kubernetes Job builder for distributed training workloads.

This module constructs Job specifications with proper:
- GPU allocation
- Distributed training environment variables
- Network configuration
- Volume mounts for checkpoints
"""

from typing import Dict, Any, List
from kubernetes import client


class JobBuilder:
    """Build Kubernetes Job specs for training workloads"""

    def __init__(self):
        self.api = client.BatchV1Api()

    def build_job_spec(
        self,
        name: str,
        namespace: str,
        spec: Dict[str, Any]
    ) -> client.V1Job:
        """
        Build complete Job specification.

        Args:
            name: TrainingJob name
            namespace: Kubernetes namespace
            spec: TrainingJob spec

        Returns:
            V1Job: Complete Job specification
        """
        num_workers = spec.get('numWorkers', 1)
        gpus_per_worker = spec.get('gpusPerWorker', 1)

        # Build pod template
        pod_template = self._build_pod_template(
            name=name,
            namespace=namespace,
            spec=spec
        )

        # Build Job spec
        job_spec = client.V1JobSpec(
            parallelism=num_workers,
            completions=num_workers,
            backoff_limit=spec.get('backoffLimit', 3),
            template=pod_template
        )

        # Build Job metadata
        metadata = client.V1ObjectMeta(
            name=f"{name}-job",
            namespace=namespace,
            labels={
                'app': 'trainingjob',
                'trainingjob': name,
                'component': 'worker'
            }
        )

        # Complete Job
        job = client.V1Job(
            api_version='batch/v1',
            kind='Job',
            metadata=metadata,
            spec=job_spec
        )

        return job

    def _build_pod_template(
        self,
        name: str,
        namespace: str,
        spec: Dict[str, Any]
    ) -> client.V1PodTemplateSpec:
        """Build pod template for worker pods"""

        # Container specification
        container = self._build_container(name, spec)

        # Pod metadata
        pod_metadata = client.V1ObjectMeta(
            labels={
                'app': 'trainingjob',
                'trainingjob': name,
                'component': 'worker'
            },
            annotations={
                'prometheus.io/scrape': 'true',
                'prometheus.io/port': '8000'
            }
        )

        # Pod spec
        pod_spec = client.V1PodSpec(
            containers=[container],
            restart_policy='OnFailure',
            volumes=self._build_volumes(name, spec),
            # GPU node selector if GPUs requested
            node_selector=self._build_node_selector(spec),
            # Tolerations for GPU nodes
            tolerations=self._build_tolerations(spec),
            # Service account
            service_account_name=f"{name}-worker",
            # DNS config for distributed training
            subdomain=f"{name}-workers",
            hostname='$(POD_NAME)'  # Set by downward API
        )

        return client.V1PodTemplateSpec(
            metadata=pod_metadata,
            spec=pod_spec
        )

    def _build_container(
        self,
        name: str,
        spec: Dict[str, Any]
    ) -> client.V1Container:
        """Build worker container spec"""

        model = spec['model']
        dataset = spec['dataset']
        hyperparams = spec.get('hyperparameters', {})
        num_workers = spec.get('numWorkers', 1)
        gpus_per_worker = spec.get('gpusPerWorker', 1)

        # Environment variables for distributed training
        env_vars = [
            # Downward API - Pod name and IP
            client.V1EnvVar(
                name='POD_NAME',
                value_from=client.V1EnvVarSource(
                    field_ref=client.V1ObjectFieldSelector(
                        field_path='metadata.name'
                    )
                )
            ),
            client.V1EnvVar(
                name='POD_IP',
                value_from=client.V1EnvVarSource(
                    field_ref=client.V1ObjectFieldSelector(
                        field_path='status.podIP'
                    )
                )
            ),
            # Training configuration
            client.V1EnvVar(name='MODEL', value=model),
            client.V1EnvVar(name='DATASET', value=dataset),
            client.V1EnvVar(name='NUM_WORKERS', value=str(num_workers)),
            client.V1EnvVar(name='GPUS_PER_WORKER', value=str(gpus_per_worker)),
            # Distributed training settings
            client.V1EnvVar(name='WORLD_SIZE', value=str(num_workers)),
            client.V1EnvVar(name='MASTER_ADDR', value=f"{name}-workers-0.{name}-workers"),
            client.V1EnvVar(name='MASTER_PORT', value='29500'),
            # NCCL settings for GPU training
            client.V1EnvVar(name='NCCL_DEBUG', value='INFO'),
            client.V1EnvVar(name='NCCL_SOCKET_IFNAME', value='eth0'),
        ]

        # Add hyperparameters as env vars
        for key, value in hyperparams.items():
            env_vars.append(
                client.V1EnvVar(
                    name=f"HYPERPARAM_{key.upper()}",
                    value=str(value)
                )
            )

        # Resource requests and limits
        resources = self._build_resources(spec)

        # Container spec
        container = client.V1Container(
            name='training-worker',
            image=f'training-image:{model}',  # Would be configured
            command=['/bin/bash', '-c'],
            args=[self._build_training_command(spec)],
            env=env_vars,
            resources=resources,
            volume_mounts=self._build_volume_mounts(name, spec),
            # Security context
            security_context=client.V1SecurityContext(
                run_as_non_root=True,
                run_as_user=1000,
                capabilities=client.V1Capabilities(
                    drop=['ALL']
                )
            )
        )

        return container

    def _build_training_command(self, spec: Dict[str, Any]) -> str:
        """Build training command to execute"""

        # This is a template; actual command depends on training framework
        return """
        # Calculate rank from pod index
        export RANK=$(echo $POD_NAME | grep -o '[0-9]*$')

        # Wait for master to be ready
        if [ "$RANK" != "0" ]; then
            echo "Waiting for master node..."
            while ! nc -z $MASTER_ADDR $MASTER_PORT; do sleep 1; done
            sleep 5
        fi

        echo "Starting training as rank $RANK"

        # Run distributed training
        python -m torch.distributed.launch \\
            --nproc_per_node=$GPUS_PER_WORKER \\
            --nnodes=$NUM_WORKERS \\
            --node_rank=$RANK \\
            --master_addr=$MASTER_ADDR \\
            --master_port=$MASTER_PORT \\
            /app/train.py \\
                --model=$MODEL \\
                --dataset=$DATASET \\
                --learning-rate=$HYPERPARAM_LEARNINGRATE \\
                --batch-size=$HYPERPARAM_BATCHSIZE \\
                --epochs=$HYPERPARAM_EPOCHS \\
                --checkpoint-dir=/checkpoints
        """

    def _build_resources(
        self,
        spec: Dict[str, Any]
    ) -> client.V1ResourceRequirements:
        """Build resource requests and limits"""

        gpus = spec.get('gpusPerWorker', 1)
        custom_resources = spec.get('resources', {})

        requests = custom_resources.get('requests', {})
        limits = custom_resources.get('limits', {})

        # Default requests
        if 'cpu' not in requests:
            requests['cpu'] = '4'
        if 'memory' not in requests:
            requests['memory'] = '16Gi'

        # Default limits
        if 'cpu' not in limits:
            limits['cpu'] = '8'
        if 'memory' not in limits:
            limits['memory'] = '32Gi'

        # Add GPU limits
        if gpus > 0:
            limits['nvidia.com/gpu'] = str(gpus)
            requests['nvidia.com/gpu'] = str(gpus)

        return client.V1ResourceRequirements(
            requests=requests,
            limits=limits
        )

    def _build_volumes(
        self,
        name: str,
        spec: Dict[str, Any]
    ) -> List[client.V1Volume]:
        """Build volumes for checkpoints and data"""

        volumes = []

        # Checkpoint volume
        checkpoint_config = spec.get('checkpoint', {})
        if checkpoint_config.get('enabled', True):
            storage_type = checkpoint_config.get('storage', {}).get('type', 'pvc')

            if storage_type == 'pvc':
                volumes.append(
                    client.V1Volume(
                        name='checkpoints',
                        persistent_volume_claim=client.V1PersistentVolumeClaimVolumeSource(
                            claim_name=f"{name}-checkpoints"
                        )
                    )
                )
            elif storage_type == 's3':
                # S3-backed volume (using CSI driver or similar)
                pass

        # Shared memory for distributed training (important for PyTorch)
        volumes.append(
            client.V1Volume(
                name='dshm',
                empty_dir=client.V1EmptyDirVolumeSource(
                    medium='Memory',
                    size_limit='2Gi'
                )
            )
        )

        return volumes

    def _build_volume_mounts(
        self,
        name: str,
        spec: Dict[str, Any]
    ) -> List[client.V1VolumeMount]:
        """Build volume mounts for container"""

        mounts = []

        # Checkpoint mount
        checkpoint_config = spec.get('checkpoint', {})
        if checkpoint_config.get('enabled', True):
            mounts.append(
                client.V1VolumeMount(
                    name='checkpoints',
                    mount_path='/checkpoints'
                )
            )

        # Shared memory mount
        mounts.append(
            client.V1VolumeMount(
                name='dshm',
                mount_path='/dev/shm'
            )
        )

        return mounts

    def _build_node_selector(
        self,
        spec: Dict[str, Any]
    ) -> Dict[str, str]:
        """Build node selector for GPU nodes"""

        gpus = spec.get('gpusPerWorker', 0)

        if gpus > 0:
            return {
                'accelerator': 'nvidia-gpu'  # Adjust for your cluster
            }

        return {}

    def _build_tolerations(
        self,
        spec: Dict[str, Any]
    ) -> List[client.V1Toleration]:
        """Build tolerations for GPU nodes"""

        gpus = spec.get('gpusPerWorker', 0)

        if gpus > 0:
            return [
                client.V1Toleration(
                    key='nvidia.com/gpu',
                    operator='Exists',
                    effect='NoSchedule'
                )
            ]

        return []

    def create_job(
        self,
        name: str,
        namespace: str,
        spec: Dict[str, Any]
    ) -> client.V1Job:
        """Create Job in Kubernetes"""

        job_spec = self.build_job_spec(name, namespace, spec)

        return self.api.create_namespaced_job(
            namespace=namespace,
            body=job_spec
        )

    def delete_job(
        self,
        name: str,
        namespace: str
    ):
        """Delete Job from Kubernetes"""

        job_name = f"{name}-job"

        try:
            self.api.delete_namespaced_job(
                name=job_name,
                namespace=namespace,
                propagation_policy='Background'
            )
        except client.rest.ApiException as e:
            if e.status != 404:
                raise

    def get_job_status(
        self,
        name: str,
        namespace: str
    ) -> Dict[str, Any]:
        """Get Job status"""

        job_name = f"{name}-job"

        try:
            job = self.api.read_namespaced_job_status(
                name=job_name,
                namespace=namespace
            )

            return {
                'active': job.status.active or 0,
                'succeeded': job.status.succeeded or 0,
                'failed': job.status.failed or 0,
                'conditions': [
                    {
                        'type': c.type,
                        'status': c.status,
                        'reason': c.reason,
                        'message': c.message
                    }
                    for c in (job.status.conditions or [])
                ]
            }
        except client.rest.ApiException as e:
            if e.status == 404:
                return None
            raise
```

**Key Concepts:**

1. **Downward API**: Injects pod metadata as environment variables
2. **Distributed Training Env Vars**: `WORLD_SIZE`, `RANK`, `MASTER_ADDR`, `MASTER_PORT`
3. **GPU Resource Limits**: `nvidia.com/gpu` for GPU allocation
4. **Shared Memory**: `/dev/shm` mount for PyTorch DataLoader workers
5. **Node Selectors & Tolerations**: Schedule pods on GPU nodes

### Step 3.2: Service Builder

Create `src/resources/service_builder.py`:

```python
"""
Service builder for distributed training communication.

Creates headless services that provide stable DNS names for worker pods,
enabling distributed training coordination.
"""

from typing import Dict, Any
from kubernetes import client


class ServiceBuilder:
    """Build Kubernetes Service specs for training workloads"""

    def __init__(self):
        self.api = client.CoreV1Api()

    def build_headless_service(
        self,
        name: str,
        namespace: str,
        spec: Dict[str, Any]
    ) -> client.V1Service:
        """
        Build headless service for distributed training.

        Provides stable DNS names:
        - {name}-workers-0.{name}-workers.{namespace}.svc.cluster.local
        - {name}-workers-1.{name}-workers.{namespace}.svc.cluster.local
        - etc.
        """

        master_port = spec.get('networking', {}).get('masterPort', 29500)

        service = client.V1Service(
            api_version='v1',
            kind='Service',
            metadata=client.V1ObjectMeta(
                name=f"{name}-workers",
                namespace=namespace,
                labels={
                    'app': 'trainingjob',
                    'trainingjob': name,
                    'component': 'workers'
                }
            ),
            spec=client.V1ServiceSpec(
                cluster_ip='None',  # Headless
                selector={
                    'app': 'trainingjob',
                    'trainingjob': name,
                    'component': 'worker'
                },
                ports=[
                    client.V1ServicePort(
                        name='master',
                        port=master_port,
                        target_port=master_port,
                        protocol='TCP'
                    )
                ]
            )
        )

        return service

    def create_service(
        self,
        name: str,
        namespace: str,
        spec: Dict[str, Any]
    ) -> client.V1Service:
        """Create service in Kubernetes"""

        service_spec = self.build_headless_service(name, namespace, spec)

        return self.api.create_namespaced_service(
            namespace=namespace,
            body=service_spec
        )
```

### Step 3.3: ConfigMap Builder

Create `src/resources/configmap_builder.py`:

```python
"""
ConfigMap builder for training configuration.

Stores hyperparameters and configuration that can be updated
without rebuilding images.
"""

from typing import Dict, Any
from kubernetes import client
import json


class ConfigMapBuilder:
    """Build Kubernetes ConfigMap specs"""

    def __init__(self):
        self.api = client.CoreV1Api()

    def build_config_map(
        self,
        name: str,
        namespace: str,
        spec: Dict[str, Any]
    ) -> client.V1ConfigMap:
        """Build ConfigMap with training configuration"""

        hyperparams = spec.get('hyperparameters', {})
        networking = spec.get('networking', {})

        config_data = {
            'model': spec.get('model', ''),
            'dataset': spec.get('dataset', ''),
            'framework': spec.get('framework', 'pytorch'),
            'hyperparameters.json': json.dumps(hyperparams, indent=2),
            'backend': networking.get('backend', 'nccl'),
            'master_port': str(networking.get('masterPort', 29500)),
        }

        config_map = client.V1ConfigMap(
            api_version='v1',
            kind='ConfigMap',
            metadata=client.V1ObjectMeta(
                name=f"{name}-config",
                namespace=namespace,
                labels={
                    'app': 'trainingjob',
                    'trainingjob': name
                }
            ),
            data=config_data
        )

        return config_map
```

---

## Phase 4: Controllers Implementation

Controllers contain the business logic for managing training jobs.

### Step 4.1: Job Controller

The JobController coordinates resource creation and deletion. We've already seen parts of this in our source code. Key responsibilities:

- Create Job, Service, ConfigMap
- Monitor Job status
- Handle resource cleanup
- Update metrics

Review `src/controllers/job_controller.py` to understand:

1. **`create_training_resources()`**: Creates all necessary K8s resources
2. **`check_resources_ready()`**: Monitors Job status and updates TrainingJob state
3. **`delete_training_resources()`**: Cleans up resources on deletion

### Step 4.2: Status Controller

The StatusController tracks training progress. Key features:

- Extract metrics from training pods
- Update TrainingJob status field
- Calculate progress percentage
- Track epoch count and timing

**Important pattern**: Status updates use the `/status` subresource to avoid conflicts with spec updates:

```python
# In status_controller.py
def update_status(self, name, namespace, status_patch):
    """Update TrainingJob status using status subresource"""

    api = client.CustomObjectsApi()

    api.patch_namespaced_custom_object_status(
        group='ml.example.com',
        version='v1',
        namespace=namespace,
        plural='trainingjobs',
        name=name,
        body={'status': status_patch}
    )
```

### Step 4.3: Checkpoint Controller

Create `src/controllers/checkpoint_controller.py`:

```python
"""
Checkpoint controller manages training checkpoints.

Handles:
- Creating PVCs for checkpoint storage
- Monitoring checkpoint creation
- Implementing checkpoint recovery
- Managing checkpoint retention
"""

from typing import Dict, Any, Optional
from kubernetes import client
import logging

logger = logging.getLogger(__name__)


class CheckpointController:
    """Manage training checkpoints"""

    def __init__(self):
        self.api = client.CoreV1Api()

    def create_checkpoint_storage(
        self,
        name: str,
        namespace: str,
        spec: Dict[str, Any]
    ) -> Optional[client.V1PersistentVolumeClaim]:
        """
        Create PVC for checkpoint storage.

        Args:
            name: TrainingJob name
            namespace: Kubernetes namespace
            spec: TrainingJob spec

        Returns:
            Created PVC or None if checkpointing disabled
        """

        checkpoint_config = spec.get('checkpoint', {})

        if not checkpoint_config.get('enabled', True):
            return None

        storage_config = checkpoint_config.get('storage', {})
        storage_type = storage_config.get('type', 'pvc')

        if storage_type != 'pvc':
            # S3 or GCS - no PVC needed
            return None

        size = storage_config.get('size', '100Gi')
        storage_class = storage_config.get('storageClass', 'standard')

        pvc = client.V1PersistentVolumeClaim(
            api_version='v1',
            kind='PersistentVolumeClaim',
            metadata=client.V1ObjectMeta(
                name=f"{name}-checkpoints",
                namespace=namespace,
                labels={
                    'app': 'trainingjob',
                    'trainingjob': name,
                    'component': 'checkpoint'
                }
            ),
            spec=client.V1PersistentVolumeClaimSpec(
                access_modes=['ReadWriteMany'],  # Shared across workers
                storage_class_name=storage_class,
                resources=client.V1ResourceRequirements(
                    requests={'storage': size}
                )
            )
        )

        created_pvc = self.api.create_namespaced_persistent_volume_claim(
            namespace=namespace,
            body=pvc
        )

        logger.info(f"Created checkpoint PVC: {created_pvc.metadata.name}")
        return created_pvc

    def get_latest_checkpoint(
        self,
        name: str,
        namespace: str
    ) -> Optional[str]:
        """
        Find latest checkpoint for resuming training.

        This would query the checkpoint storage and return
        the path to the most recent checkpoint.

        Returns:
            Path to latest checkpoint, or None
        """

        # In production, this would:
        # 1. List files in checkpoint directory
        # 2. Parse checkpoint metadata
        # 3. Return path to latest valid checkpoint

        return None

    def cleanup_old_checkpoints(
        self,
        name: str,
        namespace: str,
        keep_last: int = 3
    ):
        """
        Remove old checkpoints to save space.

        Args:
            name: TrainingJob name
            namespace: Namespace
            keep_last: Number of recent checkpoints to keep
        """

        # In production, implement retention policy
        pass
```

---

## Phase 5: Status Management and Monitoring

### Step 5.1: Comprehensive Status Updates

The status field should provide complete visibility into training progress. Extend status updates to include:

```python
# Enhanced status structure
status = {
    'state': 'Running',  # Pending, Initializing, Running, Completed, Failed
    'progress': '45%',
    'currentEpoch': 9,
    'totalEpochs': 20,

    # Worker status
    'workers': {
        'active': 4,
        'succeeded': 0,
        'failed': 0,
        'pending': 0
    },

    # Resource allocation
    'resources': {
        'allocatedGPUs': 8,
        'allocatedNodes': 4,
        'gpuUtilization': 92.5  # Average %
    },

    # Training metrics
    'metrics': {
        'loss': 0.234,
        'accuracy': 0.876,
        'throughput': 1250,  # samples/sec
        'stepTime': 0.8  # seconds
    },

    # Timing
    'startTime': '2025-10-25T10:30:00Z',
    'duration': '2h 15m',
    'estimatedCompletion': '2025-10-25T15:00:00Z',

    # Conditions (Kubernetes standard)
    'conditions': [
        {
            'type': 'ResourcesCreated',
            'status': 'True',
            'reason': 'JobCreated',
            'message': 'Kubernetes Job created successfully',
            'lastTransitionTime': '2025-10-25T10:30:00Z'
        },
        {
            'type': 'Running',
            'status': 'True',
            'reason': 'TrainingStarted',
            'message': 'All workers active',
            'lastTransitionTime': '2025-10-25T10:35:00Z'
        }
    ]
}
```

### Step 5.2: Prometheus Metrics

Create `src/utils/metrics.py`:

```python
"""
Prometheus metrics for operator observability.

Tracks:
- Number of TrainingJobs by state
- Resource allocation (GPUs, workers)
- Training progress and metrics
- Operator performance
"""

from prometheus_client import Counter, Gauge, Histogram, Info
from typing import Optional


# TrainingJob metrics
trainingjobs_created = Counter(
    'trainingjob_created_total',
    'Total TrainingJobs created',
    ['namespace']
)

trainingjobs_completed = Counter(
    'trainingjob_completed_total',
    'Total TrainingJobs completed',
    ['namespace']
)

trainingjobs_failed = Counter(
    'trainingjob_failed_total',
    'Total TrainingJobs failed',
    ['namespace', 'reason']
)

trainingjob_state = Gauge(
    'trainingjob_state',
    'Current state of TrainingJobs',
    ['namespace', 'name', 'state']
)

# Resource metrics
allocated_gpus = Gauge(
    'trainingjob_allocated_gpus',
    'Number of GPUs allocated',
    ['namespace', 'trainingjob']
)

allocated_workers = Gauge(
    'trainingjob_allocated_workers',
    'Number of workers allocated',
    ['namespace', 'trainingjob']
)

gpu_utilization = Gauge(
    'trainingjob_gpu_utilization_percent',
    'GPU utilization percentage',
    ['namespace', 'trainingjob', 'worker']
)

# Training metrics
training_progress = Gauge(
    'trainingjob_progress_percent',
    'Training progress percentage',
    ['namespace', 'trainingjob']
)

training_epoch = Gauge(
    'trainingjob_current_epoch',
    'Current training epoch',
    ['namespace', 'trainingjob']
)

training_loss = Gauge(
    'trainingjob_loss',
    'Current training loss',
    ['namespace', 'trainingjob']
)

training_accuracy = Gauge(
    'trainingjob_accuracy',
    'Current training accuracy',
    ['namespace', 'trainingjob']
)

# Operator metrics
operator_reconcile_duration = Histogram(
    'trainingjob_operator_reconcile_duration_seconds',
    'Time spent in reconciliation loop',
    ['namespace']
)

operator_errors = Counter(
    'trainingjob_operator_errors_total',
    'Total operator errors',
    ['namespace', 'error_type']
)


class OperatorMetrics:
    """Helper class for updating metrics"""

    @staticmethod
    def increment_trainingjobs_created(namespace: str = 'default'):
        trainingjobs_created.labels(namespace=namespace).inc()

    @staticmethod
    def increment_trainingjobs_completed(namespace: str = 'default'):
        trainingjobs_completed.labels(namespace=namespace).inc()

    @staticmethod
    def increment_trainingjobs_failed(
        namespace: str = 'default',
        reason: str = 'Unknown'
    ):
        trainingjobs_failed.labels(
            namespace=namespace,
            reason=reason
        ).inc()

    @staticmethod
    def update_training_progress(
        namespace: str,
        name: str,
        progress: float,
        epoch: int
    ):
        training_progress.labels(
            namespace=namespace,
            trainingjob=name
        ).set(progress)

        training_epoch.labels(
            namespace=namespace,
            trainingjob=name
        ).set(epoch)

    @staticmethod
    def update_training_metrics(
        namespace: str,
        name: str,
        loss: Optional[float] = None,
        accuracy: Optional[float] = None,
        gpu_util: Optional[float] = None
    ):
        if loss is not None:
            training_loss.labels(
                namespace=namespace,
                trainingjob=name
            ).set(loss)

        if accuracy is not None:
            training_accuracy.labels(
                namespace=namespace,
                trainingjob=name
            ).set(accuracy)
```

---

## Phase 6: Checkpoint Management

### Step 6.1: Checkpoint Strategy

Implement automatic checkpointing in the training code:

```python
# In training container (train.py)
import torch
import os
from pathlib import Path

CHECKPOINT_DIR = Path('/checkpoints')
CHECKPOINT_FREQ = int(os.getenv('CHECKPOINT_FREQUENCY', '1'))  # epochs

def save_checkpoint(epoch, model, optimizer, loss):
    """Save training checkpoint"""

    checkpoint_path = CHECKPOINT_DIR / f'checkpoint-epoch-{epoch}.pt'

    torch.save({
        'epoch': epoch,
        'model_state_dict': model.state_dict(),
        'optimizer_state_dict': optimizer.state_dict(),
        'loss': loss,
    }, checkpoint_path)

    # Create 'latest' symlink
    latest_path = CHECKPOINT_DIR / 'checkpoint-latest.pt'
    if latest_path.exists():
        latest_path.unlink()
    latest_path.symlink_to(checkpoint_path)

def load_checkpoint(model, optimizer):
    """Load latest checkpoint if exists"""

    latest_path = CHECKPOINT_DIR / 'checkpoint-latest.pt'

    if not latest_path.exists():
        return 0  # No checkpoint, start from epoch 0

    checkpoint = torch.load(latest_path)
    model.load_state_dict(checkpoint['model_state_dict'])
    optimizer.load_state_dict(checkpoint['optimizer_state_dict'])

    return checkpoint['epoch'] + 1  # Resume from next epoch
```

### Step 6.2: Fault Recovery

When a training job fails and is retried, it should automatically resume from the latest checkpoint:

```python
# In operator's create_handler
def create_handler(...):
    # Check for existing checkpoints
    latest_checkpoint = checkpoint_controller.get_latest_checkpoint(
        name=name,
        namespace=namespace
    )

    # If checkpoint exists, add env var to container
    if latest_checkpoint:
        env_vars.append(
            client.V1EnvVar(
                name='RESUME_FROM_CHECKPOINT',
                value=latest_checkpoint
            )
        )

        logger.info(f"Will resume from checkpoint: {latest_checkpoint}")
```

---

## Phase 7: Fault Tolerance and Retry Logic

### Step 7.1: Automatic Retries

Kubernetes Jobs handle pod-level retries via `backoffLimit`. The operator adds:

1. **Job-level retry**: If entire Job fails, operator can recreate
2. **Checkpoint resume**: Each retry resumes from last checkpoint
3. **Exponential backoff**: Prevent rapid failure loops

```python
@kopf.on.create('ml.example.com', 'v1', 'trainingjobs', retries=3, backoff=1.5)
def create_handler(...):
    """Kopf automatically retries on failures with exponential backoff"""
    ...
```

### Step 7.2: Preemption Handling

For spot/preemptible instances:

```python
# Detect preemption
def is_preempted(pod_status) -> bool:
    """Check if pod was preempted"""

    for condition in pod_status.conditions or []:
        if condition.reason == 'Evicted' or condition.reason == 'Preempted':
            return True

    return False

# In reconcile loop
if is_preempted(pod_status):
    logger.info("Pod preempted, will restart with checkpoint resume")
    # Kubernetes Job will automatically restart the pod
```

---

## Phase 8: Production Deployment

### Step 8.1: RBAC Configuration

Create `kubernetes/base/rbac.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: trainingjob-operator
  namespace: trainingjob-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: trainingjob-operator
rules:
  # TrainingJob CRD
  - apiGroups: ["ml.example.com"]
    resources: ["trainingjobs"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: ["ml.example.com"]
    resources: ["trainingjobs/status"]
    verbs: ["get", "update", "patch"]

  # Kubernetes resources
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["services", "configmaps", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]

  # Events
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: trainingjob-operator
subjects:
  - kind: ServiceAccount
    name: trainingjob-operator
    namespace: trainingjob-system
roleRef:
  kind: ClusterRole
  name: trainingjob-operator
  apiGroup: rbac.authorization.k8s.io
```

### Step 8.2: Operator Deployment

Create `kubernetes/base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trainingjob-operator
  namespace: trainingjob-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trainingjob-operator
  template:
    metadata:
      labels:
        app: trainingjob-operator
    spec:
      serviceAccountName: trainingjob-operator
      containers:
        - name: operator
          image: trainingjob-operator:latest
          imagePullPolicy: Always
          env:
            - name: OPERATOR_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          ports:
            - name: metrics
              containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Step 8.3: Build Container Image

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code
COPY src/ ./src/

# Run as non-root
RUN useradd -m -u 1000 operator && chown -R operator:operator /app
USER operator

# Run operator
CMD ["kopf", "run", "--standalone", "/app/src/operator/main.py"]
```

Build and push:

```bash
docker build -t your-registry/trainingjob-operator:v1.0.0 .
docker push your-registry/trainingjob-operator:v1.0.0
```

---

## Phase 9: Testing Strategy

### Step 9.1: Unit Tests

Create `tests/unit/test_job_builder.py`:

```python
"""Unit tests for JobBuilder"""

import pytest
from src.resources.job_builder import JobBuilder


def test_build_job_spec():
    """Test Job spec generation"""

    builder = JobBuilder()

    spec = {
        'model': 'resnet50',
        'dataset': 'imagenet',
        'numWorkers': 4,
        'gpusPerWorker': 2,
        'hyperparameters': {
            'learningRate': 0.001,
            'batchSize': 32,
            'epochs': 10
        }
    }

    job = builder.build_job_spec(
        name='test-job',
        namespace='default',
        spec=spec
    )

    # Verify Job structure
    assert job.spec.parallelism == 4
    assert job.spec.completions == 4

    # Verify GPU allocation
    container = job.spec.template.spec.containers[0]
    assert container.resources.limits['nvidia.com/gpu'] == '2'


def test_distributed_training_env_vars():
    """Test distributed training environment variables"""

    builder = JobBuilder()

    spec = {
        'model': 'bert',
        'dataset': 'wikitext',
        'numWorkers': 8,
        'gpusPerWorker': 1
    }

    job = builder.build_job_spec('test', 'default', spec)
    container = job.spec.template.spec.containers[0]

    env_dict = {e.name: e.value for e in container.env if e.value}

    assert env_dict['WORLD_SIZE'] == '8'
    assert env_dict['MASTER_ADDR'] == 'test-workers-0.test-workers'
    assert env_dict['MASTER_PORT'] == '29500'
```

### Step 9.2: Integration Tests

Create `tests/integration/test_operator.py`:

```python
"""Integration tests for operator"""

import pytest
from kubernetes import client, config
import time


@pytest.fixture(scope='module')
def k8s_client():
    """Load Kubernetes config"""
    config.load_kube_config()
    return client.CustomObjectsApi()


def test_create_trainingjob(k8s_client):
    """Test TrainingJob creation"""

    trainingjob = {
        'apiVersion': 'ml.example.com/v1',
        'kind': 'TrainingJob',
        'metadata': {
            'name': 'test-integration',
            'namespace': 'default'
        },
        'spec': {
            'model': 'resnet18',
            'dataset': 'cifar10',
            'numWorkers': 2,
            'gpusPerWorker': 1,
            'hyperparameters': {
                'learningRate': 0.01,
                'batchSize': 64,
                'epochs': 5
            }
        }
    }

    # Create TrainingJob
    k8s_client.create_namespaced_custom_object(
        group='ml.example.com',
        version='v1',
        namespace='default',
        plural='trainingjobs',
        body=trainingjob
    )

    # Wait for resources to be created
    time.sleep(10)

    # Verify Job created
    batch_api = client.BatchV1Api()
    job = batch_api.read_namespaced_job(
        name='test-integration-job',
        namespace='default'
    )

    assert job is not None
    assert job.spec.parallelism == 2

    # Cleanup
    k8s_client.delete_namespaced_custom_object(
        group='ml.example.com',
        version='v1',
        namespace='default',
        plural='trainingjobs',
        name='test-integration'
    )
```

### Step 9.3: End-to-End Tests

```python
"""E2E test: Full training workflow"""

def test_full_training_workflow(k8s_client):
    """Test complete training lifecycle"""

    # 1. Create TrainingJob
    # 2. Wait for state: Pending → Initializing → Running
    # 3. Monitor progress
    # 4. Wait for completion
    # 5. Verify checkpoints created
    # 6. Cleanup

    # Implementation would use real training images
    pass
```

---

## Phase 10: Monitoring and Observability

### Step 10.1: Prometheus Integration

Create `kubernetes/overlays/with-monitoring/servicemonitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: trainingjob-operator
  namespace: trainingjob-system
spec:
  selector:
    matchLabels:
      app: trainingjob-operator
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### Step 10.2: Grafana Dashboard

Create a Grafana dashboard with panels:

1. **TrainingJobs by State** (pie chart)
2. **Active Training Jobs** (time series)
3. **GPU Utilization** (gauge per job)
4. **Training Progress** (bar chart)
5. **Job Duration** (histogram)
6. **Failure Rate** (percentage)

Dashboard JSON:

```json
{
  "dashboard": {
    "title": "TrainingJob Operator",
    "panels": [
      {
        "title": "Active Training Jobs",
        "targets": [{
          "expr": "sum(trainingjob_state{state='Running'})"
        }]
      },
      {
        "title": "GPU Utilization",
        "targets": [{
          "expr": "avg(trainingjob_gpu_utilization_percent) by (trainingjob)"
        }]
      }
    ]
  }
}
```

### Step 10.3: Logging

Use structured logging throughout:

```python
logger.info(
    "Training progress updated",
    extra={
        'trainingjob': name,
        'namespace': namespace,
        'epoch': current_epoch,
        'progress': progress_percent,
        'loss': current_loss
    }
)
```

---

## Troubleshooting Guide

### Common Issues

**Issue 1: Pods stuck in Pending**

```bash
# Check pod events
kubectl describe pod <pod-name>

# Common causes:
# - Insufficient GPU nodes
# - Resource limits too high
# - Node selector doesn't match

# Solution: Adjust resource requests or add GPU nodes
```

**Issue 2: Training not starting**

```bash
# Check Job status
kubectl get job <name>-job -o yaml

# Check worker pod logs
kubectl logs <name>-job-<hash> -f

# Common causes:
# - MASTER_ADDR unreachable
# - Port conflicts
# - Missing dependencies in image

# Solution: Verify networking, check Service DNS
```

**Issue 3: Checkpoints not saving**

```bash
# Check PVC status
kubectl get pvc <name>-checkpoints

# Check mount in pod
kubectl exec <pod> -- ls -la /checkpoints

# Common causes:
# - PVC not bound
# - Permissions issues
# - Disk full

# Solution: Check PV provisioner, verify access modes
```

**Issue 4: Operator not reconciling**

```bash
# Check operator logs
kubectl logs -n trainingjob-system deployment/trainingjob-operator

# Common causes:
# - RBAC permissions missing
# - API errors
# - Network issues

# Solution: Verify RBAC, check API server connectivity
```

---

## Best Practices

### 1. Resource Limits

Always set resource limits to prevent noisy neighbors:

```yaml
spec:
  resources:
    requests:
      cpu: "4"
      memory: "16Gi"
      nvidia.com/gpu: "1"
    limits:
      cpu: "8"
      memory: "32Gi"
      nvidia.com/gpu: "1"
```

### 2. Checkpoint Frequency

Balance checkpoint frequency vs training speed:

- **Too frequent**: Slows training, wastes I/O
- **Too infrequent**: Lose more progress on failure
- **Recommendation**: Every 1-5 epochs, or every 1-2 hours

### 3. GPU Utilization

Monitor GPU utilization and adjust:

- **Low utilization**: Increase batch size or model size
- **OOM errors**: Decrease batch size or use gradient accumulation
- **Target**: 80-95% GPU utilization

### 4. Distributed Training

For multi-node training:

- Use NCCL backend for GPU training
- Ensure high-bandwidth node-to-node networking
- Place workers in same availability zone
- Use topology-aware scheduling

### 5. Security

- Run containers as non-root user
- Drop all capabilities
- Use read-only root filesystem where possible
- Store secrets in Kubernetes Secrets, not ConfigMaps
- Enable pod security policies/standards

---

## Advanced Topics

### 1. Multi-Tenancy

Support multiple teams with quotas:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: trainingjob-quota
  namespace: team-a
spec:
  hard:
    requests.nvidia.com/gpu: "16"
    count/trainingjobs.ml.example.com: "5"
```

### 2. Auto-Scaling

Implement horizontal pod autoscaling based on queue depth:

- Monitor job queue length
- Scale worker pods dynamically
- Use cluster autoscaler for node-level scaling

### 3. Cost Optimization

- Use spot/preemptible instances for fault-tolerant workloads
- Implement pod priority and preemption
- Auto-shutdown idle resources
- Monitor cost per training job

### 4. MLOps Integration

Integrate with:

- **MLflow**: Track experiments and metrics
- **Airflow**: Schedule periodic retraining
- **ArgoCD**: GitOps for operator deployment
- **Kubeflow**: Broader ML pipeline orchestration

---

## Conclusion

You've built a production-ready Kubernetes operator for ML training! Key achievements:

✅ Custom Resource Definition for TrainingJob
✅ Kopf-based operator with event handlers
✅ Automated distributed training orchestration
✅ GPU resource management
✅ Checkpoint-based fault tolerance
✅ Comprehensive monitoring and observability
✅ Production deployment with RBAC
✅ Testing at multiple levels

### Next Steps

1. **Extend functionality**:
   - Support TensorFlow, JAX, other frameworks
   - Add gang scheduling for guaranteed resources
   - Implement training job queues and priorities
   - Add hyperparameter tuning support

2. **Production hardening**:
   - Load testing with high job concurrency
   - Chaos engineering for fault injection
   - Security scanning and penetration testing
   - Disaster recovery procedures

3. **Integration**:
   - Connect to data pipelines
   - Integrate with model registry
   - Add deployment automation post-training
   - Build UI for job management

### Resources

- [Kopf Documentation](https://kopf.readthedocs.io/)
- [Kubernetes Operator Best Practices](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [PyTorch Distributed Training](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html)
- [Kubeflow Training Operators](https://www.kubeflow.org/docs/components/training/)

**Congratulations on completing this advanced project!** You now have deep expertise in Kubernetes operators, distributed systems, and ML infrastructure automation.

---

**Guide Version**: 1.0
**Last Updated**: October 25, 2025
**Estimated Time**: 40-60 hours
**Difficulty**: ⭐⭐⭐⭐⭐ (Expert)