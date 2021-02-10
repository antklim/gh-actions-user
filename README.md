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
  * Trust policy - restricts principals that permitted to assume the role
* IAM User - a container to attach access keys and assume the role
* IAM Policy - an inline policy that permits users to assume the role
* IAM AccessKey - creates access keys required for CLI access

Having the user decoupled from resource access permissions allows better resource access management. In the case of malicious user behaviour, it will be easy to terminate a session and issue new access keys. All done without interruption of the other users that can use the same role. 

## Stack creation and update

## GitHub Actions flow
![GitHub Actions flow](/docs/gh-actions-flow.png?raw=true "GitHub Actions flow")

## GitHub settings

## Testing permissions
