---
draft: true
date: 2024-04-25
slug: static-website-aws
categories:
  - iac
  - cloud
tags: 
  - Terraform
  - AWS
hide:
  - tags
---

# Hosting a statically-generated website on AWS using Terraform Cloud

## Introduction

After preparing for the AWS and Terraform associate exams, I decided to build a small project to put some concepts into practice. Additionally, I took the chance to explore Terraform Cloud (TFC).

**Objective:** Host a statically-generated website on AWS

**Design Decisions (DD):**

1. AWS services will be provisioned and managed using Terraform
2. TFC workspaces/modules & GitHub repositories will be provisioned and managed using Terraform
3. Terraform HCL code used by TFC will be stored in GitHub repositories for version controlled goodness. TFC will manage and store the Terraform state files.
4. Website files will be stored in a GitHub repository and GitHub Actions will be used to build and sync the website files to AWS. In this instance, I'm using the statically-generated website framework [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) to build the website from markdown files.
5. Website will be deployed to a test environment before rolling out to the production environment
6. Production website will be served using CloudFront. Test website will be served using native S3 static website hosting
7. Monthly run costs should be kept as loss as possible
<!-- more -->

**Prerequisites:**

- AWS, GitHub, and TFC accounts
- A domain and hosted zone registered in AWS route53 matching the domain of the website to be provisioned. These will be referenced in the `terraform-aws-s3-website` module
- AWS user with permissions to provision the required services

## Build

There are 4 components to the build, each backed by a corresponding GitHub repository. The Terraform code has been kept generic so that it can be cloned and used by anyone wanting to set up a similar website.

---

### 1. Bootstrap Environment

