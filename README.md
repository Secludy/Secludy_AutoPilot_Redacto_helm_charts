# Secludy AutoPilot Redactor Helm Chart

This repository contains a Helm chart which can be used to install Secludy AutoPilot Redactor via `helm install`.

Project structure:
```
.
├── docker-compose.yml
├── helm-chart/
│   ├── templates/
│   │   └── <All template files>
│   ├── values.yaml
│   └── Chart.yaml
└── README.md
```

## Usage

[Helm](https://helm.sh) must be installed to use the charts. Please refer to Helm's [documentation](https://helm.sh/docs) to get started.

To install the secludy-autopilot-redactor chart:

    helm install -n <your-namespace> secludy ./helm-chart -f helm-chart/values.yaml

To uninstall the chart:

    helm uninstall -n <your-namespace> secludy

## Docker Compose Alternative

For non-Kubernetes deployments:

    export AWS_ACCESS_KEY_ID=<your-key>
    export AWS_SECRET_ACCESS_KEY=<your-secret>
    docker-compose up

## Configuration

### values.yaml

Before deploying, configure the following values:

### AWS Bedrock Configuration

* aws.region: AWS region for Bedrock (default: "us-west-2")
* aws.credentials.accessKeyId: Your AWS access key
* aws.credentials.secretAccessKey: Your AWS secret key

### Pattern Discovery Settings

* discovery.sampleRate: Percentage of data to sample (default: 0.001)
* discovery.maxPatterns: Maximum patterns to discover (default: 5)

### Detection Workers

* detection.replicas: Number of detection workers to deploy (default: 3)

### Storage

* storage.rules: Storage for generated rules (default: "1Gi")
* storage.input: Storage for input data (default: "10Gi")
* storage.output: Storage for output data (default: "10Gi")

### Image Configuration

* image.repository: Docker image repository (default: "secludy/autopilot-redactor")
* image.tag: Image version tag (default: "8")

## Deploy

Create a namespace:

    kubectl create namespace secludy

Deploy Secludy AutoPilot Redactor:

    helm install secludy -n secludy ./helm-chart -f helm-chart/values.yaml

### Validate the deployment

Use `kubectl get all -n secludy` to check that the pods are running:

The deployment proceeds in two phases:
1. **Rule Generation Job** - Generates detection rules using AWS Bedrock
2. **Detection Deployment** - Starts workers using the generated rules

Once the job shows COMPLETIONS 1/1 and the deployment shows READY replicas, Secludy is operational.

Check rule generation logs:

    kubectl logs job/secludy-rule-generation -n secludy

Check detection worker status:

    kubectl logs deployment/secludy-detection -n secludy

The API service will be available at the service endpoint once all pods are running.

## License

This project is licensed under the BSD 3-Clause License - see the [LICENSE](../LICENSE) file for details.