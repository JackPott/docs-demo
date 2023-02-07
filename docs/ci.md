# CI

This example uses GitHub Actions but the basic commands are very simple: 

1. Build the static assets
1. Upload the files to the S3 hosting bucket
1. Force CloudFront to invalidate the cache

It requires a handful of configuration parameters to work:

- AWS credentials, the ARN of which is referenced in the bucket resource policy. It also needs some CloudFront permissions.
- `S3_BUCKET` - The name of the bucket to upload to
- `CF_ID` - The CloudFront distribution ID (output from Terraform)

```yaml title=".github/workflows/ci.yml" linenums="1"
name: deploy
on:
  push:
    branches:
      - master 
      - main
permissions:
  contents: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: 3.x
      - uses: actions/cache@v2
        with:
          key: ${{ github.ref }}
          path: .cache
      - name: Set AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-2
      - run: pip install mkdocs-material 
      - run: mkdocs build
      - run: aws s3 sync site/ s3://${{ vars.S3_BUCKET }} --delete --cache-control max-age=31536000,public
      - run: aws cloudfront create-invalidation --distribution-id ${{ vars.CF_ID }} --paths "/*"
```