> Repository: [liftconfig-tfc-boostrap](https://github.com/liftconfig/liftconfig-tfc-bootstrap)

To meet DDs 2 & 3, a Terraform bootstrap environment consisting of a TFC workspace backed by a GitHub repository is used.

The TFC workspace provisions the following resources:

1. TFC root workspace and GitHub repository used for provisioning the AWS services required to run the static website
2. TFC private registry module and GitHub repository for the `terraform-aws-s3-website` module. This module is used by the root workspace and contains Terraform resources for the production & test environments.
3. GitHub repository containing the raw website files in Markdown format. GitHub Actions used in this repository to build and deploy the website to production & test S3 buckets.

#### Container Diagram

![website provisioning system](../../assets/images/blog/static-website-aws/c4-container-website_prov_system.drawio.svg)

#### Manual configuration

The following manual configuration is required to set up the bootstrap environment

##### GitHub

- Create bootstrap repository (liftconfig-tfc-bootstrap) & configure:
    - Settings > branches > main > require a pull request before merging & require approvals before merging
    - Settings > general > disable all features & automatically delete head branches
    - Settings > actions > general > disable actions

##### Terraform Cloud (TFC)

- Create organisation
- Create project (liftconfig-website)
- Create workspace (liftconfig-tfc-bootstrap) & configure:
    - Version Control Workflow > Install Terraform Cloud App on GitHub. Only provide App access to bootstrap repository (add root & module repositories later after they're created) and select this as the repository to host the workspace's Terraform source code.
    - Set Terraform Working Directory to /terraform
    - Set workspace variables (see README in repository for details)

---

### 2. Terraform root workspace

> Repository: [liftconfig-tfc-root](https://github.com/liftconfig/liftconfig-tfc-root)

This Terraform root workspace is provisioned by the bootstrap workspace. It serves two purposes:

1. Provisions the required AWS services to host the production & test websites using the `terraform-aws-s3-website` module
2. Provisions the AWS IAM user, policy, and role to enable GitHub Actions in the website repository to sync the website files to the production & test S3 buckets

#### Component Diagram

![AWS services component diagram](../../assets/images/blog/static-website-aws/c4-component-aws_services.drawio.svg)

#### Manual configuration

The workspace variables must be set in TFC (see README in repository for details)

---

### 3. Terraform module

> Repository: [terraform-aws-s3-website](https://github.com/liftconfig/terraform-aws-s3-website)

This is a private terraform module stored in the TFC registry. The module uses Git tags to control TFC versioning. As with the root workspace, both the module and GitHub repository is provisioned by the bootstrap workspace.

#### Module Purpose

Provisions the required AWS resources to host and run the statically-generated website. Resources are included for a production and a test version of the website. CloudFront is used to serve the production website publically via HTTPS, while native S3 static website hosting is used to serve a test website privately via HTTP. The test website uses a bucket policy to restrict access to a list of specified external IPs.

The following list summarises the AWS resources contained in the module:

#### Production website resources

- ACM certificate for the CloudFront distribution
- CloudFront distribution with OAC to provide HTTPS access to the website
- CloudFront javascript function to redirect www requests to the bare website URL and append index.html to the URI ([required when using some static website generators](https://github.com/aws-samples/amazon-cloudfront-functions/tree/main/url-rewrite-single-page-apps))
- Route 53 CNAME records for ACM certificate DNS validation
- Route 53 A/AAAA records for the www & bare website URLs
- S3 buckets for the website files and CloudFront logs

#### Test website resources

- S3 bucket with static website hosting enabled for the bare test URL
- S3 bucket with static website hosting enabled to redirect www requests to the first bucket
- Route 53 A/AAAA records for the www & bare test website URLs

---

### 4. Website repository

> Repository: [liftconfig-website](https://github.com/liftconfig/liftconfig-website)

Contains the markdown files and [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) configuration to build the test (http://test.liftconfig.com) and production (https://liftconfig.com) websites. The website is deployed using the following GitHub Actions workflow:

1. On a pull request to main (by an approved collaborator); build website and sync files to the test S3 bucket. This bucket uses the native S3 static website hosting and is only accessible by whitelisted IPs.
2. When the pull request is approved and merged; build website and sync files to the production website S3 bucket. The cache of the CloudFront distribution that serves the production website is invalidated forcing CloudFront to fetch the updated files from S3 upon request.

#### Deployment Diagram

![Deployment diagram](../../assets/images/blog/static-website-aws/c4-deployment-website_cicd.drawio.svg)

#### Manual configuration

The secret variables required for the GitHub Action must be set within the repository (see README in repository for details)

> As an extra security measure for public facing repositories it is recommended to change the GitHub setting "actions > general > fork pull request workflows from outside collaborators" to "require approval for all outside collaborators". This safeguards against Actions being run by a PR raised by an unauthorised collaborator. Unfortunately this setting is not exposed by the GitHub API, so it can't be automatically configured using Terraform from the bootstrap workspace.

## Costs

### AWS costs

| AWS Service  | Cost Description                                | Cost per month                  |
|:-------------|:------------------------------------------------|:--------------------------------|
| ACM          | Certificate is free                             | $0                              |
| CloudFront   | HTTPS request                                   | $0.0075 per 1000 HTTP method    |
|              | Sending Data to visitors (US)                   | $0.085 per GB                   |
|              | CloudFront Function                             | $0.1 per million runs           |
| Route 53     | Domain registration (liftconfig.com)            | $1 (\$12 renewed annually)      |
|              | Hosted zone                                     | $0.50 per zone                  |
|              | Queries                                         | $0.40 per million queries       |
| S3 buckets   | Storage                                         | $0.023 per GB                   |
|              | Loading website GET requests                    | $0.0004 per 1000 GET request    |

[Walid Karray](https://medium.com/@walid.karray/mastering-static-website-hosting-on-aws-with-terraform-a-step-by-step-tutorial-5401ccd2f4fb) has estimated the monthly cost for a similar environment. Essentially, unless you're getting a ton of traffic or are serving large files the monthly AWS bill should be in the dollars (<$5).

### Terraform Cloud costs

[Free](https://www.hashicorp.com/products/terraform/pricing) for up to 500 resources per month, then 0.00014 per hour per resource. As the environment was built using a new TFC cloud account the resources for this project fall into the free tier.

### Cost savings

1. The test environment uses native S3 static website hosting eliminating the need for a test CloudFront distribution and, more importantly, a WAF to restrict this distribution to whitelist IPs. A WAF would [cost more](https://aws.amazon.com/waf/pricing/) than the rest of the AWS services combined. The drawbacks to using the native S3 website hosting for testing are it only supports HTTP and you can't test changes to the CloudFront distribution configuration. These are acceptable trade-offs given the non-critical nature of this project, but obviously in a business scenario you would want the test environment to match as close as possible to production.
2. By default, CloudFront is only served from [Price Class 100](https://aws.amazon.com/cloudfront/pricing/) edge locations which can be changed using the `cloudfront_price_class` variable in the root workspace. This means visitors from regions outside of NA, EU, and Israel will experience higher latency as they are served from these cheaper edge locations. Again, an acceptable trade-off given the nature of the project.

## Monitoring

To keep costs down, [StatusCake](https://statuscake.com) is used for free uptime & SSL tests. Basic, but it does the job.

## Benefits

There are a few benefits to using the outlined approach for hosting a statically-generated website on AWS:

- All infrastructure is version controlled and configured using IaC. The required infrastructure for hosting a website can be spun up or down in minutes.
- Creating a module for `terraform-aws-s3-website` allows the code to easily be reused for provisioning other websites in the future
- All website files are written in markdown meaning they are version controlled and can easily be ported to a different statically-generated website framework. In this instance I used [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) but examples of other popular static website generators that could be used are Hugo and Gatsby.
- Having a local development environment (dev container), a test environment (S3 website hosting) and a production environment (CloudFront) provides the confidence that any updates to the website will work as expected.
- Low cost

## Resources

- [Alex Hyett - Deploy a statically-generated website to S3 using GitHub actions](https://www.alexhyett.com/github-actions-deploy-to-s3/)
- [AWS docs - Configuring CloudFront with OAC for S3 endpoints](https://repost.aws/knowledge-center/cloudfront-serve-static-website)
- [AWS docs - Configuring S3 website hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/website-hosting-custom-domain-walkthrough.html)
- [AWS docs - CloudFront Function URL rewrite to append index.html to URL](https://github.com/aws-samples/amazon-cloudfront-functions/tree/main/url-rewrite-single-page-apps)
- [C4 model diagramming](https://c4model.com/)
- [Stack Overflow - Securing GitHub Actions during pull requests](https://stackoverflow.com/questions/64553739/how-to-prevent-github-actions-workflow-being-triggered-by-a-forked-repository-ev)
- [Walid Karray - Step-by-step tutorial for hosting static websites on AWS](https://medium.com/@walid.karray/mastering-static-website-hosting-on-aws-with-terraform-a-step-by-step-tutorial-5401ccd2f4fb)

## Appendix

### Accessing TFC state from Terraform CLI

To access the state stored in TFC from Terraform CLI (useful for troubleshooting, imports, state modification, console etc.):

1. Clone GitHub repository containing TFC workspace HCL code locally
2. Add cloud block with the name of the TFC organisation & workspace:

    ```terraform
    # --- root/providers.tf ---

    terraform {
      ...other provider arguments 

      cloud {
        organization = "org-name-here"

        workspaces {
          name = "workspace-name-here"
        }
      }
    }
    ```

3. Run `terraform login` followed by `terraform init` (API token can be created in TFC under "Account Settings > Tokens > Create an API Token")