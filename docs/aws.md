# AWS Resources

A CloudPosse open source module takes care of most of the boiler plate involved with putting S3 behind CloudFront and closing off the default access. The ACM certificate is requested here too.

## Terraform 

``` terraform title="main.tf" linenums="1"
module "base_label" {
  source   = "cloudposse/label/null"
  version = "0.25.0"

  namespace  = var.namespace
  stage      = var.stage
  name       = "docs-demo"
}

module "cdn" {
  source = "cloudposse/cloudfront-s3-cdn/aws"
  version = "0.86.0"

  context                            = module.base_label.context
  aliases                            = var.aliases
  dns_alias_enabled                  = true
  parent_zone_id                     = var.parent_zone_id
  allow_ssl_requests_only            = false
  forward_cookies                    = "all"
  deployment_principal_arns          = var.deployment_principal_arns
  acm_certificate_arn                = module.acm_request_certificate.arn
  website_enabled                    = true
  s3_website_password_enabled        = true

  depends_on = [module.acm_request_certificate]
}


# Create acm and explicitly set it to us-east-1 provider
module "acm_request_certificate" {
  source = "cloudposse/acm-request-certificate/aws"
  providers = {
    aws = aws.ue1
  }

  version = "0.17.0"
  domain_name                       = var.parent_zone_name
  zone_name                         = var.parent_zone_name
  subject_alternative_names         = var.aliases
  process_domain_validation_options = true
  ttl                               = "300"
  tags                              = module.base_label.tags
}
```


We need to use an extra provider alias to do certain setup tasks in `us-east-1`. ACM certs for use with CloudFront have to be requested in that region. 

```terraform title="provider.tf" linenums="1"
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "eu-west-2"
  default_tags {
    tags = {
      "tf:managed" = "true"
      "tf:stack"   = terraform.workspace
    }
  }
}

# For cloudfront, the acm has to be created in us-east-1 or it will not work
provider "aws" {
  region = "us-east-1"
  alias  = "ue1"
}
```


```terraform title="variables.tf" linenums="1"
variable "parent_zone_id" {
  description = "Hosted Zone ID which the DNS records will be placed in"
  type = string
}

variable "aliases" {
  description = "URLs Cloudfront will listen for. Example: [\"docs.example.com\"]"
  type = list(string)
}

variable "deployment_principal_arns" {
  type = map(list(string))
  description = "Example \"arn:aws:iam::111111222222:user/github-ci-user\" = [\"prefix1\"]"
  default = {} 
}

variable "namespace" {
  description = "Used to name and tag"
  type = string
  default = "oz"
}

variable "stage" {
  description = "Used to name and tag"
  type = string
  default = "demo"
}
```

### IAM Permissions 

Most of the required permissions should get added to the bucket directly as a resource policy, but the CI script still needs an identity to assume. Creating that isn't included in the Terraform.

For reference, the bucket policy for deployment looks like this: 

```json linenums="1"
{
    "Sid": "",
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::1111111122222222:user/github-ci-user"
    },
    "Action": [
        "s3:PutObjectAcl",
        "s3:PutObject",
        "s3:ListBucketMultipartUploads",
        "s3:ListBucket",
        "s3:GetObject",
        "s3:GetBucketLocation",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload"
    ],
    "Resource": [
        "arn:aws:s3:::BUCKET_NAME/*",
        "arn:aws:s3:::BUCKET_NAME"
    ]
}
```

It also needs permission to invalidate the CloudFront cache for smooth updating:

```json linenums="1"
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "cloudfront:GetInvalidation",
                "cloudfront:CreateInvalidation"
            ],
            "Resource": "arn:aws:cloudfront::267032675686:distribution/E288GHX"
        }
    ]
}
```