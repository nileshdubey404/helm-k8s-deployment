# Helm Kubernetes Deployment

A CI/CD project that automates building, containerizing, and deploying a React frontend application to Amazon EKS using Jenkins, Docker, Amazon Elastic Container Registry (ECR), Kubernetes, and Helm.

## Overview

This project demonstrates an end-to-end delivery workflow: code is checked out by Jenkins, built into a Docker image, pushed to Amazon ECR, and deployed to an Amazon EKS cluster using a Helm chart. Kubernetes rollout and resource checks help verify the deployment.

## Architecture

```text
Developer Push
     |
     v
Jenkins Pipeline
     |
     +--> Checkout Source Code
     |
     +--> Build Docker Image
     |
     +--> Tag and Push Image ------> Amazon ECR
     |
     +--> Configure kubectl -------> Amazon EKS
                                      |
                                      v
                               Helm Deployment
                                      |
                         +------------+------------+
                         v            v            v
                    Deployment     Service       Ingress
                         |
                         v
                  Kubernetes Pods
```

## Technology Stack

- **Frontend:** React, Vite
- **Containerization:** Docker, Nginx
- **CI/CD:** Jenkins
- **Container Registry:** Amazon ECR
- **Orchestration:** Kubernetes on Amazon EKS
- **Package Management / Deployment:** Helm
- **Cloud Provider:** AWS
- **Ingress:** Kubernetes Ingress configured for the AWS Load Balancer Controller

## Key Features

- Automated CI/CD pipeline using a Jenkins pipeline.
- Multi-stage Docker build: compiles the frontend with Node.js and serves the static build using Nginx.
- Docker image tags use the Jenkins build number, alongside a `latest` tag.
- Images are pushed to Amazon ECR.
- Helm manages the Kubernetes deployment, service, and ingress resources.
- Configurable replica count and container resource requests/limits.
- Readiness and liveness HTTP probes for the frontend container.
- Pipeline checks rollout status and prints Kubernetes resources after deployment.

## Repository Structure

```text
helm-k8s-deployment/
├── charts/
│   └── frontend-chart/
│       ├── templates/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── ingress.yaml
│       │   └── NOTES.txt
│       ├── Chart.yaml
│       ├── values.yaml
│       └── values-dev.yaml
├── src/
├── public/
├── Dockerfile
├── Jenkinsfile
├── iam_policy.json
├── package.json
└── README.md
```

## CI/CD Pipeline Workflow

The Jenkins pipeline contains these stages:

1. **Checkout:** Retrieves the source code from the configured SCM.
2. **Login to Amazon ECR:** Authenticates Docker to the ECR registry using AWS CLI credentials.
3. **Build Docker Image:** Builds the frontend image from the Dockerfile.
4. **Tag Docker Image:** Adds the Jenkins build number and `latest` tags.
5. **Push Docker Image:** Uploads the image tags to Amazon ECR.
6. **Configure kubectl:** Retrieves EKS cluster credentials using `aws eks update-kubeconfig`.
7. **Deploy with Helm:** Installs or upgrades the frontend release using `charts/frontend-chart` and the development values file.
8. **Verify Rollout:** Waits for the Kubernetes deployment rollout to complete.
9. **Verify Resources:** Lists pods, services, ingress resources, and deployments.

The pipeline also reports success or failure and cleans the Jenkins workspace after execution.

## Prerequisites

Make sure you have the following configured before running the pipeline:

- An AWS account with permissions required for ECR, EKS, and the selected deployment workflow.
- An Amazon ECR repository for the frontend image.
- An Amazon EKS cluster and a working Kubernetes context.
- Jenkins with access to the source repository and the required build tools.
- Docker, AWS CLI, `kubectl`, and Helm installed on the Jenkins agent.
- Node.js and npm for local frontend development.
- An AWS Load Balancer Controller and suitable AWS networking configuration if you intend to expose the application through the configured ALB Ingress.
- IAM permissions for Jenkins and the cluster components, following the principle of least privilege.

## Run Locally

Clone the repository and install the frontend dependencies:

```bash
git clone https://github.com/nileshdubey404/helm-k8s-deployment.git
cd helm-k8s-deployment
npm ci
```

Start the local development server:

```bash
npm run dev
```

Create a production build:

```bash
npm run build
```

Run the linter:

```bash
npm run lint
```

## Build the Docker Image

From the repository root:

```bash
docker build -t frontend:local .
```

Run the container locally:

```bash
docker run --rm -p 8080:80 frontend:local
```

Open `http://localhost:8080` in a browser.

## Deploy with Helm

The Jenkins pipeline is configured to deploy the chart to EKS. For a manual deployment, first ensure your AWS credentials and Kubernetes context are configured and that the image is available in ECR.

Review the chart settings in:

- `charts/frontend-chart/values.yaml`
- `charts/frontend-chart/values-dev.yaml`

Install or upgrade the release by replacing the image registry, tag, cluster context, and namespace with values for your environment:

```bash
helm upgrade --install frontend ./charts/frontend-chart \
  --namespace development \
  --create-namespace \
  -f charts/frontend-chart/values-dev.yaml \
  --set image.repository=<ECR_IMAGE_URI> \
  --set image.tag=<IMAGE_TAG>
```

Check the deployment and service:

```bash
kubectl rollout status deployment/frontend -n development
kubectl get pods,svc,ingress,deployments -n development
```

## Helm Configuration

The chart's default values specify:

| Setting | Default | Purpose |
|---|---|---|
| Replicas | `2` | Number of frontend pod replicas |
| Service type | `ClusterIP` | Exposes the app inside the cluster |
| Service port | `80` | Service port |
| Container target port | `80` | Nginx container port |
| CPU request | `100m` | Requested CPU per container |
| Memory request | `128Mi` | Requested memory per container |
| CPU limit | `500m` | Maximum CPU per container |
| Memory limit | `512Mi` | Maximum memory per container |
| Ingress class | `alb` | Ingress class for AWS Load Balancer Controller |

Adjust values for your cluster and workload before using this setup in production.

## AWS and Security Notes

- The repository includes `iam_policy.json` for AWS Load Balancer Controller-related permissions. Review it carefully and follow the official AWS guidance before attaching or adapting any IAM policy.
- Configure AWS credentials through a secure Jenkins credential mechanism or IAM role; do not commit access keys or secrets.
- Replace environment-specific AWS account, region, ECR repository, EKS cluster, release, and namespace settings in the Jenkinsfile to match your environment.
- Use least-privilege IAM policies and restrict cluster access.
- The pipeline uses a mutable `latest` tag in addition to the build-number tag. Prefer immutable versioned tags for traceable production releases.
- Review the pipeline's rollback behavior before relying on it for production recovery, and test rollback procedures in a non-production environment.

## Troubleshooting

- **ECR login or push fails:** Check AWS identity, region, repository name, and ECR permissions.
- **EKS access fails:** Confirm the cluster name/region and that the Jenkins identity is authorized in the cluster.
- **Pods cannot pull the image:** Verify the ECR image URI/tag and node or workload permissions to pull from ECR.
- **Ingress does not receive traffic:** Check AWS Load Balancer Controller health, ingress annotations, subnet tags, security groups, and AWS networking.
- **Rollout fails:** Inspect `kubectl describe deployment`, pod events, and container logs in the target namespace.

## Learning Outcomes

This project provides hands-on practice with CI/CD pipeline design, Docker multi-stage builds, AWS container registry workflows, Kubernetes deployments, Helm chart configuration, ingress resources, and rollout verification.

## License

No license is specified in this repository. Add a license file if you intend to define reuse and distribution permissions.
