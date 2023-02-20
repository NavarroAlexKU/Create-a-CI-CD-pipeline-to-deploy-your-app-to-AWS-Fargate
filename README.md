# Create-a-CI-CD-pipeline-to-deploy-your-app-to-AWS-Fargate
![ScreenShot](https://d1.awsstatic.com/Projects/CICD%20Pipeline/setup-cicd-pipeline2.5cefde1406fa6787d9d3c38ae6ba3a53e8df3be8.png)

## 🔗 Contact Information
[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/alexnavarro2/)
[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](https://mail.google.com/mail/u/0/#inbox?compose=GTvVlcSBpRjxKKJtxTLNxwpsKvpfbRSRnRLcTQRMZLcKCNfrJjXfcNNKPmstkbHJpzHGNZnHvhCph)

## Configure AWS CodeCommit:
![ScreenShot](https://d1.awsstatic.com/products/codecommit/Product-Page-Diagram_AWS-CodeCommit%20(1)2.b33016d587d4c7aa6f132753294929982e2648b4.png)
* Create configuration and specification files that define the code pipeline and a new task definition.
* Edit the source code to change the application's background color and save everything to a CodeCommit repository.

### Configure CodeCommit as a source control repository:
* Open new terminal and check files we will push to AWS CodeCommit:
```
ls ~/environment
```

Executing the above command in the terminal displays the following files:
- Dockerfile
- index.js
- package.json
- routes
- static

```
******************************
**** This is OUTPUT ONLY. ****
******************************

Dockerfile  index.js  package.json  routes  static
```

* You can view the contents of the Dockerfile by executing the following command in the terminal:
```
cat Dockerfile
```

To launch this application on Amazon ECS and build a pipeline that automates its deployment: we need three additional files:
 - buildspec.yaml: CodeBuild uses the commands and parameters in the buildspec file to build a Docker image. "You must have a buildspec.yml file at the root of your source code"
 - appspec.yaml: CodeDeploy uses the appspec file to select a task definition.
 - taskdef.json: After updating the application source code and building a new container, we need a second task definition that points to it. The taskdef.json file is used to create this new task definition that points to our updated application image.
 
Breakdown of buildspec.yaml file:
* You can define environment varianbles:
    - Plaintext variables
    - Secure secretrs using the SSM Parameter Store.
* Phases:
    - Install: install dependencies you may need for the build.
    - Pre-build: final commands to execute before build.
    - Build: Actual build commands
    - Post Build: Finishing touches (e.g. zip file output).
* Artifacts:
    - These get uploaded to Amazon S3 storage (encrypted with KMS).
* Cache: files to cache (usually dependencies) to Amazon S3 storage for future builds.


 ```
 cat << 'EOF' > ~/environment/buildspec.yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - ACCOUNT_ID=$(echo $CODEBUILD_BUILD_ARN | cut -f5 -d ':') && echo "The Account ID is $ACCOUNT_ID"
      - echo "The AWS Region is $AWS_DEFAULT_REGION"
      - REPOSITORY_URI=$ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ACCOUNT_ID-application
      - echo "The Repository URI is $REPOSITORY_URI"
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $REPOSITORY_URI
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=$COMMIT_HASH
  build:
    on-failure: ABORT
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
      - docker tag $REPOSITORY_URI:$IMAGE_TAG $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":”myimage","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
      - printf '{"ImageURI":"%s"}' $REPOSITORY_URI:$IMAGE_TAG > imageDetail.json
artifacts:
    files: 
      - imagedefinitions.json
      - imageDetail.json
      - appspec.yaml
      - taskdef.json
EOF
```

Looking at the code above:
* Version 0.2 is referencing the Buildspec version I'm using.
* pre_build phase is optional phase that is used to run commands before building the application code. For this specific example, the pre_build phase is used to set variables that are used throughout the build process and authenticate into Amazon ECR.
* Each newly built image is tagged with the corresponding "commit ID" from AWS CodeCommit.
* The commands included in the build phase are ran sequentially. "ABORT" command has been included to end the build if any of the commands fail.
* post_build phase pushes the Docker image produced in the build phase to Amazon ECR.
* Upon completion, the build process artifacts called "imageDetail.json" and "imagedefinitions.json", both of which are saved to the env root directory. These files are used in the deploy phase of the code pipeline and indicate which image to deploy to Amazon ECS.
* The artifacts sections specifies that the appspec.yaml and taskdef.json files uploaded to your CodeCommit repo be included as build outputs. Without these files, the deployment will fail.

### Artifacts Commands:
Create appspec.yaml file:
```
cat << EOF > ~/environment/appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "application"
          ContainerPort: 80
EOF
```

Create taskdef.json file:
```
cat << EOF > ~/environment/taskdef.json
{
    "containerDefinitions": [
        {
            "name": "application",
            "image": "<IMAGE_NAME>",
            "portMappings": [
                {
                    "containerPort": 80,
                    "hostPort": 80,
                    "protocol": "tcp"
                }
            ],
            "essential": true,
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "cicd-logs",
                    "awslogs-region": "$AWS_REGION",
                    "awslogs-stream-prefix": "ecs"
                },
            },
        }
    ],
    "family": "$FAMILY",
    "taskRoleArn": "arn:aws:iam::$ACCOUNT_ID:role/ecsTaskExecutionRole",
    "executionRoleArn": "arn:aws:iam::$ACCOUNT_ID:role/ecsTaskExecutionRole",
    "networkMode": "awsvpc",
    "status": "ACTIVE",
    "compatibilities": [
        "EC2",
        "FARGATE"
    ],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "256",
    "memory": "512",
    "tags": [
        {
            "key": "Name",
            "value": "GreenTaskDefinition"
        }
    ]
}
EOF
```
The application I'm modifying is a website that currently has a blue background. I'm going to modify the application code to make the background green.
to change the application background color to green use the following command:
```
sed -i 's/282F3D/1D8102/g' ~/environment/static/css/app.css
```

Now that the application code has been updated, I will create a new AWS CodeCommit repo and push all the files into it.
```
export SRC_REPO_URL=$( \
    aws codecommit create-repository \
        --repository-name pipeline-source-code \
        --repository-description "Repository for AWS News application source code" \
        | jq -r '.[].cloneUrlSsh'
)
echo "Repo successfully created. Use $SRC_REPO_URL to clone the repository"
```

Now Update Git configuration:
```
git config --global init.defaultBranch main
```

Initialize the envronment directory as a local Git repo:
```
cd ~/environment
git init
```

Stage the application files, commit theupdated files to the local repo, and push the application code to themain branch in AWS CodeCommit repo:
```
git add .
git commit -m "initial commit"
git push --set-upstream $SRC_REPO_URL main
```

### Create a CodeDeploy application and deployment group:
![ScreenShot](https://docs.aws.amazon.com/images/codedeploy/latest/userguide/images/deployment-process-ecs.png)

* Application:
    - An application is a name that uniquely identifies the code that you want to deploy. AWS CodeDeploy uses the application to ensure that the correct combination of revision, deployment configuration, instances, and Auto Scaling Groups are referenced when the pipeline is invoked.
* Deployment Groups:
    - AWS CodeDeploy uses deployment groups to specify the Amazon ECS service, load balancer, and target groups for your revised application code. They also include configuration details, such as how and when traffic should be rerouted to the new tasks that your pipeline creates.

Create Application:
* AWS CodeDeploy console:
    - Go to Developer Tools left side of screen:
    - Deploy
    - CodeDeploy and choose Applications


