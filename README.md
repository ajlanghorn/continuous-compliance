# Continuous Compliance

A large number of customers have compliance requirements - either because they're required to by their own internal processes and teams, or because they're regulated by external bodies and laws. Moving to the cloud, in a compliant way, can be time-consuming, and ensuring compliance over a long period, whilst letting developers build without hindrance, can be complicated and difficult.

It doesn't have to be this way!

Using AWS services such as AWS Config, AWS CloudTrail, Amazon CloudWatch, AWS CloudFormation, Amazon GuardDuty and others, it's possible to create a centralised view of compliance across multiple accounts, deployed in an automatable fashion, and visible as a one-stop compliance dashboard for stakeholders across your organisation.

## Framework

The [Center for Internet Security](https://cisecurity.org) produce benchmarks for a variety of products and services in order to help users configure these services in a secure fashion. One common benchmark used by our customers is the CIS Amazon Web Services Fundamentals, available from [the AWS website](https://d0.awsstatic.com/whitepapers/compliance/AWS_CIS_Foundations_Benchmark.pdf).

We'll use this framework, as it provides prescriptive guidance, for the rest of this workshop.

## Getting started

Before we begin to build, you'll need:

- an AWS account
- IAM credentials exported to your shell, as described [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)

To test you have things set up correctly, you can run `aws sts get-caller-identity`. The response to this command should contain your account number and the name of your IAM user.

Finally, create an environment variable referencing your AWS account number, as follows:

```
export CC_AWS_ACCOUNT=`aws sts get-caller-identity | jq '.Account' | sed 's/"//g'`
```

## Step 1: Set up Config

AWS Config requires access to an Amazon S3 bucket in order to store compliance results, an SNS topic to send compliance results to the bucket from Config, and an IAM role to set the appropriate permissions for the IAM user, the S3 bucket and the SNS topic.

To start with, let's create an Amazon S3 bucket:

```
aws s3 mb $CC_AWS_ACCOUNT-config
```

You should now see a new S3 bucket, named 'config', and prefixed with your account number. This is where Config will store its findings as it monitors for configuration changes over time.

Next, we'll create the SNS topic, used to deliver changes from Config to S3 and on to other services. You can do that as follows:

```
aws sns create-topic --name config
```

The CLI should return the topic's ARN in response.

Finally, for this step, we're going to create an IAM role. This role will allow Config to access the S3 bucket and SNS topic we created above, and get configuration details for services that are supported by Config. Supported services and resources are [documented on the AWS website](https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html).

To do this, firstly copy the following in to a new file:

```
{
 "Version": "2012-10-17",
 "Statement": [
  {
   "Effect": "Allow",
   "Principal": {
   "Service": "config.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
  }
 ]
}
```

You should see details of your role returned to you in response, including the role name and the ARN.

Now, we need to expand the scope of the IAM role to allow us access to S3 and SNS. To do this, use the following policy:

```
{
 "Version": "2012-10-17",
 "Statement": [
   {
    "Effect": "Allow",
    "Action": ["s3:PutObject"],
    "Resource": ["arn:aws:s3:::BUCKETNAME/AWSLogs/*"],
    "Condition":
      {
        "StringLike": {
          "s3:x-amz-acl":"bucket-owner-full-control"
        }
      }
    },
    {
     "Effect": "Allow",
     "Action": ["s3:GetBucketAcl"],
     "Resource": "arn:aws:s3:::BUCKETNAME"
    },
    {
     "Effect": "Allow",
     "Action": "sns:Publish",
     "Resource": "SNSTOPICARN"
    }
 ]
}
```

Be sure to change the bucket name and the SNS topic ARN before you submit the request through the CLI.

## Step 2: Determining if VPC Flow Logs are enabled

Requirement 4.3 of the Center for Internet Security's AWS Fundamentals benchmark is to check whether or not VPC Flow Logs are enabled. A Flow Log allows you to capture, automatically, the IP traffic both orginating and destined from and for network interfaces inside a given VPC. These Flow Logs can then be published to Amazon S3 and Amazon CloudWatch Logs.

We can programatically determine whether or not a given VPC has VPC Flow Logs enabled using a custom AWS Config rule. Custom rules make use of Lambda functions to evaluate the state of a given resource.

The following Python snippet will determine whether or not VPC Flow Logs are enabled, by returning one of `NOT_APPLICABLE`, `COMPLIANT` or `NONCOMPLIANT`.

```
import boto3, json

def evaluate_compliance(config_item, r_id):
    if (config_item['resourceType'] != 'AWS::EC2::VPC'):
        return 'NOT_APPLICABLE'

    elif is_flow_logs_enabled(r_id):
        return 'COMPLIANT'
    else:
        return 'NON_COMPLIANT'

def is_flow_logs_enabled(vpc_id):
    ec2 = boto3.client('ec2')
    response = ec2.describe_flow_logs(
        Filter=[
            {
                'Name': 'resource-id',
                'Values': [
                    vpc_id,
                ]
            },
        ],
    )
    if len(response[u'FlowLogs']) != 0: return True

def lambda_handler(event, context):
    
    # Create AWS SDK clients & initialize custom rule parameters
    config = boto3.client('config')
    invoking_event = json.loads(event['invokingEvent'])
    compliance_value = 'NOT_APPLICABLE'
    resource_id = invoking_event['configurationItem']['resourceId']
                    
    compliance_value = evaluate_compliance(invoking_event['configurationItem'], resource_id)
            
    response = config.put_evaluations(
       Evaluations=[
            {
                'ComplianceResourceType': invoking_event['configurationItem']['resourceType'],
                'ComplianceResourceId': resource_id,
                'ComplianceType': compliance_value,
                'Annotation': 'CIS 4.3 VPC Flow Logs',
                'OrderingTimestamp': invoking_event['notificationCreationTime']
            },
       ],
       ResultToken=event['resultToken'])
```
There are a few different functions in the above Python snippet:

- When the Lambda function is invoked, Lambda enters the `lambda_function` function. We've created the necessary SDKs and set up some initial variables for use later on. We've given these initial variables default responses, too.

- The `evaluate_compliance` function contains the logic to determine whether or not VPC flow logs have been enabled. The result - essentially, `COMPLIANT` or `NONCOMPLIANT` is assigned to the `compliance_value` variable.

- In `is_flow_logs_enabled`, we take the VPC ID as an input, and pass to the EC2 API in order to determine whether flow logs are enabled. We filter by the VPC determined by `resource_id`, and if the resulting character count is greater than 0, the function returns True.

- Finally, the Config API provides us with `put_evaluations`, where we push an evaluation to Config. Config records the resource type, resource ID, `compliance_value`, an explaining annotation and a timestamp by which to order the evaluations.

Lambda functions need a role to
