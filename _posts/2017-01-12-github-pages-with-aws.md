---
layout: post
title: "Github pages in AWS"
tags: [aws]
---

This blog post will highlight my migration from having Github host my blog via
Github pages to having it hosted in AWS with S3 and fronted by CloudFront and
Route53. This will not go into exhaustive details with regard to everything
that I went through as far as getting my domains transferred to AWS or creating
an IAM user.

### Assumptions

1. You have an [AWS account] and a basic working knowledge of AWS
2. AWS is your Registrar and/or DNS provider
3. You have integrated [Travis CI] with your Github account
4. You are willing to spend about $1/month in AWS charges to host your blog

### Getting started

I have had my blog on Github pages since December of 2013 and used the CNAME
file feature with Github to have my own domain http://williamsbdev.com. This
worked great and I was very happy for a long time. Eventually, I had this
desire to learn more about how AWS CloudFront/S3/Route53 could host my blog and
had a desire to use SSL. The first thing I did was to setup a throwaway S3
bucket (ie williamsbdev-temp.com). The throwaway bucket is important because S3
buckets in AWS are unique. Once a bucket is created, deleting it can take some
time to propagate and we would like to create our S3 bucket via CloudFormation
with the rest of the needed resources. Next create an IAM user that had access
to only the S3 bucket. The example policy (either managed or inline) below
shows what the users permissions would look like.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Only allow access to williamsbdev-temp.com",
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::williamsbdev-temp.com/*"
            ]
        }
    ]
}
```

For that user I created an access key and secret key from the "Security
Credentials" tab to be used with Travis CI to be able to publish the static
site to S3.

### Setting up Travis CI

The next thing I did was to configure the repo to build on Travis CI. I added a
.travis.yml file (contents seen below) and added the repo via the Travis CI
console. Additionally, I configured the project to have two secret environment
variables `S3_ACCESS_KEY_ID` and `S3_SECRET_KEY` (the access key and secret key
we just created above for the AWS user that will be doing our deployments).

```
# .travis.yml
language: ruby
rvm:
  - 2.1.5
sudo: false
install: gem install jekyll -v 3.1.2
script: jekyll build
deploy:
  provider: s3
  access_key_id: $S3_ACCESS_KEY_ID
  secret_access_key: $S3_SECRET_KEY
  bucket: williamsbdev-temp.com
  local_dir: _site
  region: us-west-2
  skip_cleanup: true
  on:
    repo: williamsbdev/williamsbdev.github.io
```

Once we have confirmed that the pipeline properly configured, pushing a change
to Github master triggers a build on Travis CI that is then deployed
successfully to the AWS S3 bucket, we can move on to the next step.

### Creating the AWS Resources

The first thing we will do is to create an SSL certificate via AWS ACM. As of
this post, you can only use AWS ACM certificates created via the `us-east-1`
region. In your AWS console, ensure that you are in the correct region and then
proceed to request a certificate. When I configured mine, I had both
`williamsbdev.com` and `*.williamsbdev.com` on the certificate added via the
"Add another name to this certificate" feature. You will review the certificate
and then request it. An email will be sent to the registered email for that
domain to validate it. Once validated, expanding the certificate from the list
view will show more details, specifically the `ARN` in the last field of the
second column.

This next step will configure a Route53 HostedZone, CloudFront Distribution,
and S3 bucket for your site. You will clone the Github repo I created that
contains the CloudFormation for all these resources.

    git clone https://github.com/williamsbdev/github-pages-cloudformation.git

Once you have done this, you will go in and update the default values of the
parameters. Replacing `yourdomain.tld` for the `DomainName` parameter,
`your-acm-arn` for the `AcmArn` parameter (the value we got once we validated
our certificate two paragraphs ago), and the `us-west-2` for `Region` with your
desired region. The `CloudFrontServiceHostedZoneId` parameter is a predefined
value for CloudFront found at this [link].

Once you have updated these values, you follow the [README] to create the
CloudFormation stack. Replace the `yourdomain-tld` with the desired name for
your stack and then you may also change the name of the file
`yourdomain-tld.json` to whatever you would like. In my case I did
`williamsbdev-com.json`.

    aws cloudformation update-stack --stackname yourdomain-tld --template-body file://yourdomain-tld.json

Now CloudFront Distributions take quite a bit of time to stand up (in my
experience 30-45 minutes) so you should have plenty of time to finish this
post. :) The setup I have is for the S3 bucket to enable website hosting with
`index.html` as the `Index Document`. I've also configured the S3 bucket with a
policy to only allow [CloudFront IPs] access to the bucket. The reason that I
am restricting the bucket via CloudFront IP ranges and not via an Origin Access
Identity is that I have "pretty" URLs. I do not have the `.html` extension on
each route. CloudFront and S3 have a limitation in that when you host a bucket
as a static web site, it is exposed via HTTP like so.

    http://williamsbdev.com.s3-website-us-west-2.amazonaws.com/

I really wanted only my CloudFront Distribution to be the only way to access my
site. This is possible if you would like to use an Origin Access Identity.
However, you can must also reference the S3 objects by the full name with the
extension. So in my S3 bucket, I actually have every route is a folder with an
`index.html` inside of it to take advantage of the no extension feature of S3.
This was achieved by adding a trailing `/` to the permalink in the
`_config.yml` for Jekyll. When Jekyll does the build, it will create a
directory with an index.html for each route. All together, you can notice the
same URL structure as what is provided by Github pages when hosting on Github
and my blog now has SSL.

[AWS account]: https://aws.amazon.com/
[Travis CI]: https://travis-ci.org/
[link]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-route53-aliastarget.html
[README]: https://github.com/williamsbdev/github-pages-cloudformation
[CloudFront IPs]: https://ip-ranges.amazonaws.com/ip-ranges.json
