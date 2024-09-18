# DevSecOps NodeJS App Actions
## Deploy NodeJS Application on AWS using DevSecOps Approach using GitHub Actions

![Starbucks Clone Deployment (1)](https://github.com/user-attachments/assets/44140491-8c87-4b1d-ab90-d618cafeca38)

# **Installation of Tools:**
1. Install Java 17
2. Install Temurin (formerly Adoptium) JDK 17.
3. Install Trivy (Container Vulnerability Scanner).
4. Install Terraform.
5. Install kubectl (Kubernetes command-line tool).
6. Install AWS CLI (Amazon Web Services Command Line Interface).
7. Install Node.js 16 and npm.

The script automates the installation of these software tools commonly used for development and deployment.

```
#!/bin/bash
sudo apt update -y
sudo touch /etc/apt/keyrings/adoptium.asc
sudo wget -O /etc/apt/keyrings/adoptium.asc https://packages.adoptium.net/artifactory/api/gpg/key/public
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
# Install Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
# Install Terraform
sudo apt install wget -y
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
# Install kubectl
sudo apt update
sudo apt install curl -y
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install unzip -y
unzip awscliv2.zip
sudo ./aws/install
# Install Node.js 16 and npm
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource.gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/nodesource-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nodesource-archive-keyring.gpg] https://deb.nodesource.com/node_16.x focal main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update
sudo apt install -y nodejs
```


# **verfiy the Installation of Tools:**
```
trivy --version
terraform --version
aws --version
kubectl version
node -v
java --version
```

# **GitHub Actions Workflow YAML:**

```
name: Build,Analyze,scan
on:
  push:
    branches:
      - main
jobs:
  build-analyze-scan:
    name: Build
    runs-on: [self-hosted]
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0  # Shallow clones should be disabled for a better relevancy of analysis
      - name: Build and analyze with SonarQube
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      - name: npm install dependency
        run: npm install
      - name: Trivy file scan
        run: trivy fs . > trivyfs.txt
      - name: Docker Build and push
        run: |
          docker build -t tic-tac-toe .
          docker tag tic-tac-toe amonkincloud/node-app:latest
          docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
          docker push amonkincloud/node-app:latest
        env:
          DOCKER_CLI_ACI: 1
      - name: Image scan
        run: trivy image amonkincloud/node-app:latest > trivyimage.txt
  deploy:
   needs: build-analyze-scan
   runs-on: [self-hosted]
   steps:
      - name: docker pull image
        run: docker pull amonkincloud/node-app:latest
      - name: Image scan
        run: trivy image amonkincloud/node-app:latest > trivyimagedeploy.txt
      - name: Deploy to container
        run: docker run -d --name game -p 3000:3000 amonkincloud/node-app:latest
      - name: Update kubeconfig
        run: aws eks --region <update-aws-region> update-kubeconfig --name <update-eks-cluster-name>
      - name: Deploy to kubernetes
        run: kubectl apply -f deployment-service.yml
```


