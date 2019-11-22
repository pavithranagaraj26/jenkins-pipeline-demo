<h2>CICD integration with DOCKER/JENKINS/GITHUB/DOCKERHUB/KUBERNETES</h2>

Our goal is to ensure our pipeline works well after each code being pushed. The processes we want to auto-manage:
* Code checkout
* Run tests
* Compile the code
* Create/build the Docker image
* Push the image to Docker Hub
* Pull and deploy it in kubernetes cluster

## First step, running up the services

### GitHub configuration

Create a github repo and add all the deployment files, dockerfile and jenkinsfile with the dependencies

### Jenkins configuration

## Jenkins configuration

We have configured Jenkins in the docker compose file to run on port 8080 therefore if we visit http://localhost:8080 we will be greeted with a screen like this.

![](images/004.png)

We need the admin password to proceed to installation. It’s stored in the ``/var/jenkins_home/secrets/initialAdminPassword`` directory and also It’s written as output on the console when Jenkins starts.

```
jenkins      | *************************************************************
jenkins      |
jenkins      | Jenkins initial setup is required. An admin user has been created and a password generated.
jenkins      | Please use the following password to proceed to installation:
jenkins      |
jenkins      | ********************************
jenkins      |
jenkins      | This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
jenkins      |
jenkins      | *************************************************************
```

To access the password from the container.

```
docker exec -it jenkins sh
/ $ cat /var/jenkins_home/secrets/initialAdminPassword
```

After entering the password, we will download recommended plugins and define an ``admin user``.


After clicking **Save and Finish** and **Start using Jenkins** buttons, we should be seeing the Jenkins homepage. One of the seven goals listed above is that we must have the ability to build an image in the Jenkins being dockerized. Take a look at the volume definitions of the Jenkins service in the compose file.
```
- /var/run/docker.sock:/var/run/docker.sock
```
Add the credentials for github, dockerhub and kubernetes:

dockerhub:

github:

kubernetes:

### Dockerhub configuration

Create a Dockerhub repository.

### kubernetes configuration

configure the kuberenetes with docker-desktop.

## pipeline creation

### Dockerfile :

```
FROM nginx

COPY wrapper.sh /

COPY html /usr/share/nginx/html

CMD ["./wrapper.sh"]
```
The docker image will be built using the dockerfile specifications here we use nginx as base image.

### Jenkinsfile :

```
//Jenkinsfile to perform pipeline
pipeline {
  environment {
    registry = "impavithra/cicd"
    registryCredential = 'dockerhub'
    dockerImage = '' // configurations to point to the repository inside dockerhub
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/pavithranagaraj26/jenkins-pipeline-demo.git'
      }
    } // cloning the github repo to build the image
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }// building the image using the dockerfile cloned from the github repo 
    stage('Deploy Image') {
      steps{
        script {
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    } // image is been pushed to the dockerhub with the name and tag
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    } // removing the unused image
    stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'Deploy.yml',
                    enableConfigSubstitution: true
                )
            } // using the kubeconfig file (in which deployements and services are specified) the image is been deployed 
        }
    }
}
```
### Deploy.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp1
  template:
    metadata:
      labels:
        app: webapp1
    spec:
      containers:
      - name: webapp1
        image: $registry:$BUILD_NUMBER
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: webapp1-svc
  labels:
    app: webapp1
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
  selector:
    app: webapp1
```

## using the files create a pipeline in jenkins and build the app
