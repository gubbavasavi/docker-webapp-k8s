# ** DevOps K8S Project – End-to-End CI/CD Pipeline**

## **1. Project Overview**

This project demonstrates an end-to-end DevOps implementation using Jenkins, Docker, Kubernetes, and AWS.

The project automates application building, testing, code quality analysis, artifact management, containerization, security scanning, and deployment.

## **2. Technology Stack**

| Technology | Purpose |
|---|---|
| AWS EC2 | Cloud Infrastructure |
| Git & GitHub | Source Code Management |
| Jenkins | CI/CD Automation |
| Maven | Application Build |
| SonarQube | Code Quality Analysis |
| Nexus | Artifact Repository |
| Docker | Containerization |
| Docker Compose | Multi-container Deployment |
| Trivy | Container Image Scanning |
| Kubernetes (kOps) | Container Orchestration |
| Argo CD | GitOps-based Continuous Deployment |
| Helm | Kubernetes Package Manager |
| Prometheus | Monitoring and Metrics Collection |
| Grafana | Monitoring Dashboards |
| Metrics Server | Kubernetes Resource Metrics |

## **3. Project Workflow**

1. Developers push application code to GitHub.
2. Jenkins fetches the source code from the GitHub repository.
3. Maven builds the application and generates the WAR artifact.
4. SonarQube performs code quality analysis.
5. SonarQube Quality Gates validate the code quality.
6. Nexus stores the generated WAR artifact.
7. Jenkins builds Docker images for the application and database.
8. Trivy scans Docker images for vulnerabilities.
9. Docker images are pushed to Docker Hub.
10. Docker Compose deploys the application and database containers.
11. Kubernetes manifests are maintained in GitHub.
12. Argo CD synchronizes the manifests and deploys the application into Kubernetes.
13. Prometheus and Grafana monitor cluster and application health.

---

## **4. Jenkins CI/CD Pipeline**

Jenkins is used to automate the complete Continuous Integration process and application deployment.

The Jenkins pipeline performs the following activities:

- **Git Checkout:** Retrieves application source code from GitHub.
- **Maven Build:** Compiles the application and generates the WAR file.
- **SonarQube Analysis:** Performs static code analysis.
- **Quality Gate:** Checks whether the code meets the defined quality standards.
- **Nexus Upload:** Uploads the generated WAR artifact to Nexus Repository Manager.
- **Docker Build:** Builds application and database Docker images.
- **Trivy Scan:** Scans Docker images for security vulnerabilities.
- **Docker Hub Push:** Pushes the Docker images to Docker Hub.
- **Docker Compose Deployment:** Deploys the application using Docker Compose.

### **Jenkinsfile**

```groovy
pipeline {
    agent any

    tools {
        maven "mymaven"
    }

    stages {

        stage('Getting code from git') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/gubbavasavi/docker-webapp-k8s.git'
            }
        }

        stage('build code using maven') {
            steps {
                sh 'mvn clean install'
            }
        }

        stage('QCA by sonarqube') {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh "mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=myproject -Dsonar.projectName='myproject'"
                }
            }
        }

        stage('Quality gates by Sonarqube') {
            steps {
                waitForQualityGate abortPipeline: true,
                    credentialsId: 'sonarcred'
            }
        }

        stage('Artifact Upload by Nexus') {
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'

                    nexusArtifactUploader(
                        artifacts: [[
                            artifactId: pom.artifactId,
                            classifier: '',
                            file: "target/${pom.artifactId}-${pom.version}.war",
                            type: 'war'
                        ]],
                        credentialsId: 'nexuscred',
                        groupId: pom.groupId,
                        nexusUrl: '3.110.80.236:8081',
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        repository: 'mymavenrepo',
                        version: pom.version
                    )
                }
            }
        }

        stage('Building Docker Images') {
            steps {
                sh 'docker build -t vasavigubba/k8s-javaproject:app-image .'
                sh 'docker build -t vasavigubba/k8s-javaproject:db-image Docker-db'
            }
        }

        stage('Scan Images with Trivy') {
            steps {
                sh 'trivy image vasavigubba/k8s-javaproject:app-image >> app-image-logs.txt'
                sh 'trivy image vasavigubba/k8s-javaproject:db-image >> db-image-logs.txt'
            }
        }

        stage('Pushing Images to Docker Hub') {
            steps {
                withDockerRegistry(credentialsId: 'dockercred') {
                    sh 'docker push vasavigubba/k8s-javaproject:app-image'
                    sh 'docker push vasavigubba/k8s-javaproject:db-image'
                }
            }
        }

        stage('Deploy using Compose') {
            steps {
                sh 'docker compose up -d'
            }
        }
    }
}
```

