# CI Pipeline for Grails based application using Gitolite, Jenkins, Gradle and SonarQube

This project enumerates the steps required to create a Continuous Integration Pipeline. The Pipeline is created based on the GitOps philosophy wherein Git acts as a single source of truth and the initiation of the pipeline is triggered through  Git events.

## Documentation

https://www.jenkins.io/doc/

https://docs.sonarqube.org/latest/

https://docs.sonarqube.org/latest/analyzing-source-code/scanners/sonarscanner-for-gradle/

## Architecture

![ci-pipeline drawio(7)](https://github.com/abmohanty1991/Apache-Superset/assets/108822178/1d26d6f8-9c17-4392-b755-93620c25475a)


The pipeline gets triggered by PUSH event to a specified branch in the Git. This event triggers a Jenkins job, which then under takes the following actions:

- Establish Handshake with Gitolite and clone the codes.
- Establish handshake with the Sonsarqube server and present the code for Static Code Analysis.
- Build the code and generate web archive file (.war).
- Establish handshake with NAS storage and store the .war file in the day wise folder.

## Applications/Technology used:

   -  Grails based web application
   -  Gitolite version control system (a wrapper around Git)
   -  Jenkins Server
   -  Sonarqube server

## Steps

    1. Install and configure Jenkins
       1.1 Install Jenkins Server
       1.2 Install the Git, Gradle, Pipeline and Sonarqube plugin
    
    2. Install and Configure Sonarqube server
       2.1 Install Sonarqube server
       2.2 Install SonarScanner
       2.3 Install Groovy Plugin
       2.4 Configure sonar-project.properties file

    3. Establishing Authentication Mechanism
       3.1 Authentication between Jenkins and Gitolite
       3.2 Authentication between Jenkins and Sonarqube
       3.3 Authentication between Jenkins and NAS

    4. Writing the webhook in Gitolite to trigger Jenkins Job

    5. Writing the Jenkins Pipeline Script
        Stage 1: Checkout the source code from Git and clone
        Stage 2: Execute a sonar test command with sonarqube credentials
        Stage 3: Build war file and store in predefined location
        Stage 4: Connect to NAS and store war file


## Install Java and Jenkins

- Install Java

``` bash
sudo apt update
sudo apt install openjdk-11-jre
```

Verify Java is Installed

``` bash
java -version
```

- Now, you can proceed with installing Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```
- After you login to Jenkins (at port 8080), - Run the command to copy the Jenkins Admin Password - `sudo cat /var/lib/jenkins/secrets/initialAdminPassword - Enter the Administrator password`

![image](https://github.com/abmohanty1991/Apache-Superset/assets/108822178/c831a745-d796-4f28-bcb9-ef3b24362a47)

- Click on Install suggested plugins

![image](https://github.com/abmohanty1991/Apache-Superset/assets/108822178/ac112cf8-0953-4f4f-94ca-1213fc0da85c)

- Wait for the Jenkins to Install suggested plugins

![image](https://github.com/abmohanty1991/Apache-Superset/assets/108822178/af6f4206-d4d3-4ee5-882d-f5aab7904276)

- Create First Admin User or Skip the step [If you want to use this Jenkins instance for future use-cases as well, better to create admin user] and Jenkins Installation is successful. 

- Now Go to `Manage Jenkins -> Manage Plugins -> Install the Git, Pipeline, Sonarqube and Gradle Plugin`


## Install and Configure Sonarqube

- Use the following steps to install Sonarqube server locally
``` bash
apt install unzip
adduser sonarqube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip *
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

- Now go to `Administartion -> Plugins` and install the Groovy plugin (considering our code files are written in groovy)

![image](https://github.com/abmohanty1991/Apache-Superset/assets/108822178/e1fc4afc-4323-467d-8fb2-9f7f8224378c)

- Download `SonarScanner` for the respective OS from following link https://docs.sonarqube.org/latest/analyzing-source-code/scanners/sonarscanner/. Let us call the installation directory as <INSTALLATION_DIR>. Add the <INSTALLATION_DIR> to PATH. In case of ubuntu:
```
export PATH = <INSTALLATION_DIR>:$PATH
```

In case there is no PostgreSQL database installed, follow steps of complete installation given in this link:-
https://www.vultr.com/docs/install-sonarqube-on-ubuntu-20-04-lts/

## Configuring the project 
- Now create a file called `sonar-project.properties` file in the root of the project directory

```bash
# must be unique in a given SonarQube instance
sonar.projectKey=my:project

# --- optional properties ---

# defaults to project key
sonar.projectName=My project
# defaults to 'not provided'
sonar.projectVersion=1.0
 
# Path is relative to the sonar-project.properties file. Defaults to .
#sonar.sources=.
sonar.sources = src/main, grails-app/controllers, grails-app/domains
 
# Encoding of the source code. Default is default system encoding
sonar.sourceEncoding=UTF-8
```


## Authentication between Jenkins and Sonarqube

Go to Sonarqube server-> Administrator icon on top right -> Security -> Generate a Global Analysis token for user Jenkins. Please copy and store the token somewhere as it wont't be visible again.

Now go to `Jenkins server -> Manager Jenkins -> Crendentials -> System -> Global Credentials (unrestricted)`

Choose Secret Text in the option `"Kind"`. Insert the token and put id as sonarqube. You are free to choose the id the way you want but make sure to use it while undertaking the code analysis.

![image](https://github.com/abmohanty1991/Apache-Superset/assets/108822178/5e227117-38bc-46a9-a0fb-4ecefe3af8f9)

This enables Jenkins to get authenticated with the Gitolite server whenever we fire the static code analysis.


## Writing the Jenkins file 

Create a pipeline project and put the following code in the pipeline script.

```bash
node{
    # This stage utilises the credentials of Git as stored in Jenkins, autheticates to the specified branch and clones the branch
    stage('Clone Repo'){
        git branch: 'development', credentialsId: 'git', url: 'gitserver@172.17.1.2:NAMS'
    }
    # This stage utilised the credentials of Sonarqube as stored in Jenkins and authenticates Jenkins with SOnarqube
    # cd to the project folder.
    # Sonarqube by defualt searches for the folder "main in the root project dir.
    # In our case it is at path src/main. So copying the "main" folder to root project directory
    # Utilising modified gradle-wrapper.properties to hint gradlew to use local dependencies folder rather than internet
    # Granting permission to the project root folder
    # Invoke sonarqube testing, use th project token recived after creatinga  project in Sonarqube server.
    stage('Static Code Analysis') {
      environment {
        SONAR_URL = "http://127.0.0.1:9000"
      }
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
    
        sh script:'''
        #!/bin/bash

        # Go to the pipeline folder under /var/lib
        # Copy the gradle-wrapper.properties file to the gradle/wrapper folder in your pipeline
        # Assign necessary permissions to the pipeline
        ./gradlew sonar -Dsonar.projectKey=<project key> -Dsonar.host.url=<url where sonarqube is hosted> -Dsonar.login=<unique key>
        '''
        }
    }

    # This stage invokes web archive file generation
    # cd to the project root folder.
    # Utilising modified gradle-wrapper.properties to hint gradlew to use local dependencies folder rather than internet followed by invoke war
    stage('WAR build'){
        sh script:'''
        #!/bin/bash
        # cd to the pipeline folder
        # assign necessary permissions
        # Build war (web archive)
        ./gradlew war
        
        '''
    }
    
    # This stage autheticates Jenkins with NAS
    # Using the sshagent, ssh into the NAS
    # Make a directory with name as the sysdate
    # Copy the generated .war file from the location (as done in war build stage) to folder in NAS.
    # This serves as a war version control  and will help in rollback to a previous known healthy configuration
    stage('WAR Transfer to NAS'){
        sshagent(['nas']) {
        sh script:'''
        #!/bin/bash
        ssh -o StrictHostKeyChecking=no admin@<location to NAS> mkdir <folder to save war>####WAR####/2023/MAY_23/"$(date +"%d-%m-%Y")"
        scp -o StrictHostKeyChecking=no <location to war file> admin@<location to NAS>:<folder to save war>####WAR####/2023/MAY_23/"$(date +"%d-%m-%Y")"/
        '''
        }
    }
}
```
