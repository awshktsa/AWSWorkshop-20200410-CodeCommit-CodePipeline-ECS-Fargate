# AWSWorkshop-20200410-CodeCommit-CodePipeline-ECS-Fargate
An End to End flow from CodeCommit, through CodePipeline to build docker image then push to ECR, then another pipeline will be triggered to deploy the image to ECS Fargate.

------

## In this workshop, we will have a quick review following services:
- Amazon Elastic Container Service (ECS) https://aws.amazon.com/ecs/
- Amazon Elastic Container Registry (ECR) https://aws.amazon.com/ecr/
- AWS Fargate https://aws.amazon.com/fargate/
- AWS CodeCommit https://aws.amazon.com/codecommit/
- AWS CodeBuild https://aws.amazon.com/codebuild/
- AWS CodeDeploy https://aws.amazon.com/codedeploy/
- AWS CodePipeline https://aws.amazon.com/codepipeline/
------

### Step 1:
* Switch Region on the AWS console, a drag down menu near right-up corner.
Pick one region close to you, if you don't have any prefer, use **us-east-1**

------
### Step 2: Setup IAM Role/User for this workshop
- **AWS Console > Services > IAM > Role**
- Create Role, service pick "EC2" and click "Next"
- Search "AWSCodeCommitFullAccess","AmazonEC2ContainerRegistryFullAccess" and click the check box
- Click Next, Inut Tag Key and Value if you want
- Click Next to enter "Name" = "Workshop_Cloud9"
- and "Description" = "Temp EC2 Role for Workshop " and click "Create" for the Role.
------
- Create Role, service pick "CodeDeploy", select "ECS" and click to "Next"
- Click Next to enter "Name" = "Workshop_CodeDeploy"
- and "Description" = "Temp CodeDeploy Role for Workshop " and click "Create" for the Role.

- Click "Attach Policy'
- Search "AWSCodeDeployRoleForECS", "AmazonS3ReadOnlyAccess", "AWSCodeDeployRole", "CloudWatchLogsFullAccess" and click the check box
------
- Create Role, service pick "CodeBuild" and click "Next"
- Search "AmazonEC2ContainerRegistryFullAccess","CloudWatchLogsFullAccess" and click the check box
- Click Next, Inut Tag Key and Value if you want
- Click Next to enter "Name" = "Workshop_CodeBuild"
- and "Description" = "Temp CodeBuild Role for Workshop " and click "Create" for the Role.
------

### Step 3: Launch a Cloud9 IDE for our workshop env 
* **AWS Console > Services > Cloud9 > Create Environment**
- Enter "Name" and "Description" for your Environment, and click Next Step
- Select "Create a new instance for environment (EC2)" for Environment Type
- Select "t3.micro" for Instance Type
- Select "Amazon Linux" for Platform, and click Next Step
- Input Tag Key and Value if you want, and click "Create Environment"

***Then we need to Attach the IAM Role onto this Cloud9 Environment***
- **AWS Console > Services > EC2 > Instances > Click the EC2 Instance for your Cloud9 > Actions > Instance Settings > Attach/Replace IAM Role > Choose the IAM Role we created in Step2**
- **Back to Cloud9 > Cloud9 Icon (Upper Left Corner) > Perference > AWS Settings > Credential > AWS Managed Temperory Credential > Off**
------

