# terraform-aws-codepipline
Terraform module which creates CodePipeline resources

The following resources will be created:

Example 1 Basic build pipline

- requires a buildspec.yml in the branch root that follows the buildspec.yml syntax. 
  https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html

``` hcl
provider "aws" {
    region = "us-east-1"
}

module "basic_codepipeline" {
      source                 = "../../"
      version                = "1.0.0"
      
      project_name                = "example-project"
    
      provider_type               = "GitHub"
    
      build_environment_variables = [{
        name  = "AWS_BUCKET"
        value = "example.project.com"
        type = "PLAINTEXT"
     }]
    
      full_repository_id          = "github-user/example-project"
    
      branch_name                 = "main"
    
      enable_codestar_notifications = true
    
      tags = {
        Name = "example-project"
      }
}
```

Example 2 Basic build pipline sourcing build env vars from parameter store

- requires a buildspec.yml in the branch root that follows the buildspec.yml syntax.
  https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html

``` hcl
provider "aws" {
    region = "us-east-1"
}

module "basic_codepipeline_sourcing_env_vars_from_ssm_parameter_store" {
      source                 = "../../"
      version                = "1.0.0"
      
      project_name                = "example-project"
    
      provider_type               = "GitHub"
    
      build_environment_variables = [{
        name  = "AWS_BUCKET"
        value = "/exmample-account/example-project/name"
        type = "PARAMETER_STORE"
      }]
    
      full_repository_id          = "github-user/example-project"
    
      branch_name                 = "main"
    
      enable_codestar_notifications = true
    
      tags = {
        Name = "example-project"
      }
}
```

Example build spec yml for a static javascript site
``` yaml
version: 0.2
phases:
  install:
    runtime-versions:
      nodejs: latest
  pre_build:
    commands:
      - npm install
  build:
    commands:
      - npm run build
  post_build:
    on-failure: ABORT
    commands:
      - aws s3 sync public/ s3://${AWS_BUCKET}/ --delete --cache-control max-age=31536000,public
artifacts:
  base-directory: public
  files:
    - "**/*"
  discard-paths: yes
```

Example 3 Build [x86_64 (amd64)] container image and push to AWS ecr:
``` hcl
provider "aws" {
    region = "us-east-1"
}

module "build_container_and_push_to_ecr" {
  source                 = "../../"
  version                = "1.0.0"
  
  project_name                = format("contact-me-greeting-%s", local.account_vars.tags.Environment)

  enable_container_features   = true

  provider_type               = "GitHub"

  build_environment_variables = [{
      name  = "AWS_DEFAULT_REGION"
      value = "us-east-1"
      type = "PLAINTEXT"
    },
    {
      name  = "AWS_ACCOUNT_ID"
      value = get_aws_account_id()
      type = "PLAINTEXT"
    },
    {
      name  = "IMAGE_REPO_NAME"
      value = "example-ecr-repo-name"
      type = "PLAINTEXT"
    },
    {
      name  = "IMAGE_TAG"
      value = "latest"
      type = "PLAINTEXT"
    }]

    full_repository_id          = "github-user/example-project"
  
    branch_name                 = "main"
  
    enable_codestar_notifications = true

  tags = merge(
    local.account_vars.tags,
    {
      "Product"     = "contact-me"
    }
  )
}
```

Example build spec yml for a docker file build
``` yaml
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
```
