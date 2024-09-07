Docker Hub - https://hub.docker.com/repository/docker/adithyarc26/vote/general

# Capstone1 deploy python app in minikube using Jenkins - DevSecOps Project!
### **Phase 1: Initial Setup and Deployment**
**Step 1: Launch EC2 (Ubuntu 22.04):**

- Provision an EC2 instance on AWS with Ubuntu 22.04.
- Connect to the instance using SSH.

**Step 2: Clone the Code:**

- Update all the packages and then clone the code.
- Clone your application's code repository onto the EC2 instance:
    
    ```bash
    git clone https://github.com/AdithyaRC2611/example-voting-app.git
    ```
    

**Step 3: Install Docker and Run the App Using a Container:**

- Set up Docker on the EC2 instance:
    
    ```bash
    
    sudo apt-get update
    sudo apt-get install docker.io -y
    sudo usermod -aG docker $USER  # Replace with your system's username, e.g., 'ubuntu'
    newgrp docker
    sudo chmod 777 /var/run/docker.sock
    ```
    
- Build and run your application using Docker containers:
    
    ```bash
    docker build -t vote .
    docker run -d --name vote -p 8081:80 vote:latest
    
    #to delete
    docker stop <containerid>
    docker rmi -f vote
    ```
**Step 3: Setup Minikube Kubernetes Cluster:**

    ```bash
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    minikube start
    kubectl config use-context minikube

    ```
**Phase 2: Security**
1. **Install SonarQube and Trivy:**
    - Install SonarQube and Trivy on the EC2 instance to scan code vulnerabilities.
        
        sonarqube
        ```
        docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
        ```
        
        
        To access: 
        
        publicIP:9000 (by default username & password is admin)
        
        To install Trivy:
        ```
        sudo apt-get install wget apt-transport-https gnupg lsb-release
        wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
        echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
        sudo apt-get update
        sudo apt-get install trivy        
        ```
        
        to scan image using trivy
        ```
        trivy image <imageid>
        ```
        
        
2. **Integrate SonarQube and Configure:**
    - Integrate SonarQube with your CI/CD pipeline.
    - Configure SonarQube to analyze code for quality and security issues.
  
**Phase 3: CI/CD Setup using Jenkins**
1. **Install Jenkins for Automation:**
    - Install Jenkins on the EC2 instance to automate deployment:
    Install Java
    
    ```bash
    sudo apt update
    sudo apt install fontconfig openjdk-17-jre
    java -version
    openjdk version "17.0.8" 2023-07-18
    OpenJDK Runtime Environment (build 17.0.8+7-Debian-1deb12u1)
    OpenJDK 64-Bit Server VM (build 17.0.8+7-Debian-1deb12u1, mixed mode, sharing)
    
    #jenkins
    sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
    https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
    echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null
    sudo apt-get update
    sudo apt-get install jenkins
    sudo systemctl start jenkins
    sudo systemctl enable jenkins
    ```
    
    - Access Jenkins in a web browser using the public IP of your EC2 instance.
        
        publicIp:8080
2. **Install Necessary Plugins in Jenkins:**
   1 SonarQube scanner
   2 Kubernetes plugin
   3 docker plugin
   4 Prometheus plugin

4. **Create Jenkins Pipeline:**
   ```groovy

pipeline{
    agent any
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/AdithyaRC2611/example-voting-app.git', credentialsId: 'github'
            }
        }
        stage('Verify Checkout'){
            steps{
                sh 'ls -la'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=vote1 \
                    -Dsonar.projectKey=vote1 '''
                }
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){  
                       sh "ls"
                       sh "docker build -t vote ."
                       sh "docker tag vote adithyarc26/vote:latest "
                       sh "docker push adithyarc26/vote:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image adithyarc26/vote:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -itd -p 8081:80 adithyarc26/vote:latest'
                sh 'kubectl apply -f K8s/deployment.yml'
                sh 'kubectl apply -f K8s/service.yml'
                sh 'kubectl apply -f K8s/ingress.yml'
            }
        }
    }
}


```
**Phase 4: Monitoring**
1. **Install Prometheus and Grafana**

Install Helm CLI
```bash
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
Add Helm Repositories
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
Install Prometheus
```bash
helm install prometheus prometheus-community/prometheus --namespace monitoring
```
Go to sudo /etc/systemd/Prometheus/prometheus.yml file
```bash
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["34.195.12.92:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["34.195.12.92:9100"]

  - job_name: "jenkins"
    metrics_path: "/prometheus"
    static_configs:
      - targets: ["34.195.12.92:8080"]
```

2. **Install Grafana and configure Prometheus as data source**
Step 1: Add Grafana Helm Repository
First, ensure that you have added the Grafana Helm repository and updated the repository list:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
Step 2: Install Grafana
Create a Namespace for Grafana (optional):
kubectl create namespace monitoring
```
Install Grafana using Helm:

```bash
helm install grafana grafana/grafana --namespace monitoring
```
This command installs Grafana in the monitoring namespace. If you prefer to use the default namespace, you can omit the --namespace flag.

Step 3: Access Grafana
Retrieve the Admin Password:

After installation, retrieve the Grafana admin password:

```bash
kubectl get secret --namespace monitoring grafana-admin-password -o go-template='{{ .data.password | base64decode }}'
Replace monitoring with your namespace if you used a different one.
```
Log in to Grafana:

Username: admin
Password: The password you retrieved in step 1.
Step 4: Configure Grafana
Add Prometheus as a Data Source:

If you have Prometheus running, add it as a data source in Grafana:

Log in to Grafana.
Go to Configuration (gear icon) > Data Sources.
Click Add data source.
Select Prometheus.
Set the URL to http://prometheus-server:80 (or the appropriate service name and port for your Prometheus installation).
Click Save & Test.

