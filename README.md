# Complete CI/CD Pipeline with EKS and AWS ECR

## Project Overview
A full CI/CD pipeline for a Java Maven application using Jenkins, Docker, AWS ECR, and AWS EKS. The pipeline automates building, containerizing, storing, and deploying the application to a Kubernetes cluster on AWS.

---

## Prerequisites

### AWS Setup
- **AWS Account** with permissions to create/manage ECR and EKS.
- **ECR Repository** created for Docker images.
- **EKS Cluster** created and accessible. Use `eksctl`:
  ```sh
  eksctl create cluster -f eksctl-cluster.yaml
  ```
- **Kubeconfig**: Update with your EKS cluster details. Use:
  ```sh
  aws eks update-kubeconfig --region <region> --name <cluster-name>
  ```

### Jenkins Requirements
- **Jenkins** (running in a container or VM)
- **Required Plugins**:
  - Pipeline
  - Docker Pipeline
  - Kubernetes CLI
  - Credentials Binding
  - GitHub
  - Git

### Tools to Install in Jenkins Agent/Container
- **Docker**
- **kubectl**:
  ```sh
  curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
  chmod +x kubectl && mv kubectl /usr/local/bin/
  ```
- **aws-iam-authenticator**:
  ```sh
  curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.7.4/aws-iam-authenticator_0.7.4_linux_amd64
  chmod +x aws-iam-authenticator && mv aws-iam-authenticator /usr/local/bin/
  aws-iam-authenticator version
  ```
- **envsubst** (from gettext):
  - Debian/Ubuntu:
    ```sh
    apt-get update && apt-get install -y gettext
    ```
  - Alpine:
    ```sh
    apk add --no-cache gettext
    ```
  - RedHat/CentOS:
    ```sh
    yum install -y gettext
    ```

### Jenkins Credentials
- **AWS Access Key ID** and **Secret Access Key** (store as Jenkins credentials)
- **GitHub Personal Access Token** (for pushing version bumps)

---

## Setup Instructions

### 1. Jenkins Agent/Container
- Install all required tools (see above).
- Ensure Docker is running and accessible.

### 2. Kubeconfig Setup
- Copy your kubeconfig file to the Jenkins container:
  ```sh
  docker cp ~/.kube/config <jenkins-container>:/var/jenkins_home/.kube/config
  docker exec -it <jenkins-container> chown jenkins:jenkins /var/jenkins_home/.kube/config
  chmod 600 /var/jenkins_home/.kube/config
  ```

### 3. ECR Secret for Kubernetes
- Create a Kubernetes secret for ECR image pulls:
  ```sh
  kubectl create secret docker-registry aws-registry-key \
    --docker-server=<aws-account-id>.dkr.ecr.<region>.amazonaws.com \
    --docker-username=AWS \
    --docker-password="$(aws ecr get-login-password --region <region>)" \
    --docker-email=<your-email>
  ```
- Ensure `imagePullSecrets` in your deployment YAML matches the secret name.

---

## Pipeline Workflow

### Stages
1. **Increment Version**: Bumps the Maven version for each build.
2. **Build App**: Compiles and packages the Java app.
3. **Build Image**: Builds and pushes the Docker image to ECR.
4. **Deploy**: Deploys the image to EKS using `kubectl` and `envsubst`.
5. **Commit Version Update**: Pushes the new version to GitHub.

### Automated Version Bumping
- Uses Maven plugins to increment the version in `pom.xml`.
- Commits the new version back to the repository for traceability.

---

## Manual Operations

### Update kubeconfig
If your EKS cluster changes, update kubeconfig and copy it to Jenkins:
```sh
aws eks update-kubeconfig --region <region> --name <cluster-name>
docker cp ~/.kube/config <jenkins-container>:/var/jenkins_home/.kube/config
```

### Delete EKS Cluster
```sh
eksctl delete cluster -f eksctl-cluster.yaml
```

---

## Summary
This setup provides a robust, automated CI/CD pipeline for Java applications on AWS EKS, using Jenkins, Docker, and ECR. All build artifacts are ignored from version control, and credentials are securely managed in Jenkins.
