# Jenkins
Jenkins is an automation tool to build, test and deploy software.

## Jenkins has Master node and agents.
Master controls Pipelines, schedules builds.
Agent performs builds.

There are permanent agents as dedicated physical machines and cloud agents (ephemeral and dynamic) as Docker, Kubernetes, AWS.

## Build Types

Freestyle Build
- looks like shell scripting
- simple method to create builds

Pipelines
- Groovy syntax
- Build is broken into stages

```bash
pipeline {
    agent { 
        node {
            label 'jenkins-agent-goes-here'
            }
      }
    stages {
        stage('Build') {
            steps {
                echo "Building.."
                sh '''
                echo "doing build stuff.."
                '''
            }
        }
        stage('Test') {
            steps {
                echo "Testing.."
                sh '''
                echo "doing test stuff..
                '''
            }
        }
        stage('Deliver') {
            steps {
                echo 'Deliver....'
                sh '''
                echo "doing delivery stuff.."
                '''
            }
        }
    }
}
```


## Installation

You can install Jenkins on your local computer, in cloud environment or in a virtual environment locally like Docker.

For details see, https://www.jenkins.io/doc/book/installing/

- We will use Jenkins as Docker containers.
- We have 3 steps to use Jenkins in a container. Note that this may depend on which image you decide to use.

1 ***Build Jenkins Image***
2 ***Create Jenkins Network***
3 ***Run the container***


### Build Jenkins Image

- create the Dockerfile

```Dockerfile
FROM jenkins/jenkins:2.332.3-jdk11
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.3 docker-workflow:1.28"
```

- build the image

```bash
docker build -t b107-jenkins-blueocean:2.332.3-1 .
```

### Create Jenkins Network


```bash
docker network create jenkins
```

### Run Jenkins Container

```bash
docker run --name b107-jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  b107-jenkins-blueocean:2.332.3-1
```

- Windows syntax

```bash
docker run --name b107-jenkins-blueocean --restart=on-failure --detach `
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 `
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 `
  --volume jenkins-data:/var/jenkins_home `
  --volume jenkins-docker-certs:/certs/client:ro `
  --publish 8080:8080 --publish 50000:50000 b107-jenkins-blueocean:2.332.3-1
```

### Get the Jenkins password

```bash
docker exec b107-jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

### Connect to Jenkins

```bash
https://localhost:8080
```

### Install suggested Plugins

### Create Admin User and Start Using Jenkins


### Manage Jenkins

- Explore system Configuration, Manage Plugins, Manage Nodes and Clouds
- You can add Slack plugin and notify about build status on slack channels. Add Jenkins app in slack first then add plugin in Jenkins and then configure.
- Explore Manage Credentials
- Explore Status Information
- Explore Tools and Actions > Prepare for Shutdown


### Start Building

There are two main job types:
- Freestyle project
- Pipeline

For both job types:
- Enter item name without spaces!!

Notice:
- In case Jenkins is behind a firewall you will need to configure ports.
- This can be bypassed by using `Build periodically` or `Poll SCM`
- This a private local Docker environment, you cannot be reached. 

**Add Slack Notification**
- Add Post-build Action
 - Slack Notifications
   - add any notification type that suits you

**Add Build Steps**
- Add Build Step
 - execute shell
   - add a shell command to be executed as a build step
```bash
echo hello
```

- add Jenkins ENV variable to the build step as output

```bash
echo Build ID of this $JOB_NAME job is $BUILD_ID, Build Number is $BUILD_NUMBER 
```

- send another shell command to create a file and list the files

```bash
touch file.txt
ls -ltr
```

- click job name and see the Workspace on the left over `Build Now` option
  - click Workspace to see what is in your workspace.


- configure the job so that the workspace is cleared
- Click configure
- Build Environment
  - Check `Delete workspace before build starts`
  - If you build again you will see that the workspace is cleared.


- See Jenkins file structure
    - Enter Jenkins container and explore the workspace folders

