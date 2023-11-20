<h1 align="center">DevSecOps CI/CD with Jenkins and ArgoCD</h1>

This repository contains the source code and Jenkinsfile used to automate the Continous Integration phases of a CI/CD pipeline to EKS. Continous delivery is accomplished via ArgoCD. This repo is intended to be used with the two repositories listed below.
- [eks-infra](https://github.com/yemisprojects/eks-infra) repo: contains terraform code and github workflow to automate EKS deployment
- [kubernetes-manifests](https://github.com/yemisprojects/kubernetes-manifests) repo: contains helm charts for deployment by ArgoCD

<h2 align="center">Pipeline Stages</h2>

<img alt="Jenkins pipeline" src="https://github.com/yemisprojects/eks-app/blob/main/images/Final_pipeline.png">

[Jenkins](https://www.jenkins.io) is open source and free. With numerous plugins available it provides easy integration to many third party systems. If server management is a not a major concern, Jenkins is a great choice for CI.
[Checkstyle](https://checkstyle.sourceforge.io) is a development tool to help programmers write Java code that adheres to a coding standard. It automates the process of checking Java code to spare humans of this boring (but important) task. This makes it ideal for projects that want to enforce a coding standard.
OWASP Dependency-Check is a Software Composition Analysis (SCA) tool that actively scans through a project's dependencies to detect and report on publicly disclosed vulnerabilities, thereby improving application security.


## Application Stack
The application is exposed using the AWS Application Load Balancer and the stack consists of a Java application, MySQL, Rabbitmq and Memcache 

#### Application requirements
- JDK 1.8 or later
- Maven 3 or later
- MySQL 5.6 or later
<!--- 
## Technologies 
- Spring MVC
- Spring Security
- Spring Data JPA
- Maven
- JSP
- MySQL
--->
#### Docker Images
- Memchache and Rabbitmq official docker images will be used by the application in EKS
- The DB/MySQL image has been prebuilt with some dummy data and the official MySQL docker image. 
    - The script is located in this repo at `src/main/resources/db_backup.sql` 
    - The image is available on dockerhub with this tag, `yemisiomonijo/vprofiledb:1`
    - If needed, the same image was built and pushed to dockerHub with these commands
    ```sh
    cd Docker-files-local-test/db
    docker build -t yemisiomonijo/vprofiledb:1 . 
    docker push yemisiomonijo/vprofiledb:1 .
    ```
- The application image will be built and deployed by the Jenkins pipeline with GitOps
<!--- 
mysql -u <user_name> -p accounts < accountsdb.sql
--->
## CICD setup prerequisites
1. DockerHub account
2. SonarCloud account
3. Github account
4. Email address
5. Slack Channel
5. Jenkins Instance (This should have been deployed using instructions from my [eks-infra repo](https://github.com/yemisprojects/eks-infra))
6. EKS Cluster (This should have been deployed using instructions from my [eks-infra repo](https://github.com/yemisprojects/eks-infra))

## CI/CD initial setup (required)

#### Step 1. Setup Slack
- Create a new Slack account followed by a workspace using the [steps here](https://slack.com/help/articles/206845317-Create-a-Slack-workspace). 
- Within the new workspace [create a slack channel](https://slack.com/help/articles/201402297-Create-a-channel) named `k8s-jenkins-cicd`.
- Apply the Jenkins App within the Slack workspace. 
    - Go to the Slack Jenkins app [here](https://slack.com/apps/A0F7VRFKN-jenkins-ci?tab=more_info) → Click Install → Add to Slack
    - Choose the channel `k8s-jenkins-cicd`. 
    - Add Jenkins CI integration
    - Proceed with the Setup Instructions and note down the auto-generated _Integration token credential ID_

#### Step 2. Setup Github repository
- Fork this repository
- Create Github webhook: To trigger the Jenkins pipeline automatically after each push, create a github webhook with these steps
    - Go to your GitHub repository. Under Settings → Add Webhooks. Use the information below _where x.x.x.x is jenkins public IP_. Refresh browser and confirm the test ping is successful. The Jenkins security group has been setup allow the Github webhook access
    ```
            Payload URL: http://x.x.x.x:8080/github-webhook/   
            Content type: application/json
    ```
- Create a Github personal access token(classic) with admin privileges and note down the token for a later step when setting up Jenkins

#### Step 3. Setup SonarCloud
- Go to the SonarCloud [website](https://sonarcloud.io/) and signup for a free account
- Create an organization and note down the name for a latter step
- Create a project named `vprofile-app`. 
- Create a webook 
    - Go to the `vprofile-app` → `Administration` → `Configuration` → `Webhooks`
    - Click Create and provide the information below, _where x.x.x.x is jenkins public IP_
    ```
    Name: Jenkins
    URL: http://x.x.x.x:8080/sonarqube-webhook          
    ``` 
- Create a new Quality Gate. Documentation [here](https://docs.sonarsource.com/sonarcloud/standards/managing-quality-gates/#:~:text=To%20create%20a%20new%20quality,in%20Your%20Organization%20>%20Quality%20Gates.)
    - Go to Organization → Quality Gates → Create → Name (jenkins)
    - Add a condition. Select `On Overall Code` → `Quality Gate failes when` → `Bugs` → is greater than `50` (For test purposes)
    - The app used for this project has 29 bugs so the Quality gate test will pass
    - Ensure to set the new Quality gate as default to make sure that this quality gate will apply to any new code analysis
    <!---IMPORTANT https://jenkinshero.com/sonarqube-quality-gates-in-jenkins-build-pipeline/ --->

#### Step 4. Setup Jenkins
- Login to Jenkins via SSM or SSH to get the initial password with the command below: 
   ```sh 
   cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

- Login to the Jenkins Web UI at http://x.x.x.x:8080/ _where x.x.x.x is jenkins public IP_ using the default username (admin)and the password obtained  above. Accept the recommendation to Install the recommended plugins. 

- #### Install Jenkins Plugins
    - Goto Manage Jenkins → Plugins → Available Plugins → Select all the plugins below → Install. If a plugin is not listed, it is most likely installed
        - SonarQube Scanner 
        - Email Extension Plugin
        - Eclipse Temurin Installer
        - OWASP Dependency-Check
        - Docker
        - Docker Commons
        - Docker Pipeline
        - Docker API
        - Docker-build-step
        - Slack Notification
        - Github Integration
        - HTML Publisher plugin
        - Blue Ocean plugin  (optional)

- #### Install Jenkins Tools
    - Goto Manage Jenkins → Tools. Install each tool using the configuration below. Click Apply and Save
        - JDK Installations
            - Name: jdk17
            - Tick Install automatically → Click Add Installer → Install from adoptium.net 
            - Version: JDK-17.0.8.1+1
        - SonarQube Scanner Installations
            - Name: sonar_scanner
            - Tick: Install automatically
            - Version: SonarQube Scanner 5.0.1.3006
        - Dependency-Check Installations
            - Name: dependency_check
            - Tick: Install automatically → Click Install from github.com
            - Version: dependency-check 6.5.1	

- #### Add credentials to Jenkins
    - Goto Jenkins Dashboard → Manage Jenkins → Credentials → Select global → Add credentials
    - Add Sonarqube token
        - Obtain the token by logging into SonarCloud. Goto My Account  → Security → Enter Token Name → Create a token → Click on Generate Token.
        - Add the sonar token in Jenkins using the configuration below and click Create. _Replace **** with sonarqube token_
        ```
             Kind: Secret text
             Secret: ****			        
             ID: sonar_token
             Description: sonar_token
        ```
    - Add Jenkins Slack integration token using the configuration below. _Replace **** with slack token_
    ```
            Kind: Secret text
            Secret: ****	          
            ID: slack_token
            Description: slack_token
    ```
    - Add Dockerhub login credential using the configuration below. _Replace **** with docker password and userx _ 
    ```
            Kind: Username with password
            Username: userx			
            Password: **** 		          		
            ID: docker_cred
            Description: docker_cred
    ```
    - Add Github personal token. _Replace userx with your GitHub username and **** with your [Github token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) with admin access_
    ```
            Kind: Username with password
            Username: userx			
            Password: **** 		          	
            ID: github_token
            Description: github_token
    ```  
    - Add email credentials. If you have a Gmail account, generate an App password using the [steps here](https://support.google.com/mail/answer/185833?hl=en). Add the email credential on Jenkins using the information below. _Replace **** with your app password_
    ```
        Kind: Username with password
	    Username: userx@gmail.com	
        Password: **** 		          		
	    ID: email_cred
        Description: email_cred
    ```
- #### Update Jenkins system configuration
  - Login to Jenkins, go to Dashboard → Manage Jenkins → System.Use the information below to add various server configurations

    - Under SonarQube servers and SonarQube installations, use the information below
	    ```
        Name: sonarcloud_server
  	    Server URL:  https://sonarcloud.io    
        Server authentication token: sonar_token
        ```

    -  Setup Email Notification under Extended Email Notification. Click Apply & Save
    ```
        SMTP server: smtp.gmail.com
        SMTP Port: 465
        Click Advanced
        Credentials: email_cred 
        Tick: Use SSL
        Default Content Type: HTML(text/html)
        Default triggers: 
            Tick: Always
                    Failure - Any
    ```

    - Under Slack, use the information below. Select test connection after setup. _Replace xxxx with the name of your slack workspace_
    ```
	Workspace:  xxxx  			
    credential: slack_token 
    Default channel: #k8s-jenkins-cicd   
    ```

- #### Create two Jenkins pipelines
    ##### Pipeline No. 1
    - On Jenkins Dashboard, click New Item → Select Pipeline → Enter Item name, e.g `k8s-pipeline` and click OK
    - Under Build triggers, select GitHub hook trigger for GITScm polling
    - Under Pipeline. Select these options. Replace zzzzzzzz with your github name Apply and Click Save
    ```
            Definition: Pipeline script from SCM
            SCM: Git
            Repository URL: https://github.com/zzzzzzzz/eks-app
            Branch Specifier: */main
            Script Path: Jenkinsfile
    ```
    ##### Pipeline No. 2
    - On Jenkins Dashboard, click New Item → Select Pipeline → Enter this Item name, e.g `update-k8-manifest` and click OK
    - Under Pipeline. Select these options. Replace zzzzzzzz with your github name Apply and Click Save
    ```
            Tick: This project is parameterized
                Add Parameter -> Select String Parameter:
                    Name: DOCKER_TAG
            Definition: Pipeline script from SCM
            SCM: Git
            Repository URL: https://github.com/zzzzzzzz/kubernetes-manifests
            Branch Specifier: */main
            Script Path: Jenkinsfile
    ```

#### Step 5. Update Jenkinsfile 

After forking this eks-app repo, make the following changes to the Jenkinsfile

- Replace `yemisiomonijo` in the Jenkinsfile with your dockerhub username
```
.......
    environment {
        SONAR_SCANNER_HOME = tool 'sonar_scanner'
        DOCKER_REGISTRY = "yemisiomonijo/vprofileapp"
        DOCKER_REG_CRED = 'docker_reg_cred'
    }
```
- Replace `dummyuser@yahoo.com` with the email address you wish to receive pipeline notifications, image scan results and build logs.
```
.......
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                        "Build Number: ${env.BUILD_NUMBER}<br/>" +
                        "URL: ${env.BUILD_URL}<br/>",
                to: 'dummyuser@yahoo.com',
                attachmentsPattern: 'filesystem_scanresults.txt,filesystem_scanresults.json,image_scan.txt,image_scanresults.json'
```

#### Step 6. Update helm chart repository

- Fork the [kubernetes-manifest](https://github.com/yemisprojects/kubernetes-manifests) repository containing the application's helm charts
- Replace `yemisiomonijo` in the Jenkinsfile with your dockerhub username
```
........
    environment {
        DOCKER_REGISTRY = "yemisiomonijo/vprofileapp"
    }
```
- Replace `vprofile.devopsprotech.com` in the `webapp/templates/vproapp-ingress.yaml` file with a CNAME record within your domain. I have used Route53
```
......
  rules:
    - host: vprofile.devopsprotech.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
```
- Push all changes to the repository

### Trigger Jenkins Pipeline
- Comit and push your changes to the main branch
- This should trigger the `k8s-pipeline`. This is a pipeline view using blue ocean
<img alt="Pipeline stages" src="https://github.com/yemisprojects/eks-app/blob/main/images/Pipeline%201.png">

- The OWASP dependency scan results can be seen from the UI once the pipeline completes execution
<img alt="Dependency check results" src="https://github.com/yemisprojects/eks-app/blob/main/images/Dependency%20check%20results.png">

- The SonarQube Quality gate results is also shown on the Pipeline UI
<img alt="Quality Gate" src="https://github.com/yemisprojects/eks-app/blob/main/images/Sonarcloud%20quality%20gate.png">

- The `update-k8-manifest` pipeline is triggered after the `k8s-pipeline` completes. It updates the image application tag in the `kubernetes-manifests` repository
<img alt="Helm repo Update" src="https://github.com/yemisprojects/eks-app/blob/main/images/Pipeline%202.png">

- A slack notification should be recieved on the slack channel
<img alt="Slack notification" src="https://github.com/yemisprojects/eks-app/blob/main/images/slack%20notification.png">

- You should receive an email with attachment of the build log and trivy image scan results

### Troubleshooting
- If there are any issues with pods, authenticate using the `eksadmin1` user mentioned in the eks-infra repo.
Run the command below and review the logs. Replace `eksadmin1` with the AWS CLI profile created for the `eksadmin1`user
```sh
aws eks update-kubeconfig --region us-west-2 --name eks-poc --profile <eksadmin1>
kubectl logs -f POD_ID -f
```
- To check the AWS load balancer controller logs
```sh
kubectl logs -f -n kube-system  -l app.kubernetes.io/name=aws-load-balancer-controller

```




