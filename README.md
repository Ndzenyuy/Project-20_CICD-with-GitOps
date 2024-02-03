# Project 20: GitOps CICD with Github Actions and Kubernetes

This project creates a pipeline that provisions an infrastructure in EKS using IaC with terraform

## Architecture

![Architecture]()

## Prepare Github Repositories

This project uses code stored in two repositories to build first an infrastructure on AWS comprised of a VPC, public and private subnets and a nat gatway, the second deploys the app on the created infrastructure.

### Setting up IAC

Fork the repositories on your github
```

Infrastructure: https://github.com/Ndzenyuy/Project-20_CICD-with-GitOps

Web app deployment: https://github.com/Ndzenyuy/rhena-actions

```
- Create a keypair and connect to Github
Open terminal and run the following
```
ssh-keygen
```
On the prompt, enter the path and name of the keypair to be created
```
/home/jones/.ssh/rhena_actions
```
Type enter for the next prompts. Now open the public key of the created key and copy it to github
```
cat /home/jones/.ssh/rhena_actions.pub
```
Open github -> Profile icon -> Settings -> SSH and GPG keys -> New SSH key

```
Title: rhenaactions
key type: Authentication Key
key: paste the content copied from above steps
```
Now to your terminal, run the command
```
export GIT_SSH_COMMAND="ssh -i ~/.ssh/rhena_actions"
```
configure global user name and email
```
git config --global user.name <YOUR_GITHUB_USERNAME>
git config --global user.email <YOUR GITHUB EMAIL>
```
- Setup project Directory on local machine
Now create a folder rhenaApp and cd into it

```
mkdir -p ~/projects/rhenaApp
cd ~/projects/rhenaApp
```
Then clone the forked repositories, enter the url for your own repositories
```
git clone https://github.com/Ndzenyuy/Project-20_CICD-with-GitOps
git clone https://github.com/Ndzenyuy/rhena-actions
```

Now setup the ssh keys created earlier to be used by this repo

```git
cd Project-20_CICD-with-GitOps/
git config core.sshCommand "ssh -i ~/.ssh/rhena_actions -F /dev/null"
cd ../rhena-actions
git config core.sshCommand "ssh -i ~/.ssh/rhena_actions -F /dev/null"
```

In the Project-20_CICD-with-GitOps folder, we have to checkout to the staging branch, every changes are done in the staging branch before pushing to main.

```git
cd ../Project-20_CICD-with-GitOps
git checkout stage
```

## GitHub secrets

Here we are going to create and store AWS access keys, S3 bucket, ECR repository and store in Github secrets.

### Create IAM user
Go to AWS console and create an admin user 
IAM -> User -> create user -> attach policies -> administrator access -> create access keys

On github, Project-20_CICD-with-GitOps repository go to settings -> Secrets and variables -> actions -> new repository secret and store the following keys
```
- Name: AWS_ACCESS_KEY_ID,  Secret: <AWS ACCESS KEYS>
- Name: AWS_SECRET_ACCESS_KEY: secret: <AWS SECRET KEY>

```

Do the same for the repository rhena-actions, store both secrets with the same name.

### Create S3 bucket

Goto AWS console, and create an S3 bucket, we can call it "rhenaactions"

Back to Github secrets(Project-20_CICD-with-GitOps), create a new secret having the bucket name: 
```
Name: BUCKET_TF_STATE, secret: rhenaactions
```

### Create ECR repository

On to AWS console, Elastic Container Registry -> Create repository -> name: rhenaapp -> create repository. 
Copy the repository URI(XXXXXXXXX.dkr.ecr.us-east-2.amazonaws.com/rhenaapp)

Back to github secrets (rhena-actions), Create another secret
```
Name: REGISTRY, secret: XXXXXXXXX.dkr.ecr.us-east-2.amazonaws.com

```

## Terraform code adjustments

In the Project-20_CICD-with-GitOps, open it in VSCode and move to the folder terraform, modify the contents of terraform.tf to be 

```tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.25.0"
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.5.1"
    }

    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0.4"
    }

    cloudinit = {
      source  = "hashicorp/cloudinit"
      version = "~> 2.3.2"
    }

    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23.0"
    }
  }

  backend "s3" {
    bucket = "<YOUR BUCKET NAME"
    key    = "terraform.tfstate"
    region = "us-east-2"
  }
  required_version = "~> 1.7.0"

}

```
Make sure you change the bucket name to the one you created

## Github actions workflow

In this workflow terraform will not apply the code for the infrastructure to be created(staging), but when a pull request is made, the pipeline is triggered and this time and the pipeline get applied(main), we need to put conditions.
In the file .github/workflow/terraform.yml, we should have the code

```yml
name: "Rhena IaC"
on:
    push:
        branches:
            - main
            - stage
        paths:
            - terraform/**
    pull_request:
        branches:
            - main
        paths:
            - terraform/**

env:
    # Credentials for deployment to AWS
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    # S3 bucket for the Terraform state
    BUCKET_TF_STATE: ${{ secrets.BUCKET_TF_STATE }}
    AWS_REGION: us-east-2
    EKS_CLUSTER: rhena-eks

jobs:
    terraform:
        name: "Apply terraform code changes"
        runs-on: ubuntu-latest
        defaults:
            run:
                shell: bash
                working-directory: ./terraform

        steps:
            - name: Checkout source code
              uses: actions/checkout@v4           
            
            - name: Setup Terraform with specified version on the runner
              uses: hashicorp/setup-terraform@v2
              #with:
              #  terraform_version: 1.6.3
            
            - name: Terraform init
              id: init
              run: terraform init -backend-config="bucket=$BUCKET_TF_STATE"

            - name: Terraform format
              id: fmt
              run: terraform fmt -check
            
            - name: Terraform validate
              id: validate
              run: terraform validate

            - name: Terraform plan
              id: plan
              run: terraform plan -no-color -input=false -out planfile
              continue-on-error: true

            - name: Terraform plan status
              if: steps.plan.outcome == 'failure'
              run: exit 1

            - name: Terraform Apply
              id: apple
              if: github.ref == 'refs/heads/main' && github.event_name == 'push'
              run: terraform apply -auto-approve -input=false -parallelism=1 planfile

            - name:  Configure AWS credentials
              uses: aws-actions/configure-aws-credentials@v1
              with: 
                aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
                aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
                aws-region: ${{ env.AWS_REGION }}
            
            - name: Get Kube config file
              id: getconfig
              if: steps.apple.outcome == 'success' 
              run: aws eks update-kubeconfig --region ${{ env.AWS_REGION }} --name ${{ env.EKS_CLUSTER }}

            - name: Install Ingress controller
              if: steps.apple.outcome == 'success' && steps.getconfig.outcome == 'success'
              run: kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.3/deploy/static/provider/aws/deploy.yaml


```

## App workflow

The next workflow will use the infrastructure created from the previous job, then deploy the application with best practices.

 - Create an organization in sonarcloud

 Open the browser and navigate to <www.sonar.io>

## Deployment to EKS

## Clean up