### Step 4: Setup Development Environment in your Cloud9
- After the IDE launch, click to "bash" tab in the botton of the IDE page
```
git clone https://github.com/awshktsa/AWSWorkshop-20200410-CodeCommit-CodePipeline-ECS-Fargate.git
cd AWSWorkshop-20200410-CodeCommit-CodePipeline-ECS-Fargate
aws codecommit create-repository --repository-name workshop-devops-ccef
```
And you will get an output like this:
```
{
    "repositoryMetadata": {
        "repositoryName": "workshop-devops-ccef", 
        "cloneUrlSsh": "ssh://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/workshop-devops-ccef", 
        "lastModifiedDate": 1586480711.24, 
        "repositoryId": "ad8c32e9-ffcc-4976-8573-0c2c6ca2c570", 
        "cloneUrlHttp": "https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/workshop-devops-ccef", 
        "creationDate": 1586480711.24, 
        "Arn": "arn:aws:codecommit:ap-northeast-1:384612698411:workshop-devops-ccef", 
        "accountId": "384612698411"
    }
}
```
Now we use following command to push workshop files into your new CodeCommit repository:
And please replace the <cloneUrlHttp> with your own repo url like "https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/workshop-devops-ccef"
```
git remote remove origin
git remote add origin <cloneUrlHttp>
git push origin master
```
Then we are going to initialize our ECR
```
docker build -t workshop-devops-ccef .

```
Then, create ECR with awscli:
```
sudo pip install --upgrade awscli
alias aws=/usr/local/bin/aws
aws ecr create-repository --repository-name workshop-devops-ccef
```
And you will see ECR result as:
```
{
    "repository": {
        "registryId": "384612698411", 
        "repositoryName": "workshop-devops-ccef", 
        "repositoryArn": "arn:aws:ecr:ap-northeast-1:384612698411:repository/workshop-devops-ccef", 
        "createdAt": 1586482684.0, 
        "repositoryUri": "384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef"
    }
}
```
And then please replace <repositoryUri> with your value, and <AWS_REGION> with where you are, us-east-1 as default.
```
aws ecr get-login-password --region <AWS_REGION> | docker login --username AWS --password-stdin <repositoryUri>
docker tag workshop-devops-ccef:latest <repositoryUri>:latest
docker push <repositoryUri>:latest

# Example
# aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef
# docker tag workshop-devops-ccef:latest 384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef:latest
# docker push 384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef:latest
```
------
### Step 5:
* Create Security Group, Target Group *2 and Application Loadbalancer *2.
* ***AWS Console > Services > EC2 > Security Group***
* Click ***Create Security Group***
- Name=Workshop_port_80, VPC=Default, Description=Temp Security Group for Workshop
- Inbound Rule > Add Rule > Type=HTTP, Source=Anywhere
------
* ***AWS Console > Services > EC2 > Load Balancers
* Click ***Create Load Balancer***
- Click "Application Load Balancer"
- Name=***alb-prod***, Scheme=internet-facing, IP Address Type=ipv4
- Load Balancer Protocol=HTTP, Load Balancer Port=80
- AZS > VPC=default, AZ=Select all
- Click Next to Security Group, Select "Workshop_port_80" 
- Click Next to Target Group, Select "New Target Group", Name=tg-prod, Target type=IP, Protocol=HTTP, Port=80
- Click to Review and Create
------
* ***AWS Console > Services > EC2 > Load Balancers***
* Click ***Create Load Balancer***
- Click "Application Load Balancer"
- Name=***alb-beta***, Scheme=internet-facing, IP Address Type=ipv4
- Load Balancer Protocol=HTTP, Load Balancer Port=80
- AZS > VPC=default, AZ=Select all
- Click Next to Security Group, Select "Workshop_port_80" 
- Click Next to Target Group, Select "New Target Group", Name=***tg-beta***, Target type=***IP***, Protocol=HTTP, Port=80
- Click to Review and Create
-------

