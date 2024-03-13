# Netlix-clone-cicd

AWS Instance should be t2.xlarge 

we will perform this project on ubuntu 22

--To Install jenkins in ubuntu 22--

sudo apt update

sudo apt install fontconfig openjdk-17-jre

java -version

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian/jenkins.io-2023.key

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update

sudo apt-get install jenkins

--To Install Docker in ubuntu 22--

sudo apt install docker.io -y

sudo usermod -aG docker ubuntu

newgrp docker

sudo chmod 777 /var/run/docker.sock

sudo systemctl restart docker

sudo systemctl   status  docker

--To Install Trivy in ubuntu 22

sudo apt-get install wget apt-transport-https gnupg lsb-release

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -

echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list

sudo apt-get update

sudo apt-get install trivy  

trivy --version

--Installing Sonarqube as Docker Container--

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

------------------------------------------------------------------------

Now in jenkins we will Install Plugins of Sonarqube Owasp Node Docker 'Eclipse Temurin Installer' & Kubernetes
we have to go to manage jenkins > plugins > available plugins

Then we come to manage Jenkins > tools

add jdk [ jdk17 ] install from adoptium.net  and version jdk-17.0.8.1+1

add sonarQube scanner  [ sonar-scanner ] 

add Node [ Node16 ] version NodeJs 16.2.0

add Dependency Check [DP-Check] install from github.com verion dependency check 6.5.1

add Docker [Docker] 

we can come to this documentation too https://docs.google.com/document/d/1NNU3ONLqq--FcL4Xjyd_6467uQrZd7a7SmWJg-Exz9g/edit

then we will browse to SonarQube and create a token in the name of jenkins

then come to credentials under manage jenkins select global add credential select 'secret text' in kind section give ID sonar-token

then come to manage jenkins > system > add sonarQube give name sonar-server give url of sonarQube and it will take token name also 

Now, we have to add Dockerhub credential in credential section add username[chandravir1996] & password there of dockerhub

Also we have to go to tmdb website, create account there and create api, we have to use that api in docker section 

Here we use api key in

docker build --build-arg TMDB_V3_API_KEY=cffd98e475df4af6af0e4ce156a5d0fe -t netflix ."
In above command 


Also, we are Deploying this on Kubernetes so we have to make k8s nodes either by kubeadm or eks [ I am using EKS }

After creating EKS Cluster 

we have to to add config file in credential section of Jenkins for that we have to go to our Master node config file is stored in /root/.kube here we to copy it in notepad and save the config file and choose that notepad file in credential choose secret file in kind section 

Then we have to add aws access key and secret access key in credential section 

Here is the Jekins scrip we are using

pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/devopscloudlabs/Netflix-clone.git'
            }
        }
        stage('OWASP Dependencies Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix-Clone-App \
                    -Dsonar.projectKey=Netflix-Clone-App '''
                }
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=cc983e706b41d5c3c30a48438b89897b -t netflix ."
                       sh "docker tag netflix chandravir1996/netflix:latest "
                       sh "docker push chandravir1996/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image chandravir1996/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 chandravir1996/netflix:latest'
            }
        }
        stage('Deploy to kubernetes'){
            steps{
                script{
                    dir('Kubernetes') {
                        withCredentials([[
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: 'jenkins-test',
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                        ]]) {
                            withKubeConfig(
                                caCertificate: '',
                                clusterName: '',
                                contextName: '',
                                credentialsId: 'k8s',
                                namespace: '',
                                restrictKubeConfigAccess: false,
                                serverUrl: ''
                            ) {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                            }
                        }
                    }
                }
            }
        }
    }
}

save and apply go to build now it will be succesfull.

