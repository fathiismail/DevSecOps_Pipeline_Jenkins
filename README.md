# DevSecOps Pipeline for Secure Software Development with Jenkins

This repository provides a comprehensive guide to implementing a DevSecOps pipeline for secure software development. The guide covers all stages of the pipeline, including the installation of essential tools, configuration of the environment, integration of security checks SAST/DAST , SCA and DefectDojo as vulnerability management system , and setting up a CI/CD pipeline with Jenkins.

## 1. Setting Up the Environment

### Step 1: Install Ubuntu Server on a Virtual Machine

1. **Download Ubuntu Server:** [Ubuntu Server Download](https://ubuntu.com/download/server)
2. **Download VMware Workstation:** [VMware Workstation Download](https://www.vmware.com/content/vmware/vmware-published-sites/us/products/workstation-pro/workstation-pro-evaluation.html.html)
3. **Install Ubuntu Server:** Create a new VM, power on the machine, and complete the installation.

4. **Install essential packages:**

```bash
sudo apt update
sudo apt install net-tools nano apt-transport-https ca-certificates curl gnupg-agent software-properties-common -y
```

5. **Install Docker:**

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
sudo apt install docker-compose -y
sudo apt install python3 python3-pip -y
```

### Step 2: Install Jenkins and DefectDojo

1. **Jenkins Installation:**

```bash
sudo apt install fontconfig openjdk-17-jre
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

2. **Change Jenkins Port:** Edit these files to change port 8080:

```bash
sudo nano /etc/default/jenkins
nano /lib/systemd/system/jenkins.service
sudo systemctl daemon-reload
systemctl restart jenkins
```

3. **DefectDojo Installation:**

```bash
git clone https://github.com/DefectDojo/django-DefectDojo.git
cd django-DefectDojo/
sudo ./dc-build.sh
sudo ./dc-up-d.sh postgres-redis
docker compose logs initializer | grep "Admin password:"
```

   - Log into DefectDojo using the admin password, and go to **/api/key-v2** to get the API key.

   - Add the key to Jenkins:

      - **Jenkins → Manage Jenkins → Manage Credentials:** Add **Secret Text** with the API key, ID, and description as "defect-dojo-api-key".

## 2. Configuring Jenkins

1. **Initial Setup:**

   - Use the password obtained from **/var/lib/jenkins/secrets/initialAdminPassword** to access Jenkins and install suggested plugins.

2. **Plugins Installation:**

   - Navigate to **Manage Jenkins → Plugins → Available Plugins** and install:

      1. Eclipse Temurin Installer
      2. SonarQube Scanner
      3. NodeJs Plugin
      4. Email Extension Plugin

3. **Tool Configuration:**

   - Navigate to **Manage Jenkins → Tools**:

      - **JDK Installation:** Add JDK 17 as "jdk17".
      - **Node.js Installation:** Add Node 16 as "node16".

4. **SonarQube Configuration:**

   - Create a token in SonarQube and add it to Jenkins:

      - **Jenkins → Manage Jenkins → Credentials → system → Global credentials (unrestricted):**

      - **New credentials:** Add **Secret Text** with the token and ID as "sonar-token".

   - **SonarQube Server:** 

      - Go to **Jenkins → Manage Jenkins → System → SonarQube Servers:**

      - Add "sonar-server," enter your SonarQube URL, and select "sonar-token".

5. **SonarQube Scanner Installation:**

   - Go to **Jenkins → Manage Jenkins → Tools → SonarQube Scanner:** Add "sonar-scanner".

6. **Dependency-Check and Docker Tools:**

   - **Dependency-Check Installation:**

      - Go to **Jenkins → Manage Jenkins → Tools → Dependency-Check installations:**

      - **Add "DP-Check":** Choose install automatically, install from GitHub, and keep the latest version.

   - **Docker Installation:**

      - Go to the **Docker installation section**, add "Docker," choose install automatically, and download from [docker.com](http://docker.com). Keep the version latest.

7. **DefectDojo Configuration:**

   - Log into DefectDojo, and obtain an API key:

      - **Jenkins → Manage Jenkins → Manage Credentials:** Add **Secret Text** with ID "defect-dojo-api-key".

   - **Configure a project in SonarQube:**

      - Create a project and name it "Netflix."

   - **DefectDojo Setup:**

      - Add a new product in DefectDojo, name it "Netflix," add a description, and choose "Research and Development." Click Save.

      - Go to **Engagement**, click "Add New CI/CD Engagement," and name it as desired, e.g., "Jenkins CI/CD." Choose Netflix as the product and click Done.

8. **Install OWASP ZAP Proxy:**

```bash
wget https://github.com/zaproxy/zaproxy/releases/download/v2.14.0/ZAP_2.14.0_Linux.tar.gz
sudo tar -xzvf ZAP_2.14.0_Linux.tar.gz -C /usr/share/
sudo chmod +x /usr/share/owasp-zap/ZAP_2.14.0/zap.sh
```

9. **Install Sonar-Report:**

```bash
apt install npm
npm install -g sonar-report
```

10. **Install Blue Ocean Plugins:**

   - Follow the same process for plugins in Jenkins, and search for **Blue Ocean**. This tool provides a clear design for CI/CD stages.

## 3. CI/CD Pipeline Setup

1. **Create a new job in Jenkins:**

   - Choose "Pipeline" and enter the following code:

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'your-docker-image:latest'
        APP_URL = 'http://your-app-url:8081'
        DD_API_KEY = credentials('defect-dojo-api-key')
        DD_URL = 'http://your-defectdojo-url/api/v2/import-scan/'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/fathiismail/DevSecOps'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonar-server') {
                        sh """
                            ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectName=Netflix \
                            -Dsonar.projectKey=Netflix \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs --format json -o trivyfs-report.json ."
            }
        }
        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3 API_KEY=your-api-key -t netflix ."
                        sh "docker tag netflix your-docker-image:latest"
                        sh "docker push your-docker-image:latest"
                    }
                }
            }
        }
        stage("TRIVY") {
            steps {
                sh "trivy image --format json -o trivyimage-report.json ${DOCKER IMAGE}"
            }
        }
        stage('Deploy to Container') {
            steps {
                sh 'docker run -d -p 8081:80 ${DOCKER_IMAGE}'
            }
        }
        stage('OWASP ZAP Scan') {
            steps {
                sh "/usr/share/ZAP_2.14.0/zap.sh -cmd

 -quickurl ${APP_URL} -quickprogress -quickout ~/report.xml -port 8090"
            }
        }
        stage('Send Sonarqube to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonar-server') {
                        sh """
                            sonar-report --sonarurl="http://your-sonarqube-url/" --sonartoken ${SONAR TOKEN} --sonarcomponent="Netflix" --sonarorganization="Netflix" --project="Netflix" --application="Netflix" --release="1.0.0" --branch="main" --output="sonarreport.html"
                        """
                    }
                }
            }
        }
        stage('Upload to DefectDojo') {
            steps {
                script {
                    // Upload various reports to DefectDojo
                    def dojoImportScanURL = 'http://your-defectdojo-url/api/v2/import-scan/'
                    def dojoEngagementID = "your-engagement-id"

                    sh """
                        curl -X POST "${dojoImportScanURL}" \
                            -H 'Authorization: Token ${DD API_KEY}' \
                            -F 'engagement=${dojoEngagementID}' \
                            -F 'scan_type=ZAP Scan' \
                            -F 'file=@${zapReportPath}' \
                            -F 'close_old_findings=true' \
                            -F 'active=true' \
                            -F 'verified=true'
                    """
                    sh """
                        curl -X POST "${dojoImportScanURL}" \
                            -H 'Authorization: Token ${DD_API_KEY}' \
                            -F 'engagement=${dojoEngagementID}' \
                            -F 'scan_type=Trivy Scan' \
                            -F 'file=@${trivyFsReportPath}' \
                            -F 'close_old findings=true' \
                            -F 'active=true'
                    """
                    sh """
                        curl -X POST "${dojoImportScanURL}" \
                            -H 'Authorization: Token ${DD API_KEY}' \
                            -F 'engagement=${dojoEngagementID}' \
                            -F 'scan_type=SonarQube Scan' \
                            -F 'file=@${sonarscanreport}' \
                            -F 'close_old findings=true' \
                            -F 'active=true'
                    """
                    sh """
                        curl -X POST "${dojoImportScanURL}" \
                            -H 'Authorization: Token ${DD_API_KEY}' \
                            -F 'engagement=${dojoEngagementID}' \
                            -F 'scan_type=Dependency Check Scan' \
                            -F 'file=@${dependencyCheckReportPath}' \
                            -F 'close_old_findings=true' \
                            -F 'active=true' \
                            -F 'verified=true'
                    """
                }
            }
        }
    }
}
```
