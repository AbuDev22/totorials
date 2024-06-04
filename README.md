### Say Goodbye to Manual Deployments: Automate Your EC2 Deployments with CodeDeploy and GitHub Actions

Few concepts need to know before dive into actual solution. I am considering everyone must be aware about Amazon EC2.

CodeDeploy - CodeDeploy is an Amazon deployment service that automates application deployments to Amazon EC2 instances, on-premises instances, serverless Lambda functions, or Amazon ECS services.

GitHub Actions - GitHub Actions is a continuous integration tool that operates based on your GitHub code lifecycle, such as code push, commit, PR merge, branch creation, and tag creation. You can utilize this hook to trigger other actions like Jenkins jobs, AWS services, and more.

***Prerequisites***
1.	AWS Account: Ensure you have an AWS account.
2.	IAM User with Required Permissions: Create an IAM user with permissions for CodeDeploy, EC2, and S3 (if necessary).
3.	GitHub Repository: Have your code repository set up on GitHub.
4.	EC2 Instance: Have an EC2 instance set up and running.
5.	Docker Hub Account: Have a Docker Hub account for hosting your Docker images.


## Step 1: Set Up EC2 Instance
1.	Launch EC2 Instance:
•	Open the Amazon EC2 console.
•	Choose "Launch Instance."
•	Select an Amazon AMI(e.g., Amazon Linux 2).
•	Choose an instance type, Eg. t2.micro.
•	Configure instance details, add storage, add tags, and configure security group (allow SSH and HTTP/HTTPS).

2.	**Add User Data Script:** Add the following user data script to install Docker and AWS CodeDeploy agent:
To add the user data, click on Advance details and scroll to the bottom to add your user data.
 
Copy the code below into the user data section to install docker and Codedeploy agent during the instance launch process
```sh
#!/bin/bash
# Update the package list and install necessary packages
sudo yum update -y
sudo yum install -y ruby wget

# Install Docker
sudo yum install docker -y

sudo usermod -aG docker ec2-user
sudo service docker start

# Enable Docker to start on boot
sudo systemctl enable docker

# Install AWS CodeDeploy Agent
cd /home/ec2-user
wget https://aws-codedeploy-us-west-2.s3.us-west-2.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto

# Start the CodeDeploy agent
sudo service codedeploy-agent start

# Enable CodeDeploy agent to start on boot
sudo systemctl enable codedeploy-agent

# Print versions to verify installation
docker --version
codedeploy-agent --version

```

3.	Launch the Instance: Review and launch the instance. Ensure you download the key pair (.pem file) for SSH access.
## Step 2: Create IAM Role for CodeDeploy
1.	Open the IAM Console:
•	Navigate to the AWS Management Console.
•	Go to the IAM (Identity and Access Management) service.
2.	Create a New Role:
•	In the left navigation pane, select "Roles."
•	Click on the "Create role" button.
3.	Select Trusted Entity:
•	Choose the type of trusted entity as "AWS service."
•	Under "Use case," select "EC2" and click "Next: Permissions."
4.	Attach Permissions Policies:
•	Search for and select the following policies:
•	AWSCodeDeployFullAccess
•	AmazonS3ReadOnlyAccess (if your CodeDeploy application accesses S3 for deployment packages)
For our own we will not be needing this since we not using s3
•	Click "Next: Tags" (you can add tags if needed) and then "Next: Review."
5.	Name the Role:
•	Enter a role name (e.g., EC2CodeDeployRole).
•	Add a description if desired.
•	Click "Create role."
## Step 3: Attach IAM Role to EC2 Instance
1.	Open the EC2 Console:
•	Navigate to the AWS Management Console.
•	Go to the EC2 service.
2.	Select the Instance:
•	In the left navigation pane, select "Instances."
•	Find and select the instance you want to add the IAM role to.
3.	Attach IAM Role:
•	With your instance selected, click the "Actions" button.
•	Navigate to "Security" > "Modify IAM role."
•	In the "IAM role" dropdown, select the role you created earlier (e.g., EC2CodeDeployRole).
•	Click "Update IAM role."
4.	After attaching your role, Reboot your instance to take effect.

