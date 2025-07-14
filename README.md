# Swiggy-Clone-App


``` bash
#jenkins

sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins

#install docker

sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu  
newgrp docker
sudo chmod 777 /var/run/docker.sock
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

#install trivy

sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

```

``` jenkinsfile
pipeline {
    agent any
    parameters {
        booleanParam(name: 'SKIP_SCANS', defaultValue: false, description: 'Skip all scan stages (OWASP, Sonar QC)')
    }
    tools {
        jdk 'jdk17'
        nodejs 'nodejs16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/tusuii/Swiggy-Clone-App-eks-argocd.git'
            }
        }

        stage('sonarqube') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=sonar-token \
                        -Dsonar.projectKey=sonar-token
                    '''
                }
            }
        }

        stage('quality gates') {
            when {
                expression { return !params.SKIP_SCANS }
            }
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('npm dependency') {
            steps {
                sh "npm install"
            }
        }

        stage('OWASP') {
            when {
                expression { return !params.SKIP_SCANS }
            }
            steps {
                dependencyCheck additionalArguments: '--scan ./  --disableYarnAudit --disableNodeAudit', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('trivy fs scan') {
            steps {
                sh "trivy fs . > trivy-fs.txt"
            }
        }

        stage('docker build & docker push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh 'docker build -t subkamble/swiggy-app:latest .'
                    sh 'docker push subkamble/swiggy-app:latest'
                }
            }
        }

        stage('trivy image scan') {
            steps {
                sh "trivy image subkamble/swiggy-app:latest > trivy-img.json"
            }
        }

        stage('Cleanup old Docker container') {
            steps {
                script {
                    sh '''
                    docker rm -f swiggy-app || true
                    docker rmi subkamble/swiggy-app:latest || true
                    '''
                }
            }
        }

        stage('Pull Docker Image') {
            steps {
                script {
                    sh 'docker pull subkamble/swiggy-app:latest'
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                script {
                    sh 'docker run -d --name swiggy-app -p 3000:3000 subkamble/swiggy-app:latest'
                }
            }
        }

        stage('Show Running Containers') {
            steps {
                sh 'docker ps'
            }
        }
    }
}

```
``` bash
Asia/Kolkata
* * * * *
```

plugins:
1. eclipse.
2. sonarqube.
3. nodejs
4. stagebuild.
5. Docker.
6. owasp dependency check...


