# nvidia-gpu-operator-monitoring

Repository that provides a way to monitor if the GPUs properly work in a Kubernetes Cluster.

This Helm chart deploys a CronJob that periodically runs a CUDA vector-add sample to verify GPU availability, along with a PrometheusRule that alerts when GPUs become unavailable.

## Features

- **Vector Add Checker CronJob**: Runs every 10 minutes to verify GPU functionality using NVIDIA's CUDA samples
- **Prometheus Alerting**: Alerts when GPU nodes haven't successfully run jobs for an extended period
- **Configurable**: Easy to customize alert severity, cluster information, and enable/disable monitoring

## Installation

### Prerequisites

- Kubernetes cluster with NVIDIA GPU Operator installed
- Helm 3.x
- Prometheus Operator (for PrometheusRule support)

### Install the Chart

```bash
helm install nvidia-gpu-monitoring ./nvidia-gpu-operator-monitoring \
  --namespace nvidia-gpu-operator \
  --create-namespace
```

### Install with Custom Values

```bash
helm install nvidia-gpu-monitoring ./nvidia-gpu-operator-monitoring \
  --namespace nvidia-gpu-operator \
  --create-namespace \
  --set vectorAddChecker.severity=critical \
  --set usecase=production \
  --set clusterEnvironment=prod
```

## Configuration

The following table lists the configurable parameters and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `vectorAddChecker.enabled` | Enable/disable the vector-add-checker | `true` |
| `vectorAddChecker.repository` | Container image repository for CUDA samples | `docker.io/nvidia` |
| `vectorAddChecker.severity` | Alert severity level (warning, critical, etc.) | `warning` |
| `usecase` | Use case identifier for cluster | `example` |
| `clusterEnvironment` | Cluster environment (dev, staging, prod, etc.) | `dev` |

## How It Works

1. **CronJob Execution**: The `vector-add-checker` CronJob runs every 10 minutes on nodes with GPUs (identified by the `nvidia.com/gpu.present: 'true'` node selector)

2. **GPU Verification**: Each job runs a CUDA vector-add sample to verify that GPUs are functioning correctly

3. **Metrics Collection**: Job success/failure is tracked by Kubernetes metrics (via kube-state-metrics)

4. **Alerting**: The PrometheusRule monitors the `kube_cronjob_status_last_successful_time` metric and triggers an alert if:
   - No successful job completion in the last 12 minutes
   - This condition persists for 30 minutes

## Monitoring

### Check CronJob Status

```bash
kubectl get cronjob vector-add-checker -n nvidia-gpu-operator
```

### View Recent Jobs

```bash
kubectl get jobs -n nvidia-gpu-operator -l job-name=vector-add-checker
```

### Check PrometheusRule

```bash
kubectl get prometheusrule gpu-rules -n nvidia-gpu-operator
```

## Uninstallation

```bash
helm uninstall nvidia-gpu-monitoring -n nvidia-gpu-operator
```

## License

This project is licensed under the MIT License.

