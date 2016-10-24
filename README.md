
# Git-backed static website powered entirely by AWS

![diagram](https://raw.githubusercontent.com/alestic/aws-git-backed-static-website/master/aws-git-backed-static-website-architecture.gif "Architecture dagram: Git-backed static website powerd by AWS")

[![Launch CloudFormation stack][2]][1]
[1]: https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?templateURL=https:%2F%2Fs3.amazonaws.com%2Frun.alestic.com%2Fcloudformation%2Faws-git-backed-static-website-cloudformation.yml&stackName=aws-git-backed-static-website
[2]: https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png 
￼
## Overview

This project contains a YAML CloudFormation template that creates a
Git repository and a static https website, along with the necessary
AWS infrastructure glue so that every change to content in the Git
repository is automatically deployed to the static web site.

The input is a domain name and an email address.

The output is a list of nameservers to set in your domain's registrar,
and a Git repository URL for adding and updating the website content.

Git repository event notifications are sent to an SNS topic and your
provided email address is initially subscribed.

Access logs for the website are stored in an S3 bucket.

Benefits of this architecture include:

 - Trivial to launch - Can use aws-cli or AWS console (click "launch"
   above)

 - Maintenance-free - Amazon is responsible for managing all the
   services used.

 - Negligible cost at substantial traffic volume - Starts off as low
   as $0.51 per month if running alone in a new AWS account. (Route 53
   has no free tier.) Your cost may vary over time and if other
   resources are running in your AWS account.

 - Scales forever - No action is needed to support more web site
   traffic, though the costs for network traffic and DNS lookups will
   start to add up to more than a penny per month.

The stack uses a number of AWS services including:

 - **CloudFormation** - Architecture management.

 - **CodeCommit** - Git repository.

 - **CodePipeline** - Passes Git repository content to AWS Lambda upon
   modified.

 - **AWS Lambda** - Syncs Git repository content to S3 bucket for website

 - **S3 buckets** - Website content, www redirect, access logs,
   CodePipeline artifacts

 - **CloudFront** - CDN, HTTPS management

 - **Certificate** Manager - Creation of free certificate for HTTPS

 - **CloudWatch** - AWS Lambda log output, metrics

 - **SNS** - Git repository activity notification

 - **Route 53** - DNS for website

 - **IAM** - Manage resource security and permissions

## Create CloudFormation stack for static website

    domain=example.com
    email=yourrealemail@anotherdomain.com

    template=aws-git-backed-static-website-cloudformation.yml
    stackname=${domain/./-}-$(date +%Y%m%d-%H%M%S)
    region=us-east-1

    aws cloudformation create-stack \
      --region "$region" \
      --stack-name "$stackname" \
      --capabilities CAPABILITY_IAM \
      --capabilities CAPABILITY_NAMED_IAM \
      --template-body "file://$template" \
      --parameters \
        "ParameterKey=DomainName,ParameterValue=$domain" \
        "ParameterKey=NotificationEmail,ParameterValue=$email" \
      --tags "Key=Name,Value=$stackname"
    echo region=$region stackname=$stackname

Three email messages will be sent to the address associated with your
AWS account. Open each and approve these:

 - ACM Certificate (2)
 - SNS subscription

The CloudFormation stack will be stuck until the ACM certificates are
approved. The CloudFront distributions are created afterwards and can
take over 30 minutes to complete.

### Get the name servers for updating in the registrar

    hosted_zone_id=$(aws cloudformation describe-stacks \
      --region "$region" \
      --stack-name "$stackname" \
      --output text \
      --query 'Stacks[*].Outputs[?OutputKey==`HostedZoneId`].[OutputValue]')
    echo hosted_zone_id=$hosted_zone_id

    aws route53 get-hosted-zone \
      --id "$hosted_zone_id"    \
      --output text             \
      --query 'DelegationSet.NameServers'

Set nameservers in your domain registrar to the above.

### Get the Git clone URL

    git_clone_url_http=$(aws cloudformation describe-stacks \
      --region "$region" \
      --stack-name "$stackname" \
      --output text \
      --query 'Stacks[*].Outputs[?OutputKey==`GitCloneUrlHttp`].[OutputValue]')
    echo git_clone_url_http=$git_clone_url_http

### Use Git

    repository=$domain
    profile=$AWS_DEFAULT_PROFILE   # The correct aws-cli profile name

    git clone \
      --config 'credential.helper=!aws codecommit --profile '$profile' --region '$region' credential-helper $@' \
      --config 'credential.UseHttpPath=true' \
      $git_clone_url_http

    cd $repository

## Clean up after testing

### Delete a stack (mostly)

    aws cloudformation delete-stack \
      --region "$region" \
      --stack-name "$stackname"

Leaves behind buckets and Git repo.

### Clean up Git repo and buckets

    # WARNING! DESTROYS CONTENT AND LOGS!

    domain=...

    aws s3 rm --recursive s3://logs.$domain
    aws s3 rb s3://logs.$domain
    aws s3 rm --recursive s3://$domain
    aws s3 rb s3://$domain
    aws s3 rb s3://www.$domain
    aws s3 rm --recursive s3://codepipeline.$domain
    aws s3 rb s3://codepipeline.$domain

That last command will fail, so go use the web console to delete the
versioned codepipeline bucket.

### Delete CodeCommit Git repository

    # WARNING! DESTROYS CONTENT!

    #aws codecommit delete-repository \
      --region "$region" \
      --repository-name "$repository"