### **Pipeline Stage Explanation**

| Stage | Description |
|---|---|
| Getting code from Git | Clones the application repository from GitHub |
| Maven Build | Builds the Java application and generates the WAR artifact |
| SonarQube Analysis | Performs code quality analysis |
| Quality Gate | Validates the SonarQube quality gate status |
| Nexus Upload | Uploads the WAR artifact to Nexus |
| Building Docker Images | Builds application and database images |
| Trivy Scan | Scans Docker images for vulnerabilities |
| Docker Hub Push | Publishes Docker images to Docker Hub |
| Deploy using Compose | Starts application and database containers |

---

## **5. Deployment – Argo CD (Continuous Deployment)**

- Argo CD is installed in the `argocd` namespace on the kOps cluster.
- The `argocd-server` service is exposed using a LoadBalancer.
- Jenkins publishes the following Docker images to Docker Hub:
  - `vasavigubba/k8s-javaproject:app-image`
  - `vasavigubba/k8s-javaproject:db-image`
- Kubernetes manifests reference these Docker images.
- Argo CD synchronizes Kubernetes manifests from GitHub and deploys the application into the `vprofile` namespace.
- Changes committed to Kubernetes manifests in GitHub can automatically be reflected in the cluster through GitOps synchronization.

## **6. Monitoring**

- Metrics Server is installed for Kubernetes resource monitoring using `kubectl top`.
- The `kube-prometheus-stack` is installed using Helm.
- The stack includes:
  - **Prometheus:** Collects metrics.
  - **Grafana:** Visualizes metrics through dashboards.
  - **Alertmanager:** Manages alerts.
- Grafana and Prometheus services are exposed using LoadBalancer services.
- Grafana dashboards are used to monitor cluster and application health and configure alerts.

---

# **7. Setup Guide**

## **Step 1: Jenkins + Docker + Git (Amazon Linux)**

**Set the hostname:**

```bash
hostnamectl set-hostname Jenkins-Sonar
```

**Install Java 21:**

```bash
yum install java-21-amazon-corretto -y
```

**Install Jenkins:**

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/rpm-stable/jenkins.repo

yum install jenkins -y

systemctl enable --now jenkins
```

**Install Docker:**

```bash
yum install docker -y

systemctl enable --now docker
```

**Install Git:**

```bash
yum install git -y
```

**Verify Jenkins and Docker services:**

```bash
systemctl status jenkins
systemctl status docker
```

**Get the initial Jenkins admin password:**

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Access Jenkins:**

`http://<server-ip>:8080`

## **Step 2: SonarQube (Docker)**

**Run SonarQube using Docker:**

```bash
docker run -d --name sonar -p 9000:9000 sonarqube
```

**Verify the container:**

```bash
docker ps
```

**Access SonarQube:**

`http://<server-ip>:9000`

## **Step 3: Nexus Repository Manager**

Launch a separate EC2 instance (c7i) and install Nexus Repository Manager.

Create a repository to store the generated WAR artifacts and configure Nexus credentials in the Jenkins pipeline.

**Install Java 21:**

```bash
yum install java-21-amazon-corretto -y
```

**Download and extract Nexus:**

```bash
cd /opt

wget https://download.sonatype.com/nexus/3/nexus-3.96.0-09-linux-x86_64.tar.gz

tar -zxvf nexus-3.96.0-09-linux-x86_64.tar.gz
```

**Create Nexus user and assign permissions:**

```bash
useradd nexus

chown -R nexus:nexus nexus-3.96.0-09 sonatype-work
```

**Start Nexus:**

```bash
su - nexus

cd /opt/nexus-3.96.0-09/bin/

./nexus start
```

## **Step 4: Trivy – Docker Image Scanning**

**Install Trivy:**

```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin v0.76.2
```

**Scan a Docker image:**

```bash
trivy image <image-name>
```

## **Step 5: Docker Compose**

**Create Docker CLI plugins directory:**

```bash
sudo mkdir -p /usr/libexec/docker/cli-plugins
```

**Download Docker Compose:**

```bash
sudo curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

**Provide execution permission:**

```bash
sudo chmod +x /usr/local/bin/docker-compose
```

**Create symbolic link:**

```bash
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

**Verify Docker Compose installation:**

```bash
docker-compose version
```

## **Step 6: Install kubectl and kOps**

**Install kubectl:**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

