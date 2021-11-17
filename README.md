# java-web-app-docker

## USAGE 

This is just a forked repo to test working of Jenkins Pipeline with Docker Integration

## STEPS : 

1) Install Jenkins, Docker in CI/CD Server.
2) Install Docker in Deployment Server.
3) Create Jenkins Pipeline Job to build and deploy docker image in Docker(Deployment) Server.

## Prerequisites : 

- AWS Account
- Docker Hub Account

## Setting up CI/CD Server

Update package manager
```
sudo apt update
```

Install Java
```
sudo apt install openjdk-8-jdk -y
```

Install Jenkins
```
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

Restart jenkins
```
sudo apt update
sudo systemctl status jenkins
```

Install Docker
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Add Jenkins user to docker
```
sudo usermod -a -G docker jenkins
```

Restart Jenkins
```
sudo systemctl restart jenkins
```

## Setting up the Deployment Server

Update package manager
```
sudo apt update
```

Install docker
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Add ubuntu user to docker
```
sudo usermod -aG docker ubuntu
```

## Configuring Jenkins

### Plugins to be installed

- SSH Agent
  Dashboard -> Manage Jenkins -> Manage Plugins -> SSH Agent -> Install without restart

### Install Maven

Jenkins Dashboard-> Manage Jenkins->Global Tool Configuration->Add Maven

### Add DockerHub Password as secret text : 
pipeline syntax -> withCredentials -> Add Credentials

Secret : [PASSWORD]

ID : DockerHubPwd

### Add SSH Login credentials
pipeline syntax -> SSH Agent

Add Credentials -> Add username and key

Copy your key from .pem file and paste it

### Creating the Pipeline

Dashboard-> New item -> 
  Name : Java-Web-App
  type : pipeline
  
 In pipeline script add : 
 
 ```
 node{
    def buildNumber = BUILD_NUMBER
    stage("Git Clone"){
        git url:'https://github.com/kiruthik-prakash-j/java-web-app-docker.git', branch: 'master'
    }
    
    stage("Maven Clean Package"){
        def mavenHome= tool name:"Maven", type: "maven"
        sh "${mavenHome}/bin/mvn clean package"
    }
    
    stage("Build Docker Image"){
        sh "docker build -t kiruthikprakashj/java-web-app-docker:${buildNumber} ."
    }
    
    stage("Docker Login and Push"){
        withCredentials([string(credentialsId: 'DockerHubPwd', variable: 'DockerHubPwd')]) {
            sh "docker login -u kiruthikprakashj -p ${DockerHubPwd}"
        }
        sh "docker push kiruthikprakashj/java-web-app-docker:${buildNumber}"
    }
    
    stage("Deploy Application as Docker Container in Docker Deployment Server"){
        sshagent(['Docker_Deployment_Server_SSH']) {
            sh "ssh -o StrictHostKeyChecking=no ubuntu@[DeploymentServerIP] docker rm -f javawebappcontainer || true"
            
            sh "ssh -o StrictHostKeyChecking=no ubuntu@[DeploymentServerIP] docker  run -d -p 8080:8080 --name javawebappcontainer kiruthikprakashj/java-web-app-docker:${buildNumber}"
        }
    }
}
```

### Build and Run

Build the pipeline

Open the public IP port of the Deployment Server in the browser
```
http://[DeploymentServerIP]/java-web-app/#
```
