# Deploying GitLab (Omnibus version) on Amazon Elastic Container Service (ECS)

## Prerequisites

### Development Environment 

* [jq](https://stedolan.github.io/jq/) - a lightweight and flexible command-line JSON processor
  * brew install jq
* [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) - an open source tool that enables you to interact with AWS services using commands in your command-line shell

### AWS Account

```bash
aws configure --profile $AWS_CLI_PROFILE
```

#### Create Resources

Create these AWS resources in the AWS region you will be deploying GitLab.

* VPC
* Subnets
* S3 bucket

## Installation

### Define environmental variables

```bash
export AWS_REGION=<xxxxxxx>
export AWS_CLI_PROFILE=<xxxxxxx>
export VPC_ID=<vpc-xxxx>
export PUBLIC_SUBNET1=<subnet-xxxxx>
export PUBLIC_SUBNET2=<subnet-xxxxx>
export ECS_EXEC_GITLAB_BUCKET_NAME=ecs-exec-gitlab-output-xxxxxxxxxx
export ECS_EXEC_GITLAB_CLUSTER_NAME=ecs-exec-gitlab-cluster-v1
```

### Example environmental variables

```bash
export AWS_REGION=ap-southeast-2
export AWS_CLI_PROFILE=gitlab
export VPC_ID=<vpc-xxxx>
export PUBLIC_SUBNET1=<subnet-xxxxx>
export PUBLIC_SUBNET2=<subnet-xxxxx>
export ECS_EXEC_GITLAB_BUCKET_NAME=ecs-exec-gitlab-output-919079504386
export ECS_EXEC_GITLAB_CLUSTER_NAME=ecs-exec-gitlab-cluster-v1
```

### Create Amazon S3 bucket

```bash
aws s3api create-bucket \
    --bucket $ECS_EXEC_GITLAB_BUCKET_NAME \
    --create-bucket-configuration LocationConstraint=$AWS_REGION \
    --region $AWS_REGION \
    --profile $AWS_CLI_PROFILE
```

### Create AWS KMS key

AWS CLI command reference [create-key](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/create-cluster.html)

Create a new KMS key:

```bash
KMS_KEY=$(aws kms create-key --region $AWS_REGION --profile $AWS_CLI_PROFILE)
KMS_KEY_ARN=$(echo $KMS_KEY | jq --raw-output .KeyMetadata.Arn)
aws kms create-alias --alias-name alias/ecs-exec-gitlab-kms-key --target-key-id $KMS_KEY_ARN --region $AWS_REGION --profile $AWS_CLI_PROFILE
echo "The KMS Key ARN is: "$KMS_KEY_ARN 
```

If AWS KMS key already exists:

```bash
KMS_KEY_ARN=$(aws kms describe-key --key-id alias/ecs-exec-gitlab-kms-key --region $AWS_REGION --profile $AWS_CLI_PROFILE | jq --raw-output .KeyMetadata.Arn)
echo "The KMS Key ARN is: "$KMS_KEY_ARN
```


### Create Amazon ECS cluster

AWS CLI command reference [create-cluster](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/emr/create-cluster.html)

```bash
aws ecs create-cluster \
    --cluster-name $ECS_EXEC_GITLAB_CLUSTER_NAME \
    --region $AWS_REGION \
    --profile $AWS_CLI_PROFILE \
    --configuration executeCommandConfiguration="{logging=OVERRIDE,\
                                                kmsKeyId=$KMS_KEY_ARN,\
                                                logConfiguration={cloudWatchLogGroupName="/aws/ecs/ecs-exec-gitlab",\
                                                                s3BucketName=$ECS_EXEC_GITLAB_BUCKET_NAME,\
                                                                s3KeyPrefix=exec-output}}" \
    --settings name=containerInsights,value=enabled \
    --tags key=project,value=gitlab
```




