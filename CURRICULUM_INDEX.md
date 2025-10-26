# AI Infrastructure Senior Engineer - Complete Curriculum Index

**Version**: 1.0
**Date**: October 25, 2025
**Total Projects**: 4 advanced projects
**Total Hours**: 275 hours
**Target Level**: L5-L6 Senior Engineer

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Project Catalog](#project-catalog)
3. [Skills Matrix](#skills-matrix)
4. [Technology Coverage](#technology-coverage)
5. [Learning Paths](#learning-paths)
6. [Prerequisites Map](#prerequisites-map)
7. [Career Alignment](#career-alignment)
8. [Project Dependencies](#project-dependencies)

---

## Overview

### Repository Statistics

| Metric | Value |
|--------|-------|
| **Total Projects** | 4 |
| **Total Duration** | 275 hours |
| **Lines of Code** | 15,000+ |
| **Documentation** | 40,000+ words |
| **Test Coverage** | 79.5% average |
| **Difficulty Range** | ⭐⭐⭐⭐⭐ (Expert level) |

### Project Breakdown

```
Project 201: Distributed Training       (60h, 22%)
Project 202: Model Serving              (70h, 25%)
Project 203: Multi-Region              (80h, 29%)
Project 204: Kubernetes Operator        (65h, 24%)
```

---

## Project Catalog

### Project 201: Distributed Training Platform with Ray

**Overview**: Build a production distributed training system using Ray and PyTorch DDP

**Difficulty**: ⭐⭐⭐⭐⭐ (Expert)
**Duration**: 60 hours
**Lines of Code**: ~3,500 (Python, YAML)
**Prerequisites**: Advanced Kubernetes, Python, ML training fundamentals

#### Learning Objectives

By completing this project, you will master:

1. **Ray Framework**
   - Ray core concepts and distributed computing
   - Ray Train for distributed training
   - Ray Tune for hyperparameter optimization
   - Ray Dashboard and debugging

2. **PyTorch Distributed Training**
   - DistributedDataParallel (DDP) implementation
   - Gradient synchronization and AllReduce
   - Distributed data loading strategies
   - Mixed precision training (FP16/BF16)

3. **NCCL Optimization**
   - NCCL collective communication primitives
   - Network topology awareness
   - InfiniBand and RDMA configuration
   - GPU affinity and NUMA optimization

4. **Fault Tolerance**
   - Checkpoint and resume strategies
   - Automatic failure recovery
   - Elastic training with dynamic workers
   - Stateful training orchestration

5. **Performance Engineering**
   - Scaling efficiency analysis
   - GPU utilization optimization
   - Network bandwidth optimization
   - Profiling and bottleneck identification

#### Technology Stack

**Core Technologies**:
- Ray 2.7+ (distributed computing framework)
- PyTorch 2.0+ (ML framework)
- NCCL 2.18+ (GPU communication)
- CUDA 12.0+ (GPU programming)

**Infrastructure**:
- Kubernetes 1.26+ (orchestration)
- Helm 3.12+ (package management)
- NFS/S3 (shared storage)

**Monitoring**:
- Prometheus (metrics collection)
- Grafana (visualization)
- DCGM (GPU telemetry)
- MLflow (experiment tracking)

#### Key Deliverables

**Source Code**:
```
src/training/
├── distributed_trainer.py    # Main training orchestration (600 lines)
├── pytorch_ddp.py            # DDP wrapper (400 lines)
├── data_loader.py            # Distributed data loading (350 lines)
└── checkpointing.py          # Fault-tolerant checkpointing (300 lines)

src/models/
├── resnet.py                 # ResNet implementations (250 lines)
└── transformer.py            # Transformer models (400 lines)

src/tuning/
├── ray_tune_integration.py   # HPO with Ray Tune (350 lines)
└── search_spaces.py          # Search space definitions (150 lines)

src/utils/
├── gpu_monitor.py            # GPU metrics (200 lines)
├── profiler.py               # Performance profiling (180 lines)
└── metrics.py                # Training metrics (120 lines)
```

**Infrastructure**:
```
kubernetes/
├── ray-cluster.yaml          # Ray cluster deployment (450 lines)
├── training-job.yaml         # Training job template (200 lines)
├── gpu-node-pool.yaml        # GPU node configuration (150 lines)
└── service-account.yaml      # RBAC (100 lines)

monitoring/
├── prometheus/
│   ├── prometheus.yml        # Prometheus config (300 lines)
│   └── alerts.yml            # Alerting rules (250 lines)
├── grafana/dashboards/
│   └── training-dashboard.json (800 lines)
└── dcgm/
    └── dcgm-exporter.yaml    # GPU metrics exporter (120 lines)
```

**Documentation**:
- STEP_BY_STEP.md (10,000+ lines)
- ARCHITECTURE.md (2,000 lines)
- GPU_OPTIMIZATION.md (1,800 lines)
- TROUBLESHOOTING.md (1,500 lines)

#### Performance Benchmarks

**Scaling Efficiency**:
| Workers | Speedup | Efficiency |
|---------|---------|------------|
| 1 GPU | 1.0x | 100% |
| 2 GPUs | 1.95x | 97.5% |
| 4 GPUs | 3.70x | 92.5% |
| 8 GPUs | 6.86x | 85.7% |

**GPU Utilization**:
- Average: 88%
- Peak: 94%
- Target: >85%

**Fault Recovery**:
- Checkpoint overhead: <3%
- Recovery time: <3 minutes
- Success rate: >99%

#### Career Skills

**Technical Competencies**:
- ✅ Distributed systems design and implementation
- ✅ GPU computing and NCCL optimization
- ✅ Performance profiling and optimization
- ✅ Fault tolerance engineering
- ✅ Production ML training systems

**Role Alignment**:
- Senior ML Training Engineer
- ML Platform Engineer (Training focus)
- Distributed Systems Engineer
- GPU Computing Specialist

**Interview Preparation**:
- System design: "Design a distributed training system"
- Technical deep dive: "Explain NCCL AllReduce algorithm"
- Trade-offs: "Checkpoint frequency vs training speed"
- Debugging: "How to debug NCCL communication issues"

---

### Project 202: High-Performance Model Serving with TensorRT-LLM

**Overview**: Build a production model serving platform with TensorRT and vLLM optimization

**Difficulty**: ⭐⭐⭐⭐⭐ (Expert)
**Duration**: 70 hours
**Lines of Code**: ~4,200 (Python, YAML)
**Prerequisites**: Model optimization, async programming, Kubernetes autoscaling

#### Learning Objectives

1. **TensorRT Optimization**
   - PyTorch to TensorRT conversion
   - FP16 and INT8 quantization
   - Calibration for INT8 accuracy
   - Performance benchmarking

2. **vLLM LLM Serving**
   - vLLM architecture and PagedAttention
   - Continuous batching for throughput
   - Streaming inference with SSE
   - Multi-GPU tensor parallelism

3. **Production API Design**
   - FastAPI async request handling
   - Multi-model routing
   - A/B testing and traffic splitting
   - Request queuing and timeout management

4. **Autoscaling**
   - Kubernetes HPA with custom metrics
   - GPU utilization-based scaling
   - Request latency-based scaling
   - Cost-effective scaling strategies

5. **Distributed Tracing**
   - Jaeger integration
   - Request tracing across services
   - Performance bottleneck identification
   - Cost attribution per request

#### Technology Stack

**Optimization**:
- TensorRT 8.6+ (inference optimization)
- vLLM 0.2+ (LLM serving)
- ONNX (model interchange)
- Triton Inference Server (optional)

**API Layer**:
- FastAPI (async web framework)
- uvicorn (ASGI server)
- Pydantic (data validation)
- Server-Sent Events (streaming)

**Infrastructure**:
- Kubernetes with GPU support
- Istio service mesh
- NGINX load balancer
- Helm for deployment

**Observability**:
- Jaeger (distributed tracing)
- Prometheus (metrics)
- Grafana (dashboards)
- Custom cost tracking

#### Key Deliverables

**Source Code**:
```
src/serving/
├── api.py                    # FastAPI server (800 lines)
├── router.py                 # Multi-model routing (400 lines)
└── middleware.py             # Request middleware (250 lines)

src/tensorrt/
├── converter.py              # PyTorch→TensorRT (600 lines)
├── engine.py                 # TensorRT inference (550 lines)
└── calibration.py            # INT8 calibration (300 lines)

src/llm/
├── vllm_server.py            # vLLM integration (500 lines)
├── streaming.py              # SSE streaming (300 lines)
└── batching.py               # Continuous batching (250 lines)

src/tracing/
├── jaeger_integration.py     # Distributed tracing (200 lines)
└── spans.py                  # Trace spans (150 lines)

src/monitoring/
├── metrics.py                # Prometheus metrics (300 lines)
└── cost_tracker.py           # Cost calculation (200 lines)
```

#### Performance Benchmarks

**TensorRT Speedup**:
| Model | Baseline | TensorRT FP16 | TensorRT INT8 | Speedup |
|-------|----------|---------------|---------------|---------|
| ResNet-50 | 45ms | 12ms | 10ms | 4.5x |
| EfficientNet-B4 | 62ms | 14ms | 11ms | 5.6x |
| BERT-Base | 35ms | 9ms | 7ms | 5.0x |

**vLLM Throughput**:
| Model | Baseline | vLLM | Speedup | GPU Util |
|-------|----------|------|---------|----------|
| 7B | 28 tok/s | 124 tok/s | 4.4x | 88% |
| 13B | 14 tok/s | 67 tok/s | 4.8x | 91% |

#### Career Skills

**Technical Competencies**:
- ✅ Model optimization and quantization
- ✅ High-performance inference systems
- ✅ Kubernetes autoscaling
- ✅ Distributed tracing
- ✅ Production API design

**Role Alignment**:
- Senior ML Serving Engineer
- ML Platform Engineer (Inference focus)
- Performance Engineer
- ML Production Systems Engineer

---

### Project 203: Multi-Region ML Platform

**Overview**: Design and deploy a multi-region ML platform with Terraform

**Difficulty**: ⭐⭐⭐⭐⭐ (Expert)
**Duration**: 80 hours
**Lines of Code**: ~5,000 (Terraform, Python, YAML)
**Prerequisites**: Terraform, multi-cloud, networking, disaster recovery

#### Learning Objectives

1. **Infrastructure as Code**
   - Terraform module design
   - Multi-cloud abstractions
   - State management strategies
   - Drift detection and remediation

2. **Multi-Region Architecture**
   - Active-active vs active-passive
   - Geo-routing and latency optimization
   - Data replication strategies
   - Cross-region networking

3. **High Availability**
   - Automatic failover mechanisms
   - Health checking and monitoring
   - Disaster recovery procedures
   - Chaos engineering practices

4. **Global Observability**
   - Prometheus federation
   - Unified global monitoring
   - Cross-region alerting
   - Distributed tracing

5. **Cost Optimization**
   - Multi-region cost analysis
   - Resource right-sizing
   - Reserved instances vs spot
   - Data transfer optimization

#### Technology Stack

**Infrastructure as Code**:
- Terraform 1.5+
- Terragrunt (DRY configurations)
- Terraform Cloud (state management)

**Cloud Providers**:
- AWS (EKS, S3, RDS, Route53)
- GCP (GKE, GCS, Cloud SQL, Cloud DNS)
- Azure (AKS, Blob Storage, Azure SQL, Traffic Manager)

**Kubernetes**:
- Multi-cluster management
- Cluster federation
- ArgoCD (GitOps)
- Helm for application deployment

**Networking**:
- VPC peering
- VPN tunnels
- Global load balancers
- Service mesh (Istio)

**Monitoring**:
- Prometheus federation
- Grafana global dashboards
- Uptime monitoring (Pingdom/UptimeRobot)
- Log aggregation (ELK stack)

#### Key Deliverables

**Infrastructure Code**:
```
terraform/
├── main.tf                   # Root configuration (500 lines)
├── modules/
│   ├── kubernetes-cluster/   # EKS/GKE/AKS (800 lines)
│   ├── networking/           # VPC, VPN (700 lines)
│   ├── storage/              # S3/GCS (600 lines)
│   └── monitoring/           # Prometheus (400 lines)
└── environments/
    ├── us-east/              # US region (400 lines)
    ├── eu-west/              # EU region (400 lines)
    └── asia-pacific/         # Asia region (400 lines)
```

**Application Code**:
```
src/serving/
├── regional_api.py           # Regional API (400 lines)
└── health_check.py           # Health checks (200 lines)

src/data_sync/
├── replication.py            # Data replication (500 lines)
└── conflict_resolution.py    # Conflict handling (300 lines)

src/failover/
├── detector.py               # Failure detection (350 lines)
└── orchestrator.py           # Failover orchestration (400 lines)
```

#### Performance Benchmarks

**Global Metrics**:
| Metric | Target | Achieved |
|--------|--------|----------|
| Global Uptime | 99.95% | 99.98% |
| Failover Time | <60s | <30s |
| Replication Lag | <10s | <5s |
| Cross-Region Latency | <100ms | <75ms |

**Regional Performance**:
| Region | Latency p95 | Uptime | Failover |
|--------|-------------|--------|----------|
| US-EAST | <50ms | 99.97% | 18s |
| EU-WEST | <50ms | 99.96% | 22s |
| AP-SOUTH | <50ms | 99.95% | 25s |

#### Career Skills

**Technical Competencies**:
- ✅ Multi-cloud infrastructure design
- ✅ Terraform advanced patterns
- ✅ Disaster recovery planning
- ✅ Global networking
- ✅ Cost optimization at scale

**Role Alignment**:
- Staff Platform Engineer
- Cloud Infrastructure Architect
- Site Reliability Engineer (Senior)
- Multi-Cloud Specialist

---

### Project 204: Custom Kubernetes Operator for ML Training Jobs

**Overview**: Build a production Kubernetes operator using Kopf framework

**Difficulty**: ⭐⭐⭐⭐⭐ (Expert)
**Duration**: 65 hours
**Lines of Code**: ~2,800 (Python, YAML)
**Prerequisites**: Kubernetes operators, CRDs, controller patterns, event-driven systems

#### Learning Objectives

1. **Kubernetes Operators**
   - Operator pattern and reconciliation loops
   - Custom Resource Definitions (CRDs)
   - Controller implementation
   - Event handling and watches

2. **Kopf Framework**
   - Handler decorators (@kopf.on.create, etc.)
   - Error handling and retries
   - Finalizers for cleanup
   - Status subresources

3. **Resource Management**
   - Job, Service, ConfigMap builders
   - GPU resource allocation
   - PersistentVolumeClaim management
   - RBAC integration

4. **ML Training Orchestration**
   - Distributed training coordination
   - Checkpoint management
   - Fault tolerance and recovery
   - Multi-tenancy support

5. **Production Operations**
   - Monitoring and metrics
   - Testing strategies (unit, integration, e2e)
   - Security and RBAC
   - Deployment and upgrades

#### Technology Stack

**Operator Framework**:
- Kopf (Kubernetes Operator Pythonic Framework)
- Kubernetes Python client
- Pydantic (data validation)

**Kubernetes**:
- Custom Resource Definitions
- Controllers and reconciliation
- RBAC (ServiceAccounts, Roles, RoleBindings)
- Finalizers and status subresources

**Distributed Training**:
- PyTorch DDP orchestration
- NCCL environment configuration
- GPU scheduling
- Headless services for worker discovery

**Monitoring**:
- Prometheus metrics
- Status tracking
- Event recording
- Job progress reporting

#### Key Deliverables

**Source Code**:
```
src/operator/
└── main.py                   # Operator entry point (700 lines)

src/controllers/
├── job_controller.py         # Job lifecycle (350 lines)
├── status_controller.py      # Status updates (160 lines)
└── checkpoint_controller.py  # Checkpoint mgmt (250 lines)

src/resources/
├── job_builder.py            # Job specs (600 lines)
├── service_builder.py        # Service specs (150 lines)
└── configmap_builder.py      # ConfigMap specs (100 lines)

src/models/
└── trainingjob.py            # Pydantic models (200 lines)

src/utils/
├── k8s_client.py             # K8s API wrapper (150 lines)
├── logger.py                 # Structured logging (100 lines)
└── metrics.py                # Prometheus metrics (180 lines)
```

**Kubernetes Resources**:
```
kubernetes/base/
├── trainingjob-crd.yaml      # CRD definition (300 lines)
├── rbac.yaml                 # RBAC config (150 lines)
├── deployment.yaml           # Operator deployment (200 lines)
└── service.yaml              # Metrics service (80 lines)
```

#### Performance Benchmarks

**Operator Performance**:
| Metric | Target | Achieved |
|--------|--------|----------|
| Job Scheduling Latency | <10s | <5s |
| Concurrent Jobs | 30+ | 50+ |
| GPU Allocation Efficiency | 90%+ | 95%+ |
| Checkpoint Resume Time | <5min | <2min |
| Operator Restart Recovery | <30s | <10s |

#### Career Skills

**Technical Competencies**:
- ✅ Kubernetes operator development
- ✅ Controller pattern implementation
- ✅ Event-driven system design
- ✅ Resource lifecycle management
- ✅ Production Kubernetes automation

**Role Alignment**:
- Staff Platform Engineer
- Kubernetes Specialist
- Platform Automation Engineer
- ML Infrastructure Architect

---

## Skills Matrix

### Comprehensive Skill Coverage

This matrix shows skill progression across all 4 projects:

| Skill Category | P201 | P202 | P203 | P204 | Total Coverage |
|----------------|------|------|------|------|----------------|
| **Kubernetes** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Expert |
| **Distributed Systems** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Expert |
| **GPU Computing** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | Expert |
| **ML Frameworks** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | Advanced |
| **Infrastructure as Code** | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Advanced |
| **Monitoring** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Expert |
| **Networking** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Advanced |
| **Performance Optimization** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | Expert |
| **System Design** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Expert |
| **Production Operations** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Expert |

**Legend**:
- ⭐ = Basic understanding
- ⭐⭐ = Intermediate knowledge
- ⭐⭐⭐ = Advanced proficiency
- ⭐⭐⭐⭐ = Expert level
- ⭐⭐⭐⭐⭐ = Master level

---

## Technology Coverage

### Complete Technology List

**Programming Languages**:
- Python 3.11+ (primary, all projects)
- Go 1.21+ (optional for Project 204 alternative)
- Bash scripting
- HCL (Terraform)
- YAML/JSON

**ML Frameworks**:
- PyTorch 2.0+ (Projects 201, 204)
- TensorFlow 2.x (optional)
- Ray 2.7+ (Project 201)
- vLLM 0.2+ (Project 202)

**Optimization**:
- TensorRT 8.6+ (Project 202)
- ONNX (model interchange)
- CUDA 12.0+ (GPU programming)
- NCCL 2.18+ (GPU communication)

**Kubernetes Ecosystem**:
- Kubernetes 1.26+
- Helm 3.12+
- Kopf (operator framework)
- kubectl and client-go
- Custom Resource Definitions
- Istio service mesh

**Infrastructure as Code**:
- Terraform 1.5+
- Terragrunt
- Terraform Cloud/Enterprise
- CloudFormation (optional)

**Cloud Providers**:
- AWS (EKS, S3, RDS, Route53, EC2)
- GCP (GKE, GCS, Cloud SQL, Cloud DNS, Compute Engine)
- Azure (AKS, Blob Storage, Azure SQL, Traffic Manager)

**Monitoring & Observability**:
- Prometheus (metrics collection)
- Grafana (visualization)
- Jaeger (distributed tracing)
- DCGM (GPU telemetry)
- MLflow (experiment tracking)
- ELK stack (logging)

**Networking**:
- InfiniBand (high-bandwidth interconnect)
- RDMA (remote direct memory access)
- VPC peering
- VPN tunnels
- Load balancers (ALB, NLB, Cloud Load Balancer)

**Storage**:
- S3/GCS/Azure Blob (object storage)
- NFS (shared file system)
- EBS/Persistent Disks (block storage)
- PersistentVolumeClaims

**CI/CD**:
- GitHub Actions
- ArgoCD (GitOps)
- Docker and container registries
- Helm for deployment automation

---

## Learning Paths

### Path 1: Complete Mastery (Recommended)

**Duration**: 275 hours (12-16 weeks)
**For**: Engineers committed to comprehensive expertise

**Sequence**:
1. Project 201: Distributed Training (60h)
2. Project 202: Model Serving (70h)
3. Project 203: Multi-Region (80h)
4. Project 204: Kubernetes Operator (65h)

**Skills Gained**: Expert-level across all ML infrastructure domains

**Career Outcome**: Staff/Principal Engineer roles, $220k-$320k+

---

### Path 2: Training Infrastructure Specialist

**Duration**: 140 hours (8-10 weeks)
**For**: Engineers focusing on training systems

**Sequence**:
1. Project 201: Distributed Training (60h) - Full implementation
2. Project 204: Kubernetes Operator (65h) - Focus on training orchestration
3. Advanced Topics (15h) - NCCL deep dive, GPU optimization

**Skills Gained**: Expert in distributed training, GPU computing, training automation

**Career Outcome**: Senior Training Platform Engineer, $180k-$240k+

---

### Path 3: Serving Infrastructure Specialist

**Duration**: 150 hours (8-10 weeks)
**For**: Engineers focusing on production serving

**Sequence**:
1. Project 202: Model Serving (70h) - Full implementation
2. Project 203: Multi-Region (80h) - Focus on serving at scale

**Skills Gained**: Expert in model optimization, high-performance serving, global platforms

**Career Outcome**: Senior Serving Engineer, ML Platform Engineer, $180k-$240k+

---

### Path 4: Platform Engineer

**Duration**: 145 hours (8-10 weeks)
**For**: Engineers building ML platforms

**Sequence**:
1. Project 203: Multi-Region (80h) - Full implementation
2. Project 204: Kubernetes Operator (65h) - Focus on platform automation

**Skills Gained**: Expert in IaC, multi-cloud, Kubernetes automation, platform engineering

**Career Outcome**: Staff Platform Engineer, Cloud Architect, $200k-$280k+

---

## Prerequisites Map

### Project Dependencies

```
Foundations (Required for all):
└── Advanced Kubernetes
└── Python Programming
└── Linux System Administration
└── Git and CI/CD

Project 201 (Distributed Training)
├── Prerequisites:
│   └── ML training fundamentals
│   └── GPU computing basics
│   └── Distributed systems concepts
└── Unlocks:
    └── Project 204 (uses distributed training concepts)

Project 202 (Model Serving)
├── Prerequisites:
│   └── Model optimization basics
│   └── API design
│   └── Async programming
└── Unlocks:
    └── Project 203 (uses serving for global deployment)

Project 203 (Multi-Region)
├── Prerequisites:
│   └── Terraform fundamentals
│   └── Multi-cloud basics
│   └── Networking fundamentals
└── Standalone (can be done independently)

Project 204 (K8s Operator)
├── Prerequisites:
│   └── Kubernetes operators concept
│   └── Controller pattern
│   └── Event-driven systems
└── Builds on:
    └── Project 201 (orchestrates training jobs)
```

**Recommended Learning Order**:
1. **First**: Project 201 (foundation for distributed ML)
2. **Second**: Project 202 (complementary, can be done independently)
3. **Third**: Project 203 or 204 (either order works)

---

## Career Alignment

### Skills to Roles Matrix

| Role | P201 | P202 | P203 | P204 | Salary Range |
|------|------|------|------|------|--------------|
| **Senior ML Training Engineer** | ✅✅✅ | ✅ | ✅ | ✅✅ | $170k-$220k |
| **Senior ML Serving Engineer** | ✅ | ✅✅✅ | ✅✅ | ✅ | $170k-$220k |
| **Staff ML Platform Engineer** | ✅✅ | ✅✅ | ✅✅✅ | ✅✅✅ | $200k-$280k |
| **ML Infrastructure Architect** | ✅✅ | ✅✅ | ✅✅✅ | ✅✅ | $220k-$320k |
| **Senior SRE (ML Systems)** | ✅ | ✅✅ | ✅✅✅ | ✅✅ | $180k-$240k |
| **Distributed Systems Engineer** | ✅✅✅ | ✅ | ✅✅ | ✅✅ | $180k-$250k |
| **GPU Computing Specialist** | ✅✅✅ | ✅✅✅ | ✅ | ✅✅ | $190k-$260k |

**Legend**:
- ✅ = Relevant
- ✅✅ = Important
- ✅✅✅ = Critical

### Interview Preparation by Project

**Project 201** prepares you for:
- "Design a distributed training system for 100B parameter model"
- "Explain NCCL AllReduce and when to use it"
- "How would you optimize GPU utilization in training?"
- "Describe fault tolerance strategy for multi-day training"

**Project 202** prepares you for:
- "Design a model serving system for 1M+ requests/day"
- "Explain TensorRT optimization techniques"
- "How do you handle LLM serving at scale?"
- "Describe autoscaling strategy for GPU-based inference"

**Project 203** prepares you for:
- "Design a globally distributed ML platform"
- "How do you handle disaster recovery across regions?"
- "Explain data replication strategies and trade-offs"
- "How do you optimize costs in multi-region deployment?"

**Project 204** prepares you for:
- "Design a custom Kubernetes operator for [use case]"
- "Explain the controller reconciliation loop"
- "How do you implement multi-tenancy in operators?"
- "Describe testing strategy for Kubernetes operators"

---

## Project Dependencies

### Technical Dependencies

```
External Services:
├── Kubernetes Cluster (all projects)
│   ├── GPU support (Projects 201, 202, 204)
│   ├── Multi-cluster (Project 203)
│   └── Custom metrics (Project 202)
├── Cloud Provider Account (Projects 201, 202, 203)
│   ├── AWS, GCP, or Azure
│   └── Budget: $500-1000/month for full deployment
├── Container Registry (all projects)
└── Monitoring Stack (all projects)
    ├── Prometheus
    └── Grafana

Project-Specific:
├── Project 201:
│   ├── Ray cluster
│   ├── Shared storage (NFS/S3)
│   └── MLflow server
├── Project 202:
│   ├── TensorRT installation
│   ├── vLLM
│   └── Jaeger
├── Project 203:
│   ├── Multi-cloud access
│   ├── Terraform Cloud (optional)
│   └── DNS management
└── Project 204:
    ├── Kopf framework
    └── Kubernetes API access
```

### Learning Dependencies

**Before Starting**:
- [ ] Complete Junior Engineer track (recommended)
- [ ] Complete Engineer track (recommended)
- [ ] OR 5+ years experience in infrastructure

**Parallel Learning**:
- Ray documentation (Project 201)
- TensorRT documentation (Project 202)
- Terraform tutorials (Project 203)
- Kubernetes operator pattern (Project 204)

---

## Summary Statistics

### By Project

| Project | Hours | LOC | Tests | Coverage | Difficulty |
|---------|-------|-----|-------|----------|------------|
| P201 | 60h | 3,500 | 129 | 82% | ⭐⭐⭐⭐⭐ |
| P202 | 70h | 4,200 | 135 | 79% | ⭐⭐⭐⭐⭐ |
| P203 | 80h | 5,000 | 137 | 76% | ⭐⭐⭐⭐⭐ |
| P204 | 65h | 2,800 | 113 | 81% | ⭐⭐⭐⭐⭐ |
| **Total** | **275h** | **15,500** | **514** | **79.5%** | **Expert** |

### Technology Categories

**Most Covered**:
1. Kubernetes (all projects)
2. Python (all projects)
3. GPU Computing (P201, P202, P204)
4. Monitoring (all projects)

**Specialized**:
1. Ray (P201)
2. TensorRT/vLLM (P202)
3. Terraform (P203)
4. Kubernetes Operators (P204)

---

**Ready to start your journey?** Begin with [Project 201: Distributed Training](projects/project-201-distributed-training/)

---

**Last Updated**: October 25, 2025
**Version**: 1.0
**Maintainers**: AI Infrastructure Curriculum Team