## Step 4: Configure AWS CodeDeploy
1.	Create CodeDeploy Application:
•	Open the AWS CodeDeploy console.
•	Choose "Create application."
•	Enter an application name (e.g., todoapp).
•	Choose the compute platform as "EC2/On-premises."
2.	Create Deployment Group:
•	Choose the application you created.
•	Choose "Create deployment group."
•	Enter a name for the deployment group (e.g., todoapp-deployment-group).
•	Select the service role with CodeDeploy permissions.
•	In "Deployment type," select "In-place."
 


•	In "Environment configuration," choose "Amazon EC2 instances" and select your EC2 instances using tags.
First choose Name from the Key section and select your instance in the value tab.

 








•	Uncheck enable load balancing settings .

 

•	Click "Create deployment group."



Step 5: Prepare GitHub Repository
1.	Create Deployment Scripts:
•	In your GitHub repository, create the following directory structure:
 
Create a directory named “scripts” in your root directory

2.	Add Deployment Scripts:
•	scripts/before_install.sh:
```sh
#!/bin/bash
docker pull abudev22/todoapp:latest
docker stop my-website || true
docker rm my-website || true
```
make sure to replace with your docker username and image name



•	deploy/scripts/start_server.sh:
```sh
#!/bin/bash
docker run -d --name my-website -p 8000:8000 abudev22/todoapp:latest

```


3.	create an AppSpec File in your root directory:
•	appspec.yml:
```sh
version: 0.0
os: linux
files:
  - source: /
    destination: /home/ec2-user/deployment
hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: ec2-user
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: ec2-user

```





## Step 6: Set Up GitHub Actions Workflow
1.	Create Workflow File:
•	In your GitHub repository, create the following directory structure:
.github/ workflows/ main.yml 
2.	Add Workflow Configuration:
Make sure to change the application name and application group to be the same as the one you created in  step 4.
main.yml
```sh
name: Deploy website to EC2 using AWS CodeDeploy

on:
  push:
    branches:
      - devops

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_HUB_USERNAME }}
        password: ${{ secrets.DOCKER_HUB_PASSWORD }}

    - name: Build and push Docker image
      run: |
        docker build -t ${{ secrets.DOCKER_HUB_USERNAME }}/todoapp .
        docker push ${{ secrets.DOCKER_HUB_USERNAME }}/todoapp:latest
   

    - name: Deploy to AWS CodeDeploy
      run: |
        aws deploy create-deployment \
          --application-name todoapp \
          --deployment-group-name todoapp-deployment-group \
          --github-location repository=${{ github.repository }},commitId=${{ github.sha }}      
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_REGION: us-east-2
```

## Step 7: Set Up GitHub Secrets
1.	Add GitHub Secrets:
•	In your GitHub repository, go to "Settings" > "Secrets" > "Actions."
•	Add the following secrets:
•	DOCKER_HUB_USERNAME: Your Docker Hub username.
•	DOCKER_HUB_PASSWORD: Your Docker Hub password.
•	AWS_ACCESS_KEY_ID: Your AWS access key ID.
•	AWS_SECRET_ACCESS_KEY: Your AWS secret access key.
## Step 8: Push Changes to GitHub
1.	Push Changes:
•	Commit and push your changes to the main branch in your GitHub repository.
2.	Monitor Deployment:
•	Monitor the GitHub Actions workflow for any issues.
 
•	Monitor the AWS CodeDeploy console for the deployment status.
 

By following these steps, you will have a completed setup for deploying your application from GitHub to EC2 using AWS CodeDeploy.



Step 9: Confirm the App has been deployed
1.	Ssh into your instance using your pem file
2.	Execute docker ps to confirm the container has been deployed on the instance.
 
3.	If you can see it there, then the deployment was a success.

Here is a diagram for the workflow.
 