```bash
docker exec -it containerID bash
``` 
- attention to the prompt as it is not a root user prompt
```bash
jenkins@632da2d84858:/$
```
- see `/var/jenkins_home``
    - you can troubleshoot here
    - you can see all plugins here

- see `/var/jenkins_home/workspace` that there is a folder as your job name and inside your job files

### Create Another Freestyle Project

- create a freestyle job and name it python-test

- check if python is installed on the Jenkins master container

- use Source Code Management
    - use Git
        - repository URL
        ```bash
        https://github.com/mefekax/jenkins-101
        ```
- add build step
```bash
python3 hello.py
```
- Build the Project and observe the output


### Manage Nodes and Clouds

- Go to main Dashboard
- Go to Manage Jenkins then Manage Nodes and Clouds

- New Node: Permanent agent
- Configure Clouds: Ephemeral agents

- click Configure Clouds and click `Go to plugin manager``
- Plugin Manager automatically filters `Cloud Providers``

- Select Docker as cloud agent
- Click `Download now and install after restart``

### Set Docker Cloud Agent

- Go to Configure Clouds again
- Select Docker and then Docker Cloud Details
- Set URI and click test Connection
    - for URI setting, you need to get Docker Desktop link. To be able to get that link we use another container.
    ```bash
    docker run -d --restart=always -p 2376:2375 --network jenkins -v /var/run/docker.sock:/var/run/docker.sock alpine/socat tcp-listen:2375,fork,reuseaddr unix-connect:/var/run/docker.sock
    docker inspect <container_id> | grep IPAddress
    ```
    - get the IP address and add it in this form into URI field.
    ```bash
    tcp://172.20.0.3:2375
    ```

- Do not forget to select Enabled
- Now you have set up Docker as Cloud agent/node

### Configure Docker Cloud agent Image/Template

- Now configure Docker agent images, click Docker Agent Templates
    - Labels: docker-agent-alpine
    - Enabled: check
    - Name: docker-agent-alpine
    - Docker Image: jenkins/agent:alpine-jdk11
    - Instance Capacity: 2
    - Remote File System Root: /home/jenkins 
- Save changes

- go to Dashboard and then select one of the previous Projects
- Click Configure
- Click Restrict where this project can be run
    - Label Expression: docker-agent-alpine

- Save and Build
- Check Docker Desktop for the agent container

### Set another Docker Cloud Agent Image/Template

- To be able to run python on a docekr agent the image should be ready for python
- use the image to add another Cloud Template
    image name: mefekadocker/jenkinsagent:python

- now build python project restricting to run on this docker agent

### Poll SCM

- Go to python-test project
- Go to Build Triggers
- Select Poll SCM
    Schedule: */2 * * * *
    (best practice to poll git changes)
- As Jenkins runs on localhost/containers we may not configure Github webhook 

### Create Pipeline

- Create New Item as a Pipeline named my-build-pipeline
- Pipeline:
```bash
pipeline {
    agent { 
        node {
            label 'jenkins-agent-goes-here'
            }
      }
    stages {
        stage('Build') {
            steps {
                echo "Building.."
                sh '''
                echo "doing build stuff.."
                '''
            }
        }
        stage('Test') {
            steps {
                echo "Testing.."
                sh '''
                echo "doing test stuff..
                '''
            }
        }
        stage('Deliver') {
            steps {
                echo 'Deliver....'
                sh '''
                echo "doing delivery stuff.."
                '''
            }
        }
    }
}
```

- modify script as the following and save it in the Github repo as Jenkinsfile 

```bash
pipeline {
    triggers {
        pollSCM '* * * * *'
    }
    agent { 
        node {
            label 'docker-agent-python'
            }
      }
    stages {
        stage('Build') {
            steps {
                echo "Building.."
                sh '''
                echo "doing build stuff.."
                '''
            }
        }
        stage('Test') {
            steps {
                echo "Testing.."
                sh '''
                
                python3 hello.py 
                '''
            }
        }
        stage('Deliver') {
            steps {
                echo 'Deliver....'
                sh '''
                echo "doing delivery stuff.."
                '''
            }
        }
    }
}
```

- Build it first, then make a small change in Github repo and wait for it to poll.


### Discover BlueOcean