mv kubectl /usr/local/bin/
```

**Install kOps:**

```bash
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64

chmod +x kops

sudo mv kops /usr/local/bin/kops
```

## **Step 7: Create Kubernetes Cluster Using kOps**

**Configure the kOps state store:**

```bash
export KOPS_STATE_STORE=s3://<your-bucket-name>
```

**Create the Kubernetes cluster:**

```bash
kops create cluster --name vasav.k8s.local \
  --zones us-east-2a,us-east-2b \
  --master-count 1 --master-size m7i-flex.large --master-volume-size 30 \
  --node-count 2 --node-size c7i-flex.large --node-volume-size 25 \
  --image=<ami-id>
```

**Apply the cluster configuration:**

```bash
kops update cluster --name vasavii.k8s.local --yes --admin
```

**Verify Kubernetes nodes:**

```bash
kubectl get nodes
```

## **Step 8: Install Argo CD**

**Create the Argo CD namespace:**

```bash
kubectl create namespace argocd
```

**Install Argo CD:**

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Verify Argo CD resources:**

```bash
kubectl get all -n argocd
```

**Expose Argo CD using a LoadBalancer:**

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

**Check the Argo CD service:**

```bash
kubectl get svc -n argocd
```

**Retrieve the initial admin password:**

Username: `admin`

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## **Step 9: Install Metrics Server**

**Install Metrics Server:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Verify resource metrics:**

```bash
kubectl top pods -n argocd
```

## **Step 10: Install Helm, Prometheus and Grafana**

**Install Helm:**

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```

**Verify Helm installation:**

```bash
helm version
```

**Install kube-prometheus-stack:**

```bash
helm install prometheus oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack
```

**Expose Grafana using a LoadBalancer:**

```bash
kubectl patch svc prometheus-grafana -p '{"spec":{"type":"LoadBalancer"}}'
```

**Expose Prometheus using a LoadBalancer:**

```bash
kubectl patch svc prometheus-kube-prometheus-prometheus -p '{"spec":{"type":"LoadBalancer"}}'
```

**Verify services:**

```bash
kubectl get svc
```

**Retrieve Grafana admin password:**

Username: `admin`

```bash
kubectl --namespace default get secrets prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d

echo
```

## **Step 11: Verify Application Deployment**

**Check application pods in the vprofile namespace:**

```bash
kubectl get pods -n vprofile
```

**Check all resources:**

```bash
kubectl get all -n vprofile
```

---

# **8. Jenkins Plugins Required**

Install the following plugins in Jenkins:

1. **Pipeline Stage View**
2. **SonarQube Scanner**
3. **Nexus Artifact Uploader**
4. **Pipeline Utility Steps**
5. **Docker Pipeline**

**Docker Socket Permission:**

The original setup uses the following command to allow Jenkins to access Docker:

```bash
chmod 777 /var/run/docker.sock
```

**Note:** This grants broad access to the Docker daemon and is not recommended for production. Prefer configuring appropriate group permissions for Jenkins.

# **9. Jenkins Credentials Configuration**

Create the following credentials in Jenkins:

| Credential | Type | Purpose |
|---|---|---|
| GitHub | Username/Password or token | Access private repositories |
| Nexus | Username and Password | Upload WAR artifacts |
| SonarQube | Secret Text | SonarQube authentication token |
| Docker Hub | Username and Password | Push Docker images |

**GitHub Credentials:**
- Required when accessing private repositories.
- Configure repository credentials in Jenkins.

**Nexus Credentials:**
- Add Nexus username and password.
- Use the credentials ID in the Jenkins pipeline.

**SonarQube Credentials:**
- Generate an authentication token in SonarQube.
- Add it in Jenkins as Secret Text.
- Configure the SonarQube server in Jenkins.

**Docker Hub Credentials:**
- Add Docker Hub username and password.
- Use the credentials ID when pushing Docker images.

---

# **10. Final Project Outcome**

This project demonstrates an automated DevOps workflow covering:

- **Continuous Integration:** Jenkins pipeline automation.
- **Code Quality:** SonarQube analysis.
- **Artifact Management:** Nexus Repository Manager.
- **Containerization:** Docker and Docker Compose.
- **Security Scanning:** Trivy image scanning.
- **Container Orchestration:** Kubernetes using kOps.
- **Continuous Deployment:** Argo CD and GitOps.
- **Monitoring:** Prometheus, Grafana and Alertmanager.

The implementation provides hands-on experience with modern DevOps tools and cloud-native application deployment.
