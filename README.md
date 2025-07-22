# Complete-CICD-Pipeline-with-EKS-and-AWS-ECR

Capstone Project 1: Complete CI/CD Pipeline with EKS and AWS ECR

## Project Overview
This project demonstrates a full CI/CD pipeline for a Java Maven application using Jenkins, Docker, AWS ECR (Elastic Container Registry), and AWS EKS (Elastic Kubernetes Service). The pipeline automates building, containerizing, storing, and deploying the application to a Kubernetes cluster on AWS.

## Required Jenkins Plugins
To run this pipeline, install the following Jenkins plugins:

- **Pipeline**: Enables Jenkins Pipeline (Declarative and Scripted Pipeline support).
- **Docker Pipeline**: Allows Jenkins to build and interact with Docker containers in pipelines.
- **Kubernetes CLI Plugin**: Provides the kubectl CLI tool for interacting with Kubernetes clusters.
- **Credentials Binding Plugin**: Allows you to bind credentials (like AWS keys, GitHub tokens) to environment variables in your pipeline.
- **GitHub Plugin**: Integrates Jenkins with GitHub repositories.
- **Git Plugin**: Provides Git SCM support for Jenkins.

## Prerequisites
To successfully run this pipeline, ensure the following prerequisites are met:

- **AWS Account**: You must have an AWS account with permissions to create and manage ECR repositories and EKS clusters.
- **ECR Repository**: An AWS ECR repository must be created to store Docker images.
- **EKS Cluster**: An AWS EKS cluster must be set up and accessible.
    - If you do not have an EKS cluster, you can create one using the provided `eksctl-cluster.yaml` file and the [eksctl](https://eksctl.io/) CLI:
      - Run: `eksctl create cluster -f eksctl-cluster.yaml`
    - The cluster name does not need to be hardcoded in your manifests or Jenkinsfile; deployments will go to the cluster configured in your kubeconfig context.
    - The standard kubeconfig template (`config.yaml`) is provided in this repository. **You must fill in your EKS cluster details** (certificate-authority-data, endpoint URL, and cluster name) as described in the instructions below. This file is required for Jenkins to authenticate and deploy to your EKS cluster.
- **IAM Permissions**: Jenkins (or the machine running the pipeline) must have AWS IAM credentials with permissions for ECR (push/pull images), EKS (deploy/manage resources), and other necessary AWS services.
- **Jenkins Credentials**: Store AWS credentials, ECR login credentials, and Git credentials securely in Jenkins (as credentials IDs referenced in the Jenkinsfile).
- **Docker**: Docker must be installed in the Jenkins container to build and push images.
- **kubectl**: The Kubernetes CLI (`kubectl`) must be installed and configured to access your EKS cluster from the Jenkins agent.
- **envsubst**: The `envsubst` utility (part of the GNU gettext package) must be installed on the Jenkins agent. This is required for processing Kubernetes manifests with environment variables before applying them with `kubectl`. Install with `apt-get install -y gettext` (Debian/Ubuntu) or `apk add --no-cache gettext` (Alpine).
- **AWS CLI**: The AWS CLI must be installed and configured on the Jenkins agent.

### Installing and Configuring AWS CLI in the Jenkins Container

#### 1. Install AWS CLI

Run these commands inside your Jenkins container (as root or with sudo):

```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
aws --version
```

- If unzip is not installed, install it first: `apt-get update && apt-get install -y unzip` (Debian/Ubuntu) or `yum install -y unzip` (RedHat/CentOS).

#### 2. Configure AWS CLI

You need to provide AWS credentials to the CLI. There are two main ways:

**A. Using aws configure interactively (not recommended for automation):**
```sh
aws configure
```
You will be prompted for:
- AWS Access Key ID
- AWS Secret Access Key
- Default region name (e.g., eu-west-2)
- Default output format (e.g., json)

**B. By mounting or copying your AWS credentials/config files:**
- Mount your local `~/.aws` directory to `/var/jenkins_home/.aws` in the container (recommended for automation):

  ```sh
  docker run -v ~/.aws:/var/jenkins_home/.aws ... jenkins/jenkins:lts
  ```

- Or copy your credentials into the container:

  ```sh
  docker cp ~/.aws <jenkins-container-name-or-id>:/var/jenkins_home/.aws
  ```

**C. By setting environment variables (for pipelines or container):**
```sh
export AWS_ACCESS_KEY_ID=your-access-key-id
export AWS_SECRET_ACCESS_KEY=your-secret-access-key
export AWS_DEFAULT_REGION=eu-west-2
```

- After setup, test with: `aws sts get-caller-identity`
- The AWS CLI must be able to access your credentials to interact with ECR/EKS.
- **Maven**: Maven must be installed and available on the Jenkins agent.
- **Kubernetes Manifests**: Ensure `kubernetes/deployment.yaml` and `kubernetes/service.yaml` are present and correctly reference your ECR image and application settings.
- **GitHub Personal Access Token (PAT)**: For pushing changes to GitHub, you must create a Personal Access Token (PAT) and store it in Jenkins credentials (see below).

### Accessing the Jenkins Environment

#### 1. SSH into Your Jenkins Host (if running on a VM or server)
Use SSH to connect to your Jenkins host:
```sh
ssh <username>@<jenkins-server-ip>
```
Replace `<username>` and `<jenkins-server-ip>` with your actual SSH username and server IP address.

#### 2. Access the Jenkins Container in Interactive Mode (if running Jenkins in Docker)
First, find the Jenkins container ID or name:
```sh
docker ps
```
Then, start an interactive shell session inside the container:
```sh
docker exec -it <jenkins-container-name-or-id> /bin/bash
```
or, if the container uses sh:
```sh
docker exec -it <jenkins-container-name-or-id> /bin/sh
```
This allows you to run commands, install tools, and troubleshoot directly inside the Jenkins container.

### Installing aws-iam-authenticator in the Jenkins Container
To enable EKS authentication, you must install `aws-iam-authenticator` in your Jenkins container:

```sh
curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/aws-iam-authenticator
chmod +x aws-iam-authenticator
mv aws-iam-authenticator /usr/local/bin/aws-iam-authenticator
aws-iam-authenticator help
```
- You may need to run these commands as root (use `sudo` if needed).
- After installation, verify with `aws-iam-authenticator help`.

### Installing kubectl in the Jenkins Container
To install the latest stable version of kubectl inside your Jenkins container or agent, run:

```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

- You may need to run these commands as root (use `sudo` if needed).
- After installation, verify with `kubectl version --client`.

### Copying kubeconfig from Host to Jenkins Container

To make your kubeconfig file available to kubectl inside the Jenkins container:

1. **Create the .kube directory inside the Jenkins container (if it doesn't exist):**
   ```sh
   docker exec -it <jenkins-container-name-or-id> mkdir -p /var/jenkins_home/.kube
   ```

2. **Copy the kubeconfig file from the host to the Jenkins container:**
   ```sh
   docker cp /home/ec2-user/config <jenkins-container-name-or-id>:/var/jenkins_home/.kube/config
   ```

3. **Set the correct permissions (optional but recommended):**
   ```sh
   docker exec -it <jenkins-container-name-or-id> chown jenkins:jenkins /var/jenkins_home/.kube/config
   ```

Replace `<jenkins-container-name-or-id>` with your Jenkins container's name or ID.

After these steps, kubectl in your Jenkins container will use the correct kubeconfig for EKS access.


### Manual Step: Create ECR Repository and Update Jenkinsfile
1. In your AWS Console, go to the ECR (Elastic Container Registry) service.
2. Create a new repository and name it `java-maven-app`.
3. After creation, copy the repository URI (it will look like `<aws-account-id>.dkr.ecr.<region>.amazonaws.com/java-maven-app`).
4. Open your Jenkinsfile and update the value of `DOCKER_REPO_SERVER` in the environment section to match the ECR URL (excluding `/java-maven-app`).
   - Example:
     ```groovy
     environment {
         DOCKER_REPO_SERVER = '<aws-account-id>.dkr.ecr.<region>.amazonaws.com'
         DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
     }
     ```
5. Save the Jenkinsfile.

### Setting up GitHub Credentials in Jenkins
1. **Create a Personal Access Token (PAT) on GitHub:**
   - Go to GitHub > Settings > Developer settings > Personal access tokens.
   - Generate a new token with the appropriate scopes (at least `repo` for private repos, or `public_repo` for public repos).
2. **Store the Token in Jenkins Credentials:**
   - Go to Jenkins > Credentials.
   - Add a new “Username with password” credential:
     - Username: your GitHub username
     - Password: your GitHub PAT
   - Use the ID `github-credentials` to match the Jenkinsfile reference.

### Creating the ECR Registry Key for Kubernetes
To allow your EKS cluster to pull images from AWS ECR, create a Kubernetes secret of type docker-registry:

```sh
kubectl create secret docker-registry aws-registry-key \
  --docker-server=<your-aws-account-id>.dkr.ecr.<your-region>.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region <your-region>)" \
  --docker-email=<your-email>
