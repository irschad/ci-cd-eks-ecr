# Complete CI/CD Pipeline with EKS and AWS ECR

## Overview
This project demonstrates the implementation of a complete CI/CD pipeline integrating Jenkins, Kubernetes, AWS EKS (Elastic Kubernetes Service), and AWS ECR (Elastic Container Registry). The pipeline automates the entire process of versioning, building, deploying, and updating a Java-based application using Docker containers.

## Technologies Used
- **Kubernetes**
- **Jenkins**
- **AWS EKS**
- **AWS ECR**
- **Java**
- **Maven**
- **Linux**
- **Docker**
- **Git**

## Project Workflow
1. **Create a private AWS ECR Docker repository**
2. **Set up Jenkins credentials for ECR**
3. **Adjust the Jenkinsfile to build and push Docker images to AWS ECR**
4. **Integrate deployment to the EKS cluster**
5. **Update the application version and commit changes**

## CI/CD Pipeline Stages
1. **CI Step:** Increment version
2. **CI Step:** Build artifact for the Java Maven application
3. **CI Step:** Build and push Docker image to AWS ECR
4. **CD Step:** Deploy the new application version to the EKS cluster
5. **CD Step:** Commit the version update

## Detailed Steps

### Step 1: Create a Private AWS ECR Docker Repository
1. Navigate to the AWS Management Console.
2. Open the **ECR Dashboard** (Services > Containers > Elastic Container Registry).
3. Click **Create Repository** and specify the repository name (e.g., `java-maven-app`).
4. Ensure the visibility is set to **Private** and click **Create Repository**.

### Step 2: Authenticate AWS ECR Using `aws ecr get-login-password`
To push or pull Docker images, you need to authenticate with AWS ECR. Use the `aws ecr get-login-password` command as follows:

```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com
```

#### Explanation:
- `aws ecr get-login-password` retrieves the authentication token for ECR.
- The token is piped to the `docker login` command to authenticate the Docker client.
- Replace `<region>` with your AWS region (e.g., `us-east-1`) and `<account_id>` with your AWS account ID.

This command eliminates the need to manually retrieve and manage passwords for ECR authentication.

### Step 3: Set Up Jenkins Credentials for ECR
1. Navigate to your Jenkins dashboard.
2. Go to **Manage Jenkins > Manage Credentials > System > Global Credentials (unrestricted)**.
3. Click **Add Credentials**.
4. Select **Username with Password** as the kind.
   - **Username:** `AWS`
   - **Password:** Use the output of `aws ecr get-login-password`.
   - **ID:** Provide a descriptive ID (e.g., `ecr-credentials`).
5. Click **Create**.

### Step 4: Create a Kubernetes Secret for AWS ECR
Kubernetes requires a Docker registry secret to pull images from ECR. Use the following command:

```bash
kubectl create secret docker-registry aws-registry-key \
  --docker-server=<account_id>.dkr.ecr.<region>.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region <region>)
```

Verify the secret creation:

```bash
kubectl get secrets
```

Update your Kubernetes `deployment.yaml` to reference this secret:

```yaml
imagePullSecrets:
  - name: aws-registry-key
```

### Step 5: Adjust the Jenkinsfile
The Jenkinsfile orchestrates the CI/CD pipeline. Key configurations:

#### Environment Variables
```groovy
environment {
    DOCKER_REPO_SERVER = '<account_id>.dkr.ecr.<region>.amazonaws.com'
    DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
}
```

#### Build and Push Docker Image
```groovy
stage('build image') {
    steps {
        script {
            echo "Building and pushing Docker image..."
            withCredentials([usernamePassword(credentialsId: 'ecr-credentials', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                sh "docker build -t ${DOCKER_REPO}:${IMAGE_NAME} ."
                sh 'echo $PASS | docker login -u $USER --password-stdin ${DOCKER_REPO_SERVER}'
                sh "docker push ${DOCKER_REPO}:${IMAGE_NAME}"
            }
        }
    }
}
```

### Step 6: Deploy to Kubernetes
Use `kubectl apply` commands within the `deploy` stage to update the Kubernetes deployment and service configurations. Example:

```bash
sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
```

### Step 7: Commit Version Update
Automate version updates in Git by leveraging the `commit version update` stage in the Jenkinsfile. This stage ensures that changes to the application version are consistently pushed to the Git repository.

#### Key Steps in the Jenkinsfile
```groovy
stage('commit version update') {
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'jenkinspush', passwordVariable: 'PAT', usernameVariable: 'USER')]) {
                sh 'git remote set-head origin master'
                sh 'git config --global user.email "jenkins@example.com"'
                sh 'git config --global user.name "jenkins"'
                sh 'git add .'
                sh "git remote set-url origin https://${PAT}@github.com/irschad/ci-cd-eks-ecr.git"
                sh "git commit -m 'ci: version bump'"
                sh 'git push origin HEAD:master'
            }
        }
    }
}
```

#### Explanation
1. **Git Configuration:**
   - The global user email and name are set for Jenkins to identify commits.
2. **Staging Changes:**
   - The `git add .` command stages all changes in the working directory.
3. **Commit and Push:**
   - Commits are made with a clear message indicating a version bump.
   - The `git push` command updates the master branch with the new version.

```

## Execute Jenkins Pipeline and verify deployment
1. Commit the updated code to the GitHub repository.
2. Ensure no conflicting pods are running in the cluster:

```bash
kubectl get pods
```

3. Trigger the Jenkins pipeline build.
4. Verify the ECR repository for the new image and tag.
5. Check the Kubernetes cluster:

```bash
kubectl get pods
kubectl describe pod <pod_name>
```

Confirm the image URL matches the expected ECR repository.