### Step 6:
* Now we are going to create our ECS Fargate Services for our Beta & Prod Env
* ***AWS Console > Services > ECS***
* Clusters > Create Cluster > Type=Network Only (Fargate) > Cluster Name=fargate-prod
* Clusters > Create Cluster > Type=Network Only (Fargate) > Cluster Name=fargate-beta
* ***In this workshop, we try to simulate how we implement on real corp env, so we separate the services into two fargate clusters.***
------
* Task Definitions > Create New Task Definition > Launch Type=Fargate > Next 
* Task Definition Name=td-workshop-devops-ccef, Task Role=ecsTaskExecutionRole, Task Execution Role=ecsTaskExecutionRole
* Task Size > Memory=1GB, CPU=0.5vCPU
* Add Container > Container name=MyWebService,  Image=<repositoryUri>:latest, Soft Limit=1024, Port Mapping=80/TCP
Example --> Image = 384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef:latest)
* Click ADD then back to Task Definiton
------
* Now we are going to create servcie on fargate cluster
* Clusters > fargate-prod > Servcies > Create
* Launch Type=FARGATE, Task Defintion=MyWebService, Cluster=fargate-prod
* Service Name=***service-prod***, Service Type=REPLICA, Number of Tasks=2
* Deployments > Deployment Type=Blue/green deployment (powered by AWS CodeDeploy), Depolyment Configuration=CodeDeployDefault.ECSAllAtOnce, Service role for CodeDepoly=Workshop_CodeDeploy, Placement Templates=AZ Balanced Spread
* Click Next and start to setup VPC > VPC=default, Subnets=Select All, Security Group=***Workshop_port_80***, Health check grace period=30, Load balancer type=Application Load Balancer
* Load balancer name=***alb-prod***, choose the Container name:port=MyWebService:80 and click ***Add to load balancer
* Test Listener > unclick
* Target group 1 name > Select existed > tg-prod
* Target group 2 name > create new > use default like "tg-fargate-service-prod"
* Service discovery > unclick
* Click Next to Create
------
And also we will repeat this action to create a service for beta env.
* Clusters > fargate-beta > Servcies > Create
* Launch Type=FARGATE, Task Defintion=MyWebService, Cluster=fargate-prod
* Service Name=***service-beta***, Service Type=REPLICA, Number of Tasks=2
* Deployments > Deployment Type=Blue/green deployment (powered by AWS CodeDeploy), Depolyment Configuration=CodeDeployDefault.ECSAllAtOnce, Service role for CodeDepoly=Workshop_CodeDeploy, Placement Templates=AZ Balanced Spread
* Click Next and start to setup VPC > VPC=default, Subnets=Select All, Security Group=***Workshop_port_80***, Health check grace period=30, Load balancer type=Application Load Balancer
* Load balancer name=***alb-beta***, choose the Container name:port=MyWebService:80 and click ***Add to load balancer
* Test Listener > unclick
* Target group 1 name > Select existed > tg-beta
* Target group 2 name > create new > use default like "tg-fargate-service-beta"
* Service discovery > unclick
* Click Next to Create
------
**From Step.4 ~ Step.6, now we are able to create service on Amazon Elastic Container Service with AWS Fargate sytle.
------
### Step 7.
* Start to built our automation, we will leverage AWS CodePipeline to connect the service and flow.
* Here we will separate the flow into two pipeline CI & CD
* Now we build up the CI pipeline. 
* ***AWS Console > Services > CodePipeline > Create Pipeline***
* pipeline name=Code_to_Docker > Next
* Source Stage > Source Provider=CodeCommit, Repository Name=workshop-devops-ccef, Branch Name=master, Change detection options
=Amazon CloudWatch Events (recommended) > Next
* Build Stage > Build Provider=CodeDeploy
* Click Project > Project Name=Build_Code_To_ECR, Environment image=Managed image, Operating System=Ubuntu, Runtime=Standard, Image=aws/codebuild:2.0, Environment Type=Linux, ***Privileged (check)***
* Role Name=***Workshop_CodeBuild***
* Buildspec > select "Use a buildspec file" > Continue to Codepipeline
Then you will see an information *Successfully created Build_Code_To_ECR in CodeBuild.*
Before click to next stage, please click *add environment variable* and input following setting
- AWS_ACCOUNT_ID=&lt;your account Id&gt;
- AWS_DEFAULT_REGION=&lt;where you are&gt;
- IMAGE_REPO_NAME=devops-workshop-ccef
- IMAGE_TAG=latest
* Click Next and ***Skip Deploy Stage*** > Create
------
**Now Click "Release Change" and see how it goes**
* AWS Console > Services > Elastic Container Registry > workshop-devops-ccef
And you will see new Image been pushed onto ECR.
------
### Step.8
* Now we are going to build up CD pipeline
* ***AWS Console > Services > CodePipeline > Create Pipeline***
* pipeline name=ECR_to_ECS > Next
* * pipeline name=Code_to_Docker > Next
* Source Stage > Source Provider=CodeCommit, Repository Name=workshop-devops-ccef, Branch Name=master, Change detection options
=Amazon CloudWatch Events (recommended) > Next
* ***Skip Build Stage*** > Next
* Deploy Stage > Deploy Provider=Amazon ECS(Blue/Green), Application=AppECS-fargate-beta-service-beta, Deployment Group=Dgp-ECS-fargate-beta-service-beta, Amazon ECS task definition=SourceArtifact/taskdef.json, AWS CodeDeploy AppSpec file
=SourceArtifact/appspec.yaml
* Click Next > Create
------
Now we are not yet done on CD pipeline, for the Source provide we still lack the other half - ECR.
* Click Edit on ***ECR_to_ECS*** pipeline
* Then Edit on Source Stage, Click "Add Action"
* Action Name=Image, Action Provider=Amazon ECR, Repository Name=workshop-devops-ccef, Image tag=latest, Output Artifact=***IMAGE1***
* Click Done on Action and Source Stage
* Click Edit on Deploy Stage
* Edit CodeDeploy
* Add Input Artifact, and select ***IMAGE1***
* update Input artifact with image details=***IMAGE1***
* update Placeholder text in the task definition=IMAGE1_NAME
* Click Done on Action and Deploy Stage
------
**Now Try to click "Release Change"
------
### Step.8 (challenge)
* Click Edit on ***ECR_to_ECS*** pipeline
* Add a stage after deploy stage
* Select Manual Approvaal as provider
* Add a new stage after manual approval
* **optional: Create an SNS topic to receive approval notification, and also add a subscription for this approval**
* Repeat Step.7 to create a deployment toward prod Env (Application=AppECS-fargate-prod-service-prod)

### After Workshop -- Clean up
* ***clean up the 2 ECS-Fargate Cluster from AWS Console > Service > ECS***
* ***clean up 2 CodePipeline from AWS Console > Service > CodePipeline***
* ***Terminate the Cloud9 Env from AWS Console > Cloud9 > Environment > Delete***
* ***Remove 3 IAM Role from AWS Console > IAM > Role/User > Delete***
