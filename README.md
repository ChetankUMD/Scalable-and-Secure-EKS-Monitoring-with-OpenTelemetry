# Scalable-and-Secure-EKS-Monitoring-with-OpenTelemetry
Scalable and Secure EKS Monitoring with OpenTelemetry

This project showcases how to deploy the [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo) on Amazon Elastic Kubernetes Service (EKS) using AWS CloudFormation, Helm, Prometheus, Grafana, and an end-to-end CI/CD pipeline.

## Table of Contents
1. [Architecture Overview](#architecture-overview)  
2. [Prerequisites](#prerequisites)  
3. [Repository Structure](#repository-structure)  
4. [Deployment Steps](#deployment-steps)  
5. [Monitoring & Observability](#monitoring--observability)  
6. [Alerting & Notifications](#alerting--notifications)  
7. [CI/CD Integration](#cicd-integration)  
8. [Contributors](#contributors)  

---

## Architecture Overview
1. **AWS CloudFormation** templates create a **VPC**, subnets, security groups, **EKS cluster**, and worker **EC2** instances.
2. **EKS** hosts the OpenTelemetry Demo application, which is deployed via **Helm** or **kubectl**.
3. **Prometheus** collects application and Kubernetes metrics, while **Grafana** visualizes them through custom dashboards.
4. **Jaeger** and **OpenTelemetry Collector** track distributed traces of the microservices.
5. **Alerting** is configured via Prometheus Alertmanager (or Grafana alerts) with email/Slack/SNS notifications.
6. A **CI/CD** pipeline (GitHub Actions + Docker Hub) automatically builds and deploys changes to the EKS cluster.

---

## Prerequisites
- AWS Account with IAM permissions for EKS, CloudFormation, and EC2.
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) and [eksctl](https://eksctl.io/) installed locally (or in an EC2 client instance).
- [kubectl](https://kubernetes.io/docs/tasks/tools/) and [Helm](https://helm.sh/docs/intro/install/) installed.
- Docker Hub (or an alternative container registry) for pushing/pulling Docker images.
- GitHub repository for CI/CD workflows (optional if you want to replicate the full pipeline).

---

## Repository Structure

├── CloudFormation Templates/
│   ├── 1_vpc_template.yaml
│   ├── 2_security_groups.yaml
│   ├── 3_eks_cluster_template.yaml
│   ├── 4_eks_nodes.yaml
│   └── 5_eks_client_ec2.yaml
├── Kubernetes/
│   ├── monolithic_opentelemetry_demo.yaml       # (Original single manifest)
│   ├── splitted-manifests/
│   │   ├── 01-namespaces/
│   │   ├── 02-core/
│   │   ├── 03-opentelemetry-collector/
│   │   ├── 04-jaeger/
│   │   ├── 05-prometheus/
│   │   ├── 06-grafana/
│   │   ├── 07-opensearch/
│   │   └── 08-application/
│   └── Helm/
│       └── helm_chart_values.yaml              # Custom values for Helm deployment
├── ci-cd-pipeline.yml                          # Sample GitHub Actions workflow
├── Dockerfile                                  # For building custom images
├── README.md                                   # (This file)

- **CloudFormation Templates/**: Templates for VPC, security groups, EKS cluster, node groups, and an EC2 bastion/client instance.
- **Kubernetes/monolithic_opentelemetry_demo.yaml**: The original single YAML deployment for the OpenTelemetry demo.
- **Kubernetes/splitted-manifests/**: Organized, service-specific manifests for easier debugging and partial deployments.
- **Kubernetes/Helm/**: Helm charts or custom `values.yaml` used to manage and deploy the application.
- **ci-cd-pipeline.yml**: Demonstrates GitHub Actions to build/push Docker images and deploy to EKS automatically.

---

## Deployment Steps

### 1. Provision AWS Infrastructure
1. Clone this repository:
   ```bash
   git clone https://github.com/<your-username>/<your-repo>.git
   cd <your-repo>

2.	Deploy CloudFormation stacks in order:
# VPC stack
aws cloudformation create-stack --stack-name vpc-stack \
  --template-body file://CloudFormation\ Templates/1_vpc_template.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Security Groups
aws cloudformation create-stack --stack-name sg-stack \
  --template-body file://CloudFormation\ Templates/2_security_groups.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# EKS Cluster
aws cloudformation create-stack --stack-name eks-cluster-stack \
  --template-body file://CloudFormation\ Templates/3_eks_cluster_template.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# EKS Nodes
aws cloudformation create-stack --stack-name eks-nodes-stack \
  --template-body file://CloudFormation\ Templates/4_eks_nodes.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Optional: EC2 Client (bastion)
aws cloudformation create-stack --stack-name eks-client-stack \
  --template-body file://CloudFormation\ Templates/5_eks_client_ec2.yaml \
  --capabilities CAPABILITY_NAMED_IAM

3.	Wait for each stack to complete. Check AWS Console or CLI:
aws cloudformation describe-stacks --stack-name <stack-name>
  --aws eks update-kubeconfig --region <your-region> --name <eks-cluster-name>
  --kubectl apply -R -f Kubernetes/splitted-manifests/

4. Helm-Based Deployment
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Create a new namespace (e.g., "otel-demo-helm")
kubectl create namespace otel-demo-helm

# Deploy the chart
helm install my-otel-demo open-telemetry/opentelemetry-demo --namespace otel-demo-helm

5.	Verify all pods and services:
    --kubectl get all -n otel-demo-helm

6. Monitoring & Observability
	1.	Prometheus collects cluster metrics (CPU, memory, node status) via kube-state-metrics, node-exporter, and custom scraping configs for the OpenTelemetry Collector.
	2.	Jaeger receives traces from the OpenTelemetry Collector to visualize distributed traces of the microservices.
	3.	Grafana dashboards are provided for:
	•	Cluster-Level Metrics (total pods, node health, CPU, memory usage)
	•	Node-Level Metrics (CPU/memory/disk usage per node)
	•	Pod-Level Metrics (restarts, availability)
	•	Service-Level (latency, error rates)
	•	Network & Storage (pod-to-pod traffic, persistent volume usage)

7. Alerting & Notifications
	•	Prometheus Alert Rules monitor pod restarts, CPU usage, memory thresholds, etc.
	•	Grafana Alerts (time-series based) notify on high CPU, memory, or error rates.
	•	Notifications can be sent via:
	•	Amazon SES or SNS (email/SMS)
	•	Slack, PagerDuty, or other supported channels

8. CI/CD Integration

A sample GitHub Actions workflow demonstrates:
	1.	Building and pushing Docker images to Docker Hub (or AWS ECR).
	2.	Deploying the new container image to EKS.
	3.	Using kubectl and eksctl within the pipeline to automate rollouts.

Usage:
	1.	Update credentials in GitHub Secrets:
	•	AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY for AWS.
	•	DOCKERHUB_USERNAME / DOCKERHUB_TOKEN for Docker Hub (if used).
	2.	Update the pipeline file (branch name, image repo, cluster name).
	3.	Commit and push changes to trigger the workflow.