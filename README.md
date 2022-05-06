# CI/CD for mono repo with SAM pipelines [WIP]


## Intro

This repo aims to provide a first implementation of  CI/CD pipelines for mono repo based on 
* Github repo
* [SAM pipelines](https://aws.amazon.com/blogs/compute/introducing-aws-sam-pipelines-automatically-generate-deployment-pipelines-for-serverless-applications/)
* AWS CodePipeline, CodeBuild etc.
* [Monorepo selective pipeline trigger](https://aws.amazon.com/blogs/devops/integrate-github-monorepo-with-aws-codepipeline-to-run-project-specific-ci-cd-pipelines/)

![mono repo trigger high level](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/04/23/Codepipeline.jpg)

![mono repo trigger flow](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/04/23/Codepipeline-Sample-Arch.jpg)

#TODO Add Multi account structure diagram with CI/CD account, staging and prod

## Components

* `pipeline-trigger`: the implementation of this [blog post](https://aws.amazon.com/blogs/devops/integrate-github-monorepo-with-aws-codepipeline-to-run-project-specific-ci-cd-pipelines/) to be able to trigger the right pipeline based on what changed
<!-- * `common-pipeline` an example on how you can potentially share the same pipeline config accross project -->
* `projectX`: the sub project folder containing the SAM app to deploy including it's pipeline (`codepipeline.yaml`, `pipeline/`, `.aws-sam/pipelineconfig.toml`), it's business logic code (`hello_world/`) and it's dependent infrastructure (`template.yaml`)




# Usage

## Prerequisite

* [SAM cli](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
* A github account
* one or more AWS Accounts (preferably 3: one for CI/CD pipeline hosting, one for staging/QA and one for Prod)


1. Fork repo: https://github.com/flochaz/sam-pipelines-monorepo/fork
   ```
   git clone https://github.com/<YOUR ALIAS>/sam-pipelines-monorepo.git
   cd sam-pipelines-monorepo
   ```
1. Setup your different aws profile for each account. In this instruction
* `dev` profile will point to `dev` account, 
* `cicd` profile to the account hosting the pipeline trigger and all the sub project pipeline
* `staging` profile for the first stage of deployment of the projects pipelines
* `prod` profile for the second stage of deployment



## Setup Mono repo pipeline trigger (deployed in ci/cd account) 

```
cd pipeline-trigger
sam deploy --guided --profile cicd
```

## Use projectA example

```
cd projectA
sam pipeline bootstrap
sam deploy -t codepipeline.yaml --stack-name projectA-pipeline --capabilities=CAPABILITY_IAM --profile cicd
```

## Setup new subproject

```
sam init projectX
cd projectX
sam pipeline init --bootstrap
# Add Name to pipeline "projectX" and fix path to yamls 
# Disable change detection #TODO
sam deploy -t codepipeline.yaml --stack-name projectX-pipeline --capabilities=CAPABILITY_IAM --profile cicd
```



