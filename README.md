# Create-a-CI-CD-pipeline-to-deploy-your-app-to-AWS-Fargate
![ScreenShot](https://d1.awsstatic.com/Projects/CICD%20Pipeline/setup-cicd-pipeline2.5cefde1406fa6787d9d3c38ae6ba3a53e8df3be8.png)

## 🔗 Contact Information
[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/alexnavarro2/)
[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](https://mail.google.com/mail/u/0/#inbox?compose=GTvVlcSBpRjxKKJtxTLNxwpsKvpfbRSRnRLcTQRMZLcKCNfrJjXfcNNKPmstkbHJpzHGNZnHvhCph)

## Configure AWS CodeCommit:
![ScreenShot](https://d1.awsstatic.com/products/codecommit/Product-Page-Diagram_AWS-CodeCommit%20(1)2.b33016d587d4c7aa6f132753294929982e2648b4.png)
* Create configuration and specification files that define the code pipeline and a new task definition.
* Edit the source code to change the application's background color and save everything to a CodeCommit repository.

### AWS Cloud9:
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
 - buildspec.yaml: CodeBuild uses the commands and parameters in the buildspec file to build a Docker image.
 - appspec.yaml: CodeDeploy uses the appspec file to select a task definition.
 - taskdef.json: After updating the application source code and building a new container, we need a second task definition that points to it. The taskdef.json file is used to create this new task definition that points to our updated application image.