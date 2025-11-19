# nvidia-gpu-operator-monitoring

Repository that provides a way to monitor if the GPUs properly work in a Kubernetes Cluster.

This Helm chart deploys a CronJob that periodically runs a custom-built CUDA vector-add sample to verify GPU availability, along with a PrometheusRule that alerts when GPUs become unavailable.

## Features

- **Custom CUDA Vector Add Sample**: Built from source using the latest CUDA version (currently 12.6.2)
- **Automated Image Builds**: GitHub Actions workflow builds and publishes Docker images to GitHub Container Registry
- **Automatic CUDA Updates**: Dependabot monitors CUDA base image updates and automatically updates the Helm chart
- **Vector Add Checker CronJob**: Runs on a configurable schedule (default: every 10 minutes) to verify GPU functionality
- **Prometheus Alerting**: Alerts when GPU nodes haven't successfully run jobs for an extended period
- **Fully Configurable**: Easy to customize alert severity, cluster information, schedule, and enable/disable monitoring

## Repository Structure

```
.
├── docker/                          # Docker image for CUDA vector-add sample
│   └── Dockerfile                   # Multi-stage build using NVIDIA CUDA base images
├── .github/
│   ├── workflows/
│   │   ├── build-docker.yml        # Builds and pushes Docker image on changes
│   │   └── update-helm-chart.yml   # Auto-updates Helm chart when CUDA version changes
│   └── dependabot.yml              # Monitors CUDA and GitHub Actions updates
└── nvidia-gpu-operator-monitoring/  # Helm chart
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── vector-add-checker.yaml  # CronJob and PrometheusRule templates
        ├── NOTES.txt
        └── _helpers.tpl
```

## Installation

### Prerequisites

- Kubernetes cluster with NVIDIA GPU Operator installed
- Helm 3.x
- Prometheus Operator (for PrometheusRule support)
- Nodes with GPUs labeled with `nvidia.com/gpu.present: 'true'`

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
  --set vectorAddChecker.schedule="*/5 * * * *" \
  --set usecase=production \
  --set clusterEnvironment=prod
```

## Configuration

The following table lists the configurable parameters and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `vectorAddChecker.enabled` | Enable/disable the vector-add-checker | `true` |
| `vectorAddChecker.image.repository` | Container image repository | `ghcr.io/lukas-holzner/nvidia-gpu-operator-monitoring/cuda-vector-add` |
| `vectorAddChecker.image.tag` | Image tag (defaults to Chart.appVersion) | `cuda-12.6.2` |
| `vectorAddChecker.image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `vectorAddChecker.schedule` | CronJob schedule | `*/10 * * * *` (every 10 minutes) |
| `vectorAddChecker.severity` | Alert severity level | `warning` |
| `vectorAddChecker.alert.timeThreshold` | Time in minutes before alert triggers | `12` |
| `vectorAddChecker.alert.for` | Duration alert must persist before firing | `30m` |
| `vectorAddChecker.alert.labels` | Additional custom labels for the alert | `{}` |
| `vectorAddChecker.alert.annotations` | Additional custom annotations for the alert | `{}` |
| `usecase` | Use case identifier for cluster | `example` |
| `clusterEnvironment` | Cluster environment (dev, staging, prod, etc.) | `dev` |
| `namespace` | Namespace where CronJob metrics are monitored | `nvidia-gpu-operator` |

## How It Works

### 1. Custom CUDA Image Build

The `docker/Dockerfile` contains a multi-stage build that:
- Uses NVIDIA CUDA development image to compile the vector-add sample
- Creates a minimal runtime image with only the compiled binary
- Currently uses CUDA 12.6.2 (automatically updated by Dependabot)

### 2. Automated Updates

When Dependabot detects a new CUDA version:
1. It opens a PR updating the Dockerfile
2. The `update-helm-chart.yml` workflow automatically:
   - Extracts the new CUDA version
   - Updates the Helm chart's `appVersion`
   - Bumps the chart version
   - Updates the default image tag in `values.yaml`
3. The `build-docker.yml` workflow builds and publishes the new image

### 3. CronJob Execution

The `vector-add-checker` CronJob:
- Runs on the configured schedule
- Targets nodes with `nvidia.com/gpu.present: 'true'`
- Requests 1 GPU resource
- Executes the CUDA vector-add sample to verify GPU functionality

### 4. Metrics Collection & Alerting

The PrometheusRule monitors `kube_cronjob_status_last_successful_time` and triggers an alert if:
- No successful job completion in the configured time threshold (default: 12 minutes)
- This condition persists for the configured duration (default: 30 minutes)

## Monitoring

### Check CronJob Status

```bash
kubectl get cronjob -n nvidia-gpu-operator
```

### View Recent Jobs

```bash
kubectl get jobs -n nvidia-gpu-operator -l app.kubernetes.io/component=gpu-checker
```

### Check Job Logs

```bash
kubectl logs -n nvidia-gpu-operator -l app.kubernetes.io/component=gpu-checker --tail=50
```

### Check PrometheusRule

```bash
kubectl get prometheusrule -n nvidia-gpu-operator
```

## Development

### Building the Docker Image Locally

```bash
cd docker
docker build -t cuda-vector-add:local .
```

### Testing the Image

```bash
docker run --rm --gpus all cuda-vector-add:local
```

Expected output:
```
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

### Testing Helm Chart Locally

```bash
# Lint the chart
helm lint nvidia-gpu-operator-monitoring

# Render templates
helm template test-release nvidia-gpu-operator-monitoring

# Dry-run install
helm install test-release nvidia-gpu-operator-monitoring --dry-run --debug
```

## Uninstallation

```bash
helm uninstall nvidia-gpu-monitoring -n nvidia-gpu-operator
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License.

