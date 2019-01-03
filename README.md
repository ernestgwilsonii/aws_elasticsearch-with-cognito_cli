# aws_elasticsearch-with-cognito_cli
### AWS Elasticsearch with AWS Cognito (for Kibana) via AWS CLI

#### Using AWS for ChatOps logging and reporting
#### Components:
- [AWS Elasticsearch](https://aws.amazon.com/elasticsearch-service/) to log data
- [Kibana](https://aws.amazon.com/elasticsearch-service/kibana/) (comes with AWS Elasticsearch) to query the data / reporting
- [AWS Cognito](https://aws.amazon.com/cognito/) to authenicate users into Kibana
- REF: [Amazon Cognito Authentication for Kibana](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-cognito-auth.html)
- [AWS SDK](https://aws.amazon.com/sdk-for-node-js/) for Node.js to send logs
- REF: https://stackoverflow.com/questions/48273935/how-to-access-aws-elasticsearch-from-node-js
- REF: https://hackernoon.com/centralised-logging-for-aws-lambda-b765b7ca9152


## Prerequisites

##### Functional AWS CLI (likely an administrative account with API access)
##### REF: https://aws.amazon.com/cli/
##### Note: Creating AWS credentials - https://serverless.com/framework/docs/providers/aws/guide/credentials/
```
pip install awscli
aws --version
aws configure
aws iam get-user

aws es list-elasticsearch-versions
aws es list-elasticsearch-instance-types --elasticsearch-version 6.3
```


## Overview of steps
1. Configure Cognito for Elasticsearch (Kibana) authentication
- Create Amazon Cognito user pool
- Pick a GLOBALLY UNIQUE domain prefix for your user pool domain
- Create Amazon Cognito identity pool
- Create IAM Role for Cognito Authenticated
- Create IAM Role for Cognito Unauthenticated
- Link the identity pool roles
2. Create IAM Role with AmazonESCognitoAccess policy attached
3. Gather facts / values needed to deploy Elasticsearch
4. Deploy the Elasticsearch Domain / Cluster
5. Create a Kibana user in Cognito
6. Log in to Kibana


## Configure Cognito for Elasticsearch (Kibana) authentication
##### REF: https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-cognito-auth.html
```
# Create Amazon Cognito user pool
#################################
# REF: https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html
aws cognito-idp create-user-pool --pool-name ChatOps_Cognito_User_Pool \
    --policies PasswordPolicy='{MinimumLength=12,RequireUppercase=true,RequireLowercase=true,RequireNumbers=true,RequireSymbols=true}' \
    --auto-verified-attributes email \
    --username-attributes email \
    --sms-verification-message 'Your verification code is {####}' \
    --email-verification-message 'Your verification code is {####}' \
    --email-verification-subject 'Your verification code' \
    --verification-message-template SmsMessage='"Your verification code is {####} ",EmailMessage="Your verification code is {####} ",EmailSubject="Your verification code",EmailMessageByLink="Please click the link below to verify your email address. {##Verify Email##} ",EmailSubjectByLink="Your verification link",DefaultEmailOption="CONFIRM_WITH_LINK"' \
    --sms-authentication-message '"Your authentication code is {####} "' \
    --mfa-configuration "OFF" \
    --device-configuration ChallengeRequiredOnNewDevice=false,DeviceOnlyRememberedOnUserPrompt=true
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/create-user-pool.html

#aws cognito-idp list-user-pools --max-results=60
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/index.html#cli-aws-cognito-idp
#aws cognito-idp describe-user-pool --user-pool-id "us-east-1_YourUserPoolIdHere"
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/describe-user-pool.html


# Pick a GLOBALLY UNIQUE domain prefix for your user pool domain
################################################################
# You can change it later, just some some random name for now like: chatops12345
# AWS Console --> Cognito --> Manage User Pools --> ChatOps_Cognito_User_Pool --> App integration --> Domain name
# Via AWS CLI:
aws cognito-idp list-user-pools --max-results=60
aws cognito-idp create-user-pool-domain --user-pool-id us-east-1_YourPoolHere --domain chatops12345


# Create Amazon Cognito identity pool
#####################################
# REF: https://docs.aws.amazon.com/cognito/latest/developerguide/identity-pools.html
aws cognito-identity create-identity-pool \
    --identity-pool-name Chatops_Cognito_Identity_Pool \
    --no-allow-unauthenticated-identities
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-identity/create-identity-pool.html

#aws cognito-identity list-identity-pools --max-results=60
#aws cognito-identity describe-identity-pool --identity-pool-id "us-east-1:YourIdentityPoolIdHere"
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-identity/index.html#cli-aws-cognito-identity
#aws cognito-identity delete-identity-pool --identity-pool-id us-east-1:YourIdentityPoolIdHere


# Specify IAM Roles for this Cognito identity pool (for both authenticated and unauthenticated)
# AWS Console --> Cognito --> Manage Identity Pools --> Chatops_Cognito_Identity_Pool --> Dashboard --> "Click here to fix it"
# Using the AWS CLI (below):

# Create IAM Role for Cognito Authenticated
###########################################
aws iam create-role --role-name ChatOps_Cognito_Auth_Role --assume-role-policy-document file://ChatOps_Cognito_Identity_Pool_Auth_Assume_Role_Policy.json
# REF: https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html
# Put an inline policy on the role:
aws iam put-role-policy --role-name ChatOps_Cognito_Auth_Role --policy-name ChatOps_Cognito_Auth_Role_Inline_Policy --policy-document file://ChatOps_Cognito_Identity_Pool_Auth_Attach_Role_Policy.json
# REF: https://docs.aws.amazon.com/cli/latest/reference/iam/put-role-policy.html

# Create IAM Role for Cognito Unauthenticated
#############################################
aws iam create-role --role-name ChatOps_Cognito_Unauth_Role --assume-role-policy-document file://ChatOps_Cognito_Identity_Pool_Unauth_Assume_Role_Policy.json
# REF: https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html
# Put an inline policy on the role:
aws iam put-role-policy --role-name ChatOps_Cognito_Unauth_Role --policy-name ChatOps_Cognito_Auth_Role_Inline_Policy --policy-document file://ChatOps_Cognito_Identity_Pool_Unauth_Attach_Role_Policy.json
# REF: https://docs.aws.amazon.com/cli/latest/reference/iam/put-role-policy.html


# Link the identity pool roles
##############################
# Get IdentityPoolId
aws cognito-identity list-identity-pools --max-results=60
# Get the Authenticated ARN
aws iam get-role --role-name ChatOps_Cognito_Auth_Role
# Get the Unauthenticated ARN
aws iam get-role --role-name ChatOps_Cognito_Unauth_Role
# Set Identity Pool Roles
aws cognito-identity set-identity-pool-roles --identity-pool-id "us-east-1:YourIdentityPoolIdHere" --roles authenticated="arn:aws:iam::XXXXXXXXXXXX:role/ChatOps_Cognito_Auth_Role",unauthenticated="arn:aws:iam::XXXXXXXXXXXX:role/ChatOps_Cognito_Unauth_Role"
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-identity/set-identity-pool-roles.html
#aws cognito-identity get-identity-pool-roles --identity-pool-id "us-east-1:YourIdentityPoolIdHere"
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-identity/get-identity-pool-roles.html
```


## Create IAM Role with AmazonESCognitoAccess policy attached
##### Amazon Elasticsearch needs permissions to configure the Amazon Cognito user and identity pools and use them for authentication
```
# REF: https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-cognito-auth.html#es-cognito-auth-role

# Create your own role:
# REF: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html
aws iam create-role --role-name ChatOps_Elastisearch_Cognito_Role --assume-role-policy-document file://ChatOps-Elastisearch-Cognito-Role-Trust-Policy.json
# REF: https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html

# Attach an AWS supplied "AWS Managed Policy" (AmazonESCognitoAccess) to your IAM Role
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonESCognitoAccess --role-name ChatOps_Elastisearch_Cognito_Role
# REF: https://docs.aws.amazon.com/cli/latest/reference/iam/attach-role-policy.html
```


## Gather facts / values needed to deploy Elasticsearch
##### To enable Cognito in Elasticsearch, we need to know/gather the current Cognito parameters
```
# AWS CLI to get the UserPoolId:
aws cognito-idp list-user-pools --max-results=60
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/index.html#cli-aws-cognito-idp
#aws cognito-idp describe-user-pool --user-pool-id "us-east-1_YourUserPoolIdHere"
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/describe-user-pool.html

# AWS CLI to get the IdentityPoolId:
aws cognito-identity list-identity-pools --max-results=60
#aws cognito-identity describe-identity-pool --identity-pool-id "us-east-1:YourIdentityPoolIdHere"
# REF: https://docs.aws.amazon.com/cli/latest/reference/cognito-identity/index.html#cli-aws-cognito-identity

# AWS CLI to get the RoleArn:
aws iam get-role --role-name ChatOps_Elastisearch_Cognito_Role
# REF: https://docs.aws.amazon.com/cli/latest/reference/iam/get-role.html
```


## Deploy the Elasticsearch Domain / Cluster
```
# Working create Elasticsearch example!
aws es create-elasticsearch-domain --domain-name chatops \
        --region us-east-1 \
        --access-policies "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"AWS\":\"arn:aws:sts::XXXXXXXXXXXX:assumed-role/Cognito_Chatops_Cognito_Identity_PoolAuth_Role/CognitoIdentityCredentials\"},\"Action\":\"es:*\",\"Resource\":\"arn:aws:es:us-east-1:XXXXXXXXXXXX:domain/chatops/*\"}]}" \
        --elasticsearch-version "6.3" \
        --elasticsearch-cluster-config InstanceType=t2.small.elasticsearch,InstanceCount=1 \
        --ebs-options EBSEnabled=true,VolumeSize=10 \
        --cognito-options Enabled=true,UserPoolId="us-east-1_YOURUSERPOOLIDHERE",IdentityPoolId="us-east-1:YOURIDENTITYPOOLIDHERE",RoleArn="arn:aws:iam::XXXXXXXXXXXX:role/ChatOps_Elastisearch_Cognito_Role"
# REF: https://docs.aws.amazon.com/cli/latest/reference/es/create-elasticsearch-domain.html

# Delete Elasticsearch
#aws es list-domain-names
#aws es describe-elasticsearch-domain --domain-name chatops
aws es delete-elasticsearch-domain --domain-name chatops
# REF: https://docs.aws.amazon.com/cli/latest/reference/es/delete-elasticsearch-domain.html
```


## Create a Kibana user in Cognito
##### Blah
```
TBD
```


## Log in to Kibana
##### Blah
```
TBD
```
