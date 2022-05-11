# CI/CD for mono repo with SAM pipelines [WIP]


## Intro

This repo aims to provide a first implementation of  CI/CD pipelines for mono repo based on 
* Github repo
* Multi Accounts (CI/CD dedicated account, Staging account and Prod account)
* [SAM pipelines](https://aws.amazon.com/blogs/compute/introducing-aws-sam-pipelines-automatically-generate-deployment-pipelines-for-serverless-applications/)
* AWS CodePipeline, CodeBuild etc.
* [Monorepo selective pipeline trigger](https://aws.amazon.com/blogs/devops/integrate-github-monorepo-with-aws-codepipeline-to-run-project-specific-ci-cd-pipelines/)

Pipeline trigger high level view:

![mono repo trigger high level](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/04/23/Codepipeline.jpg)

Pipeline trigger lower level view:

![mono repo trigger flow](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/04/23/Codepipeline-Sample-Arch.jpg)

Multi accounts pipeline high level view:

#TODO Add Multi account structure diagram with CI/CD account, staging and prod

## Components

* `pipeline-trigger`: the implementation of this [blog post](https://aws.amazon.com/blogs/devops/integrate-github-monorepo-with-aws-codepipeline-to-run-project-specific-ci-cd-pipelines/) to be able to trigger the right pipeline based on what changed
* `common-pipeline` an example on how you can potentially share the same pipeline config accross project
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

1. Deploy the pipeline trigger
   ```
   cd pipeline-trigger
   sam deploy --guided --profile cicd
   ```
1. Add webhook to your github repo (see "Creating a GitHub webhook" [this blog](https://aws.amazon.com/blogs/devops/integrate-github-monorepo-with-aws-codepipeline-to-run-project-specific-ci-cd-pipelines/) for more details making sure you selected `Content type` as `application/json` !!!)

## Use projectA example

1. (optional) Deploy projectA to your dev account. This will deploy your workload infrastructure
   ```
   cd projectA
   sam deploy --guided --profile dev
   ```
1. Bootstrap your accounts
   1. create a dummy user into CI/CD account (to work around https://github.com/aws/aws-sam-cli/issues/3857)
      ```bash
      aws iam create-user --user-name "dummy-aws-sam-pipeline-user" --profile cicd
      ```
   1. Bootstrap your first stage account account: `test`
      ```bash
      sam pipeline bootstrap --pipeline-user arn:aws:iam::<CICD ACCOUNT ID>:user/dummy-aws-sam-cli-user --stage test
      ```
      Make sure to select the right profile to access this test account
   1. Bootstrap your second stage account: `prod`
      ```bash
      sam pipeline bootstrap --pipeline-user arn:aws:iam::<CICD ACCOUNT ID>:user/dummy-aws-sam-cli-user --stage prod
      ```
      Make sure to select the right profile to access this test account
   1. Create your pipeline infrastructure:
      ```
      sam pipeline init                                                                                                                   
      ```
      Make sure to select
      * first ***"1 - AWS Quick Start Pipeline Templates"***
      * then ***"5 - AWS CodePipeline"***

1. Update your pipeline for Mono repo support:
   1. Update `codepipeline.yaml`
      1. Add a cloudformation parameter to be able to adapt to your subprojects
         ```diff
         diff --git a/codepipeline.yaml b/codepipeline.yaml
         index 24edac8..49d581d 100644
         --- a/codepipeline.yaml
         +++ b/codepipeline.yaml
         @@ -27,12 +27,14 @@ Description: >
         
         
         Parameters:
         +  SubFolderName:
         +    Type: String
            GitProviderType:
            Type: String
            Default: "GitHub"
         ```
      1. Name your pipeline using this parameter
         ```diff
         @@ -116,6 +118,7 @@ Resources:
            Pipeline:
            Type: AWS::CodePipeline::Pipeline
            Properties:
         +      Name: !Sub ${SubFolderName}
               ArtifactStore:
                  Location: !Ref PipelineArtifactsBucket
                  Type: S3
         @@ -155,7 +159,8 @@ Resources:
                        ParameterOverrides: !Sub |
                           {
                              "FeatureGitBranch": "${FeatureGitBranch}",
         -                    "CodeStarConnectionArn": "${CodeStarConnectionArn}"
         +                    "CodeStarConnectionArn": "${CodeStarConnectionArn}",
         +                    "SubFolderName": "${SubFolderName}"
                           }
                        InputArtifacts:
                        - Name: SourceCodeAsZip
         ```
      1. Disable auto build on code change since we are delegating this logic to pipeline-trigger
         ```diff
         @@ -134,6 +137,7 @@ Resources:
                        ConnectionArn: !If [CreateConnection, !Ref CodeStarConnection, !Ref CodeStarConnectionArn]
                        FullRepositoryId: !Ref FullRepositoryId
                        BranchName: !If [IsFeatureBranchPipeline, !Ref FeatureGitBranch, !Ref MainGitBranch]
         +                DetectChanges: false
         ```
      1. Update buildspec locations:
         ```diff
         @@ -548,7 +552,7 @@ Resources:
            #     ServiceRole: !GetAtt CodeBuildServiceRole.Arn
            #     Source:
            #       Type: CODEPIPELINE
         -  #       BuildSpec: pipeline/buildspec_unit_test.yml
         +  #       BuildSpec: !Sub ${SubFolderName}/pipeline/buildspec_unit_test.yml
         
            CodeBuildProjectBuildAndDeployFeature:
            Condition: IsFeatureBranchPipeline
         @@ -579,7 +583,7 @@ Resources:
               ServiceRole: !GetAtt CodeBuildServiceRole.Arn
               Source:
                  Type: CODEPIPELINE
         -        BuildSpec: pipeline/buildspec_feature.yml
         +        BuildSpec: !Sub ${SubFolderName}/pipeline/buildspec_feature.yml
         
            CodeBuildProjectBuildAndPackage:
            Condition: IsMainBranchPipeline
         @@ -614,7 +618,7 @@ Resources:
               ServiceRole: !GetAtt CodeBuildServiceRole.Arn
               Source:
                  Type: CODEPIPELINE
         -        BuildSpec: pipeline/buildspec_build_package.yml
         +        BuildSpec: !Sub ${SubFolderName}/pipeline/buildspec_build_package.yml
         
            # Uncomment and modify the following step for running the integration tests
            # CodeBuildProjectIntegrationTest:
         @@ -630,7 +634,7 @@ Resources:
            #     ServiceRole: !GetAtt CodeBuildServiceRole.Arn
            #     Source:
            #       Type: CODEPIPELINE
         -  #       BuildSpec: pipeline/buildspec_integration_test.yml
         +  #       BuildSpec: !Sub ${SubFolderName}/pipeline/buildspec_integration_test.yml
         
            CodeBuildProjectDeploy:
            Condition: IsMainBranchPipeline
         ```
   1. Move buildspecs to your project folder
      ```
      mv pipeline projectA/pipeline
      ```
   1. Update buildspec to work in the right folder
      ```diff
      diff --git a/projectA/pipeline/buildspec_build_package.yml b/projectA/pipeline/buildspec_build_package.yml
      index 537d1d9..bfa6e18 100644
      --- a/projectA/pipeline/buildspec_build_package.yml
      +++ b/projectA/pipeline/buildspec_build_package.yml
      @@ -4,6 +4,7 @@ phases:
         runtime-versions:
            python: 3.8
         commands:
      +      - cd projectA
            - pip install --upgrade pip
            - pip install --upgrade awscli aws-sam-cli
            # Enable docker https://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker-custom-image.html
      @@ -21,6 +22,7 @@ phases:
                           --region ${PROD_REGION}
                           --output-template-file packaged-prod.yaml
      artifacts:
      +  base-directory: projectA
         files:
         - packaged-test.yaml
         - packaged-prod.yaml
      ```
   1. Commit and push to git
      ```
      git add .
      git commit -m "Add pipeline for projectA"
      ```
1. Deploy the pipeline
   ```
   PROJECT_SUB_FOLDER_NAME=projectA sam deploy -t codepipeline.yaml --stack-name ${PROJECT_SUB_FOLDER_NAME}-pipeline  --parameter-overrides "SubFolderName=${PROJECT_SUB_FOLDER_NAME}" --capabilities=CAPABILITY_IAM --profile cicd
   ```
1. Activate the github connection
   1. Go to your CI/CD account console
   1. Got to [Developer tools settings](https://console.aws.amazon.com/codesuite/settings/connections) (if you don't see your connection, check the region you are in)
1. push a new change modifying/creating anything in `projectA` 
1. see it flow through the `projectA` pipeline in CI/CD account

## Setup new subproject

```
sam init projectX
cd projectX
sam pipeline init --bootstrap
# Add Name to pipeline "projectX" and fix path to yamls 
# Disable change detection #TODO
sam deploy -t codepipeline.yaml --stack-name projectX-pipeline --capabilities=CAPABILITY_IAM --profile cicd
```


# TODOs

- [ ] Improve doc around what needs to be edited (codepipeline.yaml vs. folder names, trust relation ship, disable change detection)
- [ ] Test entire procedure
- [ ] Automatically disable change detection in codepipeline
- [ ] Fix trust relationship https://github.com/aws/aws-sam-cli/issues/3857
- [ ] Add common pipeline example

