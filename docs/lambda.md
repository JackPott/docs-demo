# Lambda@Edge

Lambda@Edge is a special cut-down version of lambda that can be run during the lifecycle of a CloudFront CDN request/response. 

AWS Labs have an [example](https://github.com/awslabs/cognito-at-edge) of using it with Cognito to gate access to people who are logged in, and an older [blog post](https://aws.amazon.com/blogs/networking-and-content-delivery/authorizationedge-how-to-use-lambdaedge-and-json-web-tokens-to-enhance-web-application-security/) going into some depth on how it should work.

The CloudPosse TerraForm module supports inserting a lambda@edge function but it doesn't support using external libraries (you have to inline all the code). That makes it largely useless, but you can author a normal Lambda in us-east-1 and specify it as a target for the CloudFront distribution. 

```terraform title="main.tf" linenums="10" hl_lines="15-21"
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
  lambda_function_association = [
    {
      event_type   = "viewer-request"
      include_body = false
      lambda_arn   = aws_lambda_function.default.qualified_arn
    }
  ]

  depends_on = [module.acm_request_certificate]
}
```

One limitation of **lambda@edge** is that you can't inject environment variables, so configuration values like OAuth params have to be baked into the code. This can be managed in the Terraform that deploys the Lambda and its ancillaries. 

Fun gotcha debugging CloudFront and Lambda. The logs don't appear in us-east-1, they appear in the region closest to where you made the request!

```js title="index.js" linenums="1"
const { Authenticator } = require('cognito-at-edge');

const authenticator = new Authenticator({
  region: 'eu-west-2', // user pool region
  userPoolId: 'eu-west-2_ABcdef', // user pool ID
  userPoolAppId: 'abc123abc123', // user pool app client ID
  userPoolDomain: 'docsdemo.auth.eu-west-2.amazoncognito.com', // user pool domain
  // logLevel: 'debug',
});

exports.handler = async (request) => authenticator.handle(request);
```

