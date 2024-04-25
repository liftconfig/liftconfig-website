# AWS S3 static website - liftconfig.com

## Purpose

- Markdown files for https://liftconfig.com
- The website is built using the statically-generated website framework [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) and served from AWS
- See [Terraform root workspace](https://github.com/liftconfig/liftconfig-tfc-root/) & [Terraform cloud bootstrap](https://github.com/liftconfig/liftconfig-tfc-bootstrap) IaC respositories for deploying production, test, and bootstrap environments using Terraform.

## Deployment

The website is deployed using the following GitHub Actions workflow:

1. On a pull request to main (by an approved collaborator); build website and sync files to the test S3 bucket. This bucket uses the native S3 website hosting option and is only accessible by whitelisted IPs.
2. When the pull request is approved and merged; build website and sync files to the production website S3 bucket. The cache of the CloudFront distribution that serves the production website is invalidated forcing CloudFront to fetch the updated files from S3 upon request.

<br/>
<p align="center">
<img src="docs/assets/images/blog/static-website-aws/c4-deployment-website_cicd.drawio.svg"/>
  </a>
</p>

## GitHub Actions

The following secrets and variables are required for the build and deploy GitHub Action

| Secret                         | Description                                                                                                              |
|:-------------------------------|:-------------------------------------------------------------------------------------------------------------------------|
| AWS_ACCESS_KEY_ID              | Access key for AWS account with permissions to assume `AWS_ROLE`                                                         |
| AWS_SECRET_ACCESS_KEY          | Secret key for AWS account                                                                                               |
| AWS_ROLE                       | `aws_s3_role_arn` output from [Terraform root workspace](https://github.com/liftconfig/liftconfig-tfc-root/). Role is provisioned with required S3 and CloudFront permissions |
| AWS_REGION                     | `aws_region` output from [Terraform root workspace](https://github.com/liftconfig/liftconfig-tfc-root/)                  |
| AWS_CLOUDFRONT_DISTRIBUTION_ID | `website_cloudfront_id` output from [Terraform root workspace](https://github.com/liftconfig/liftconfig-tfc-root/)       |
| AWS_WEBSITE_BUCKET_NAME        | `website_s3_bucket_name` output from [Terraform root workspace](https://github.com/liftconfig/liftconfig-tfc-root/)      |
| AWS_WEBSITE_TEST_BUCKET_NAME   | `website_test_s3_bucket_name` output from [Terraform root workspace](https://github.com/liftconfig/liftconfig-tfc-root/) |
