# Secure MkDocs Example

You are looking at MkDocs, for full documentation visit [mkdocs.org](https://www.mkdocs.org). This is the mkdocs-material customisation which has excellent documentation itself at [squidfunk.github.io/mkdocs-material](https://squidfunk.github.io/mkdocs-material/reference).

The source code is on [GitHub](https://github.com/JackPott/docs-demo)

This example is hosted at [docsdemo.jackpott.me](https://docsdemo.jackpott.me/). 

**Demo Credentials:**

    Username: `alice@example.com`

    Password: `Password1!`


## What is this? 

Its a demo using mkdocs-material, a static site generator that turns markdown into HTML.

- Its hosted on S3 with CloudFront CDN in front to allow TLS. The cert is handled automatically by ACM.

- All the AWS resources are provisioned with Terraform, you can see the examples for that on the [AWS](aws.md) page. 

- The words you are reading are updated by CI, in this case using GitHub Actions. On merge to master the static site is built and uploaded to an S3 bucket. You can see the source for that on the [CI](ci.md) page. 

- To restrict access, a [Lambda@Edge](lambda.md) function checks for a JWT and if its missing redirects to an IDP (Cognito in this example). If its present, the lambda validates its signature, issuer etc and if all correct loads the page. This relies on the IDP taking responsibly for AuthZ, only giving out access tokens to those who are allowed to see the docs site.  

## Running locally

To play with this locally, install mkdocs-materials:

```shell
pip install mkdocs-material

# Run a local dev server with hot reload
mkdocs serve

# Bundle up for deployment 
mkdocs build
```