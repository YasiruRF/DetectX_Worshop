# DetectX — Solution Architecture

## Executive Summary

DetectX is an AI-native Automated Quality Control (AQC) system for semiconductor and motherboard manufacturing. It provides high-throughput, low-latency defect detection for PCBs and assembled boards using an optimized YOLO-based CV pipeline deployed on on-prem GPU servers with containerized services and Kubernetes orchestration.

## Goals

- Detect visual defects (scratches, component misalignment, soldering faults) in real time.
- Maintain per-unit latency suitable for production-line speeds (target: 100–300 ms).
- Keep data on-premises for security and compliance.
- Provide traceability (raw image archive, inference results) and an automated retraining loop.

## Key Assumptions

- Production: ~10,000 units/day (adjustable for multiple lines and bursts).
- Environment: on-prem servers with NVIDIA A100/H100 GPUs available.
- Network: local, high-bandwidth LAN between edge and server cluster.

## Requirements

Functional

- Real-time capture and inference for each inspected unit.
- Human QC console for reviewing flagged defects.
- Integration with MES/PLC for automated line control.

Non-functional

- Latency: ≤300 ms per unit (goal ≤100 ms with optimization).
- Throughput: scale horizontally to handle peak line bursts.
- Security: TLS, RBAC, on-prem storage.
- Observability: metrics, logs, and alerting.

## High-level Architecture

- Edge: high-speed cameras → frame grabber (FPGA/CPU) → edge server (Docker) running preprocessing and optional lightweight model.
- Inference: Kubernetes cluster with GPU node pool serving optimized YOLO models (Triton/ONNX/TensorRT).
- Post-processing: rules engine → results DB + alerting + human QC console.
- Storage: on-prem S3 (MinIO) for image archive; PostgreSQL for structured results.
- MLOps: MLflow/DVC for experiments and dataset versioning; private registry (Harbor) for images.

## Detailed Components

1. Capture & Edge

- Cameras: industrial high-speed cameras (GenICam compatible) capturing frames per unit.
- Frame Grabber: deterministic capture; optionally implemented on FPGA for precise timing.
- Edge Preprocessor: Docker container performs ROI extraction, resize, denoise, color correction, and lightweight filtering.

2. Inference

- Model: YOLO variant (YOLOX / YOLOv8 recommended). Train on labeled defect dataset; export to ONNX then optimize via TensorRT.
- Serving: NVIDIA Triton Server recommended for multi-model, GPU-optimized serving. Alternatives: ONNX Runtime Server or TorchServe.
- Batch strategy: micro-batching with bounded latency to improve GPU utilization.

3. Post-processing & Business Rules

- Consolidate detections across frames, apply confidence thresholds, map to defect classes, and create structured events.
- Rules engine configurable via YAML/DB to allow line-specific policies.

4. Storage & Traceability

- Results DB: PostgreSQL (HA). Stores inference outcome, timestamps, part IDs, and pointers to archived images.
- Image Archive: MinIO (S3) on-prem for raw frames and audit trail. Retention policy configurable.

5. MLOps & CI/CD

- Training: GPU workstation/cluster; experiment tracking in MLflow, dataset versioning with DVC.
- Registry & artifacts: Harbor or private registry for container images; MLflow registry for models.
- CI/CD: GitHub Actions builds containers, runs unit tests, pushes to registry; ArgoCD/Flux for K8s GitOps deployments.

6. Observability & Ops

- Metrics: Prometheus scraping Triton, preprocessing, and business services.
- Dashboards: Grafana for throughput, latency, and accuracy trends.
- Logs: Loki or ELK stack; alerts via Alertmanager.

## Tech Stack (Recommended)

- Capture: GStreamer, GenICam drivers.
- Preprocessing: Python (OpenCV, NumPy) or C++ libs for high throughput.
- Model: YOLOX/YOLOv8 → ONNX → TensorRT (FP16/INT8 where safe).
- Model Server: NVIDIA Triton.
- Orchestration: Kubernetes (K8s) with node pools for GPU/CPU.
- Container registry: Harbor or Artifactory.
- Storage: MinIO (S3 compatible), PostgreSQL (HA via Patroni).
- MLOps: MLflow, DVC.
- CI/CD: GitHub Actions + ArgoCD/Flux (GitOps).
- Monitoring: Prometheus, Grafana, Loki/ELK.
- Security: HashiCorp Vault, Trivy for image scanning, network segmentation.

## Deployment Topology

Key deployment notes:

- Edge servers run Docker containers for low-latency preprocessing and buffering.
- K8s cluster deployed on-prem with GPU node pool for inference (A100/H100), CPU node pool for other services.
- Data plane flows: Camera → Edge → Inference Pods → Post-processing → DB/Archive → MES/Console.

Mermaid: System Overview

```mermaid
flowchart LR
  subgraph Edge
    A[Cameras / Line Sensors] --> B[Frame Grabber (FPGA/CPU)]
    B --> C[Edge Preprocessor (resize, denoise, augment)]
  end
  C --> D[Inference Cluster (K8s, GPU nodes)]
  subgraph Inference
    D --> E[YOLO Inference (Triton / ONNX / TorchServe)]
    E --> F[Post-processing & Rules Engine]
  end
  F --> G[(Results DB) PostgreSQL]
  F --> H[Alerting / Human QC Console]
  E --> I[Image Archive (on-prem S3)]
  D --> J[Model Registry (MLflow / Harbor)]
  subgraph Observability
    K[Prometheus] --> L[Grafana]
    E --> K
    D --> K
  end
  G --> L
  H --> MES[Manufacturing Execution System / PLC]
  MES --> LineControl[Line Control]
```

Mermaid: Deployment Diagram

```mermaid
flowchart TB
  subgraph CI/CD
    GH[GitHub Actions] --> CR[Container Registry]
    CR --> K8s
  end
  subgraph OnPrem
    K8s[Kubernetes Cluster]
    K8s --> GPU[GPU Node Pool (A100/H100)]
    K8s --> CPU[CPU Node Pool]
    GPU --> InferencePod[YOLO Inference Pods (Triton)]
    CPU --> PreprocPod[Preprocessing Pods]
    K8s --> Storage[MinIO (S3) / NFS]
    K8s --> DB[Postgres HA]
    K8s --> Monitoring[Prometheus + Grafana]
  end
  Camera[High-speed Cameras] --> EdgeBox[Edge Server (Docker) \n low-latency preprocessing]
  EdgeBox --> K8s
```

## Sizing & Benchmark Plan

- Start small: 1 A100 for baseline inference tests on a representative sample.
- Benchmark steps:
  1. Export model to ONNX.
  2. Optimize with TensorRT (FP16, INT8 where safe).
  3. Run Triton perf client to measure p99 latency and throughput per GPU.
  4. Increase concurrency and batch size to find sweet spot between latency and GPU utilization.

## Operational Playbook (Short)

- Rolling updates: use Triton model control + K8s Pod disruption budgets.
- Incident: failover to human QC console; buffer images on edge for re-processing.
- Retraining: schedule weekly or per-drift threshold; label in human QC console and feed into DVC pipeline.

## Security Considerations

- Network segmentation: isolate camera/edge network from corporate network.
- Secrets: Vault for DB and registry credentials.
- Image Scanning: Trivy in CI pipeline.

## Next Steps (recommended immediate tasks)

1. Run an end-to-end POC: single camera → edge preprocess → Triton inference on one GPU node.
2. Collect 2–4 weeks of labeled images for robust training.
3. Define retention policy and archival storage sizing.

---

Document created for initial review. Feedback will be incorporated and diagrams can be exported to PNG/SVG on request.
