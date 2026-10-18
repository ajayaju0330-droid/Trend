# Trendify - React App Deployment (Docker, Terraform, EKS, Jenkins)

Production-style deployment of the Trendify React app (https://github.com/Vennilavanguvi/Trend.git) on AWS.

**Docker Hub image:** `ajayrag/trend-app`
**Load Balancer ARN:** `arn:aws:elasticloadbalancing:us-west-2:196390795947:loadbalancer/a0f4a21e8a93c487f975667000dbe1aa`
**App URL:** http://a0f4a21e8a93c487f975667000dbe1aa-1744414188.us-west-2.elb.amazonaws.com

## Architecture

GitHub push -> webhook -> Jenkins (EC2) -> build Docker image -> push to Docker Hub -> deploy to Amazon EKS -> Kubernetes LoadBalancer service -> users.

## Repository layout

- `dist/` - prebuilt React production bundle
- `Dockerfile`, `nginx.conf` - nginx image serving the app on port 3000
- `.dockerignore`, `.gitignore` - ignore rules
- `terraform/main.tf` - VPC, subnets, IAM role, security group, Jenkins EC2
- `k8s/deployment.yaml`, `k8s/service.yaml` - Kubernetes manifests
- `Jenkinsfile` - declarative CI/CD pipeline

## Setup instructions

1. **Docker:** the app is served by nginx on port 3000.

        docker build -t trend-app:v1 .
        docker run -d -p 3000:3000 trend-app:v1

2. **Docker Hub:** public repo `ajayrag/trend-app`; image pushed with `docker push`.

3. **Terraform:** creates the VPC, 2 public subnets, internet gateway, IAM role and instance profile, security group, and an EC2 instance that installs Jenkins, Docker, kubectl and AWS CLI on first boot.

        cd terraform
        terraform init
        terraform plan
        terraform apply

4. **EKS cluster:** created with eksctl inside the Terraform VPC (2 x t3.medium nodes).

        eksctl create cluster --name trend-cluster --region us-west-2 \
          --vpc-public-subnets <subnet-1>,<subnet-2> --node-type t3.medium --nodes 2

   The Jenkins IAM role was granted cluster admin through an EKS access entry.

5. **Kubernetes:** `deployment.yaml` runs 2 replicas on port 3000; `service.yaml` exposes them through a LoadBalancer (port 80 to 3000).

6. **Jenkins:** plugins installed: Docker Pipeline, Docker, Kubernetes, Kubernetes CLI, GitHub, Git. A Docker Hub credential (`dockerhub-creds`) is stored in Jenkins. A pipeline job `trend-pipeline` uses "Pipeline script from SCM" pointing at this repo.

7. **Webhook:** GitHub webhook to `http://<jenkins-ip>:8080/github-webhook/` (push events) so every commit triggers a build.

8. **Monitoring:** Prometheus and Grafana (open source) installed with Helm.

        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
        helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring \
          --set grafana.service.type=LoadBalancer

## Pipeline explanation

The `Jenkinsfile` defines four stages:

1. **Checkout** - pulls the latest code from GitHub.
2. **Build Image** - builds the Docker image tagged with the build number and `latest`.
3. **Push Image** - logs in to Docker Hub with the stored credential and pushes both tags.
4. **Deploy to EKS** - updates kubeconfig, applies the `k8s/` manifests, sets the new image, and waits for the rollout to finish.

## Screenshots

| Step | Screenshot |
|------|------------|
| Workstation EC2 running | ![](screenshots/01-workstation-ec2-running.png) |
| SSH connected | ![](screenshots/02-ssh-connected.png) |
| Git and Docker installed | ![](screenshots/03-git-docker-installed.png) |
| Repo cloned | ![](screenshots/04-repo-cloned.png) |
| Repo contents | ![](screenshots/05-repo-contents.png) |
| Dockerfile created | ![](screenshots/06-dockerfile-created.png) |
| Docker build | ![](screenshots/07-docker-build.png) |
| Container running | ![](screenshots/08-container-running.png) |
| Security group port 3000 | ![](screenshots/09-sg-port-3000.png) |
| App in browser | ![](screenshots/10-app-in-browser.png) |
| Docker Hub repo | ![](screenshots/11-dockerhub-repo.png) |
| Docker Hub push | ![](screenshots/12-dockerhub-push.png) |
| Ignore files | ![](screenshots/13-ignore-files.png) |
| GitHub repo created | ![](screenshots/14-github-repo-created.png) |
| Git commit | ![](screenshots/15-git-commit.png) |
| GitHub push | ![](screenshots/16-github-push.png) |
| Terraform and AWS CLI installed | ![](screenshots/17-terraform-awscli-installed.png) |
| AWS configured | ![](screenshots/18-aws-configured.png) |
| Terraform validate | ![](screenshots/19-terraform-validate.png) |
| Terraform plan | ![](screenshots/20-terraform-plan.png) |
| Terraform apply | ![](screenshots/21-terraform-apply.png) |
| Jenkins server check | ![](screenshots/22-jenkins-server-check.png) |
| Jenkins login | ![](screenshots/23-jenkins-unlock.png) |
| Jenkins plugins installed | ![](screenshots/24-jenkins-plugins-installed.png) |
| Jenkins Docker Hub credential | ![](screenshots/25-jenkins-dockerhub-creds.png) |
| eksctl installed | ![](screenshots/26-eksctl-installed.png) |
| EKS cluster created | ![](screenshots/27-eks-cluster-created.png) |
| EKS nodes ready | ![](screenshots/28-eks-nodes-ready.png) |
| Kubernetes YAML files | ![](screenshots/29-k8s-yaml-files.png) |
| EKS access for Jenkins | ![](screenshots/30-eks-jenkins-access.png) |
| GitHub push (all code) | ![](screenshots/31-github-push-all.png) |
| Pipeline first successful build | ![](screenshots/33-pipeline-first-build.png) |
| Pods and service | ![](screenshots/34-k8s-pods-service.png) |
| App via Load Balancer | ![](screenshots/35-app-via-loadbalancer.png) |
| Load Balancer name | ![](screenshots/36-loadbalancer-name.png) |
| GitHub webhook | ![](screenshots/37-github-webhook.png) |
| Webhook auto build | ![](screenshots/38-webhook-auto-build.png) |
| Helm installed | ![](screenshots/39-helm-installed.png) |
| Monitoring pods | ![](screenshots/40-monitoring-pods.png) |
| Grafana dashboard | ![](screenshots/41-grafana-dashboard.png) |