```
- Replace `<your-aws-account-id>`, `<your-region>`, and `<your-email>` with your actual values.
- This secret should match the name referenced in your deployment.yaml under `imagePullSecrets`.

**Note:**
The value for `imagePullSecrets.name` in your `kubernetes/deployment.yaml` must match the name of the secret you create (e.g., `aws-registry-key`):

```yaml
imagePullSecrets:
  - name: aws-registry-key
```
If you use a different name for your secret, update `deployment.yaml` accordingly.

### Kubernetes Contexts and Target Cluster
- Your kubeconfig file can contain multiple Kubernetes cluster contexts.
- The context that is currently active (set as `current-context`) determines which cluster kubectl (and thus Jenkins) will interact with.
- To list all available contexts and see which is active, run:
  - `kubectl config get-contexts`
- To set the context explicitly (recommended in CI/CD pipelines), use:
  - `kubectl config use-context <context-name>`
- In this project, the Jenkins pipeline sets the context to `Glen@java-maven-app-eks.eu-west-2.eksctl.io` before deploying. This ensures deployments always target the intended EKS cluster, even if your kubeconfig contains multiple clusters.
- **Reason:** Explicitly setting the context avoids accidental deployments to the wrong cluster and makes your pipeline predictable and safe.

### Removing the EKS Cluster
To delete the EKS cluster created with `eksctl-cluster.yaml`, run:
```
eksctl delete cluster -f eksctl-cluster.yaml
```
This command will remove the EKS cluster and all associated resources defined in the configuration file.

## CI/CD Pipeline Workflow

### 1. Build the Application
- Jenkins checks out the latest code from the repository.
- Maven is used to build and package the Java application.

### 2. Create a Docker Image
- Jenkins uses a Dockerfile to build a Docker image containing the application and its dependencies.
- The image is tagged with the application version and build number.

### 3. Push the Image to AWS ECR
- Jenkins authenticates to AWS ECR using credentials.
- The Docker image is pushed to a private ECR repository, making it available for deployment.

### 4. Deploy to AWS EKS
- Jenkins uses `kubectl` to apply Kubernetes manifests (deployment and service YAML files) to the EKS cluster.
- The deployment manifest references the Docker image in ECR, ensuring the latest version is deployed.
- The application is updated or rolled out in the EKS cluster automatically.

## Jenkinsfile Pipeline Stages
- **Increment Version:** Updates the application version for each build.
- **Build App:** Compiles and packages the Java application using Maven.
- **Build Image:** Builds a Docker image and pushes it to AWS ECR.
- **Deploy:** Deploys the new Docker image to the EKS cluster using Kubernetes manifests. The pipeline explicitly sets the context to the correct EKS cluster before deploying.
- **Commit Version Update:** Optionally commits the new version back to the source repository using GitHub credentials and a Personal Access Token.

### How Automated Version Bumping Works

The pipeline includes an **increment version** stage that automatically bumps the application version for every build. This is achieved using the Maven `build-helper` and `versions` plugins:

1. **Parsing and Incrementing the Version:**
   - The pipeline runs:
     ```sh
     mvn build-helper:parse-version versions:set -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} versions:commit
     ```
   - This command parses the current version in `pom.xml` and increments the patch (or minor/major, as configured) version for the new build.

2. **Why Version Bumping is Important:**
   - **Traceability:** Each build produces a unique version, making it easy to track which code changes are deployed.
   - **Reproducibility:** Artifacts (Docker images, JARs) are tagged with the version, ensuring you can always deploy or roll back to a specific build.
   - **Automation:** Automating version management reduces manual errors and enforces consistent versioning across builds.
   - **CI/CD Best Practice:** Automated versioning is a standard practice in modern CI/CD pipelines for reliable artifact management.

3. **Committing the Version Update:**
   - After the version is bumped, the pipeline commits the updated `pom.xml` (and any other changed files) back to the repository using a dedicated commit and push step.
   - This ensures the repository always reflects the latest version used in the build and deployment process.

**Summary:**
Automated version bumping and committing in the pipeline ensures every build is uniquely identifiable, traceable, and reproducible, supporting robust DevOps and release management practices.

## Kubernetes Deployment
- The `kubernetes/deployment.yaml` manifest defines how the application is deployed in the EKS cluster, referencing the Docker image in ECR.
- The `kubernetes/service.yaml` manifest exposes the application within the cluster.

## Creating a kubeconfig File for Jenkins (EKS)

To allow Jenkins to deploy to your EKS cluster, you need a valid kubeconfig file in the Jenkins container. You can use the provided `config.yaml` template and follow these steps:

### 1. Extract Required Values from AWS EKS

- **Cluster Name:**
  - The name you gave your EKS cluster (e.g., `java-maven-app-eks`).
  - List clusters: `aws eks list-clusters --region <your-region>`

- **Endpoint URL and Certificate Data:**
  - Run: `aws eks describe-cluster --name <cluster-name> --region <region>`
  - In the output, find:
    - `endpoint`: Use as `<endpoint-url>`
    - `certificateAuthority.data`: Use as `<certificate-data>` (base64 string)

### 2. Fill in the Template

- Open `config.yaml` and replace:
  - `<certificate-data>` with the value from AWS
  - `<endpoint-url>` with the value from AWS
  - `<cluster-name>` with your EKS cluster name

### 3. Copy the File into the Jenkins Container

- If your Jenkins container is named `jenkins`, copy the file:
  ```sh
  docker cp config.yaml jenkins:/var/jenkins_home/.kube/config
  ```
  (Create the `.kube` directory first if it doesn't exist.)

### 4. Set Permissions and Ownership

- Enter the Jenkins container as root:
  ```sh
  docker exec -it jenkins bash
  ```
- Run:
  ```sh
  mkdir -p /var/jenkins_home/.kube
  mv /path/to/config.yaml /var/jenkins_home/.kube/config
  chown jenkins:jenkins /var/jenkins_home/.kube/config
  chmod 600 /var/jenkins_home/.kube/config
  ```

After this, Jenkins will be able to authenticate to your EKS cluster using the kubeconfig file.

## Summary
This setup ensures that every code change is automatically built, containerized, stored in a secure image repository, and deployed to a scalable Kubernetes environment on AWS, following best practices for modern DevOps and cloud-native applications.
