# gh-actions-user

This repository provides an AWS CloudFormation template to create AWS IAM User for GitHub Actions.

[GitHub Actions](https://github.com/features/actions) allows developers automate software workflows. It is a CI/CD service running on GitHub. A significant amount of community-powered [workflow](https://docs.github.com/en/actions/learn-github-actions/introduction-to-github-actions#workflows) templates and [actions](https://docs.github.com/en/actions/creating-actions/about-actions) are available. So that configuring a pipeline for most of the projects could be done in a few clicks.

It's not rare when CI/CD service should have access to AWS resources - upload assets to S3, invoke lambda, send notifications, etc. Provided stack creates an AWS IAM User with the least privileges to allow GitHub Actions to communicate with the AWS resources. Feel free to clone repository and tune permissions scope for your needs.

## Scope of the stack
![GitHub User stack](/docs/gh-actions-user.png?raw=true "GitHub User stack")

## Stack resources
The CloudFormation stack produces the following resources:
* IAM Managed Policy - grants access to AWS Resources
* IAM Role - includes managed policy to obtain resource permissions
  * Trust policy - restricts principals that permitted to assume the role*
* IAM User - a container to attach access keys and assume the role
* IAM Policy - an inline policy that permits users to assume the role
* IAM AccessKey - creates access keys required for CLI access

_Note*:_ Trust policy includes ExternalID. It is an additional safety measure. When a user assumes the role, it should provide a unique ExternalID assigned to the user. ExternalID is a way to differentiate role users.

Having the user decoupled from resource access permissions allows better resource access management. In the case of malicious user behaviour, it will be easy to terminate a session and issue new access keys. All done without interruption of the other users that can use the same role. 

## Stack creation and update
There are two ways to create and update stack:
* AWS Console
* AWS CLI

Following is an example of CLI command to create the stack:
```
aws cloudformation create-stack --stack-name gh-actions-user \
  --template-body file://main.yml \
  --parameters ParameterKey=AssetsBucket,ParameterValue=assets-bucket \
  ParameterKey=ExternalId,ParameterValue=ExternalID \
  ParameterKey=ProjectName,ParameterValue=MyProject \
  --tags Key=project,Value=MyProject \
  --region ap-southeast-2 \
  --capabilities CAPABILITY_NAMED_IAM \
  --output yaml \
  --profile my-profile
```

When stack successfully created, it outputs access key and secret access keys. These credentials used to authenticate in AWS.

## Testing permissions
Run the following commands to verify that created user and role have required permissions.

1. Add credentials to `~/.aws/credentials` file (AWS CLI configuration [documentaiton](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)).
```
[gh-actions-user]
aws_access_key_id=ABC123
aws_secret_access_key=abc123
```

2. Use `gh-actions-user` to assume role.
```
aws sts assume-role \
  --role-arn arn:aws:iam::{Your AWS Account}:role/role-name \
  --role-session-name TestSession \
  --external-id ExternalID \
  --profile gh-actions-user
```

_Note:_
* To get role ARN, go to CloudFormation stack resources and open the role page. Role ARN provided on the role page.
* Make sure that the value of `--external-id` flag matches the value of the `ExternalId` parameter of the stack.

The command should return the following response.
```
{
  "Credentials": {
      "AccessKeyId": "ABC123",
      "SecretAccessKey": "abc123",
      "SessionToken": "XYZ==",
      "Expiration": "2021-01-01T23:23:23+00:00"
  },
  "AssumedRoleUser": {
      "AssumedRoleId": "ABC:TestSession",
      "Arn": "arn:aws:sts::{Your AWS Account}:assumed-role/role-name/TestSession"
  }
}
```

3. Use `AccessKeyId`, `SecretAccessKey`, and `SessionToken` to test permissions to AWS Resources.
```
AWS_ACCESS_KEY_ID=ABC123 AWS_SECRET_ACCESS_KEY=abc123 AWS_SESSION_TOKEN=XYZ== aws s3 ls s3://assets-bucket
```

## GitHub settings
GitHub Actions process needs IAM User credentials to authenticate in AWS. To store sensitive information, GitHub provides [secrets](https://docs.github.com/en/actions/reference/encrypted-secrets). Secrets are encrypted environment variables that are available to use in GitHub Actions workflows.

To manage secrets, go to repository settings and choose secrets option:
![GitHub Secrets settings](/docs/gh-secrets-settings.png?raw=true "GitHub Secrets settings")

Store `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ROLE_TO_ASSUME`, and `AWS_ROLE_EXTERNAL_ID` in GitHub Secrets. Now these values are available to use in GitHib Actions workflow.
```yml
name: Upload assets to S3 bucket

on:
  push:
    branches:
    - master

jobs:
  upload:
    name: Upload assets to S3 bucket

    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v2
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-southeast-2
        role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
        role-external-id: ${{ secrets.AWS_ROLE_EXTERNAL_ID }}
        role-duration-seconds: 1200
        role-session-name: AssetsUploadSession
    - name: Copy files to S3 bucket
      run: |
        aws s3 sync . s3://my-s3-assets-bucket
```

The workflow above uses [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) action to get access to AWS resources.

### GitHub Actions flow
![GitHub Actions flow](/docs/gh-actions-flow.png?raw=true "GitHub Actions flow")
