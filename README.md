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

Next, set the `AWS_DEFAULT_REGION` environment variable, as follows:

```
export AWS_DEFAULT_REGION="us-east-1"
```

Finally, create an environment variable referencing your AWS account number, as follows:

```
export CC_AWS_ACCOUNT=`aws sts get-caller-identity | jq '.Account' | sed 's/"//g'`
```

## Step 1: Set up Config

AWS Config requires access to an Amazon S3 bucket in order to store compliance results, an SNS topic to send compliance results to the bucket from Config, and an IAM role to set the appropriate permissions for the IAM user, the S3 bucket and the SNS topic.

To start with, let's create an Amazon S3 bucket:

```
aws s3 mb s3://$CC_AWS_ACCOUNT-config
```

You should now see a new S3 bucket, named 'config', and prefixed with your account number. This is where Config will store its findings as it monitors for configuration changes over time.

Next, we'll create the SNS topic, used to deliver changes from Config to S3 and on to other services. You can do that as follows:

```
aws sns create-topic --name config
```

The CLI should return the topic's ARN in response.

Finally, for this step, we're going to create an IAM role. This role will allow Config to access the S3 bucket and SNS topic we created above, and get configuration details for services that are supported by Config. Supported services and resources are [documented on the AWS website](https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html).

To do this, firstly copy the following in to a new file called `assumerolepolicy.json`:

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

Next, run:

```
aws iam create-role --role-name GRC338-ContinuousCompliance-ConfigRole --assume-role-policy-document file://assumerolepolicy.json
```

You should see details of your role returned to you in response, including the role name and the ARN.

Now, we need to expand the scope of the IAM role to allow us access to S3 and SNS. To do this, use the following policy, which you need to save locally as `ConfigRolePolicy.json`:

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

Be sure to change the bucket name and the SNS topic ARN.

You need to create the policy, and then attach it, as follows:

```
aws iam create-policy --policy-name GRC338-ContinuousCompliance-ConfigRole-Policy --policy-document file://ConfigRolePolicy.json
```

This command will return you details, including the Policy ARN, which you need to substitute in place of `POLICYARN` below:

```
aws iam attach-role-policy --role-name GRC338-ContinuousCompliance-ConfigRole --policy-arn POLICYARN 
```

## Step 2: Determining if VPC Flow Logs are enabled

Requirement 4.3 of the Center for Internet Security's AWS Fundamentals benchmark is to check whether or not VPC Flow Logs are enabled. A Flow Log allows you to capture, automatically, the IP traffic both orginating and destined from and for network interfaces inside a given VPC. These Flow Logs can then be published to Amazon S3 and Amazon CloudWatch Logs.

We can programatically determine whether or not a given VPC has VPC Flow Logs enabled using a custom AWS Config rule. Custom rules make use of Lambda functions to evaluate the state of a given resource.

The following Python snippet will determine whether or not VPC Flow Logs are enabled, by returning one of `NOT_APPLICABLE`, `COMPLIANT` or `NONCOMPLIANT`. We're going to use the following in a Lambda function, but - first - let's take a look at what it does:

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

To get started, we need to create an IAM role through which our Lambda function can operate. Lambda requires, at a minimum, the `AWSLambdaExecute` policy to be attached to its role, but since we also wish to interrogate a VPC (which uses the EC2 API), and allow Config to execute our Lambda function, we need to also ensure we add the `AmazonEC2ReadOnlyAccess` and `AWSConfigRulesExecutionRole` policies.

1. Open the IAM console, at https://console.aws.amazon.com/iam
1. On the left, choose Roles, then click the blue Create Role button at the top of the page
1. Choose 'AWS Service' for the type of trusted entity, and then 'Lambda' for the specific service. Click Next.
1. Using the search box, filter for, and then check, the `AWSLambdaExecute`, `AWSConfigRulesExecutionRole` and `AmazonEC2ReadOnlyAccess` policies.
1. Skip the step where you add tags - if this was your real organisation, though, you'd be advised to add tags here.
1. For RoleName, use `GRC338-ContinuousCompliance`. Check that there are three attached policies. Click Create Role.

Next, we need to create our Lambda function. We're going to do this in the Console again, too.

1. Open the Lambda console, at https://console.aws.amazon.com/lambda
1. Click Create Function, and then choose Author From Scratch.
1. Use `GRC338-ContinuousCompliance-CIS-4-3` for the name of the function, and choose the Python 3.7 runtime.
1. Under Permissions, click `Choose or create an execution role`, and then click `Use an existing role`.
1. In the Existing Role drop-down that appears, choose `GRC338-ContinuousCompliance`.

Lambda sends us to the Function Designer, where we can see our Lambda function, along with what triggers it and what it can interrogate, trigger or describe. Nothing currently triggers our function, so the left side is empty. On the right, however, we can see a number of services, including Config, CloudWatch, CloudWatch Logs, EC2, EC2 Auto Scaling, ELB and S3. These services are added because Lambda has interrogated the execution role and worked out what services that role gives our Lambda function access to.

Scroll down the page to the Function Code section. Delete the existing boilerplate code, and paste the code from above in to the box.

The Lambda function designer does not save changes automatically, so we need to save our function at this point. To do this, scroll to the top and click the orange Save button at the top right of the page.

By default, our Lambda function will only run for a maximum of three seconds. In some cases, however, we need to run a Lambda function for longer than this, especially when we have a dependency on an external API to respond, as is the case here. We are issuing calls to the EC2 API to request information about a VPC. Amend the timeout to 0min 30sec.

Click Save once again.

We are now ready to test our Lambda function returns correctly. To do this, we'll use Lambda's in-built testing functionality.

1. Along the top of the page, click the drop-down next to Test
1. Click 'Configure test events', and then choose 'Create new test event'
1. Leave the event template set to 'Hello World', and give test event the name `GRC338`
1. Leave the included JSON in the box, and click Create
1. Finally, choose the GRC338 test event from the drop-down and click Test

# Step 3: Create the Config rule

Earlier, in step one, we created an IAM role for our Config rule to use, and in step two, we created a Lambda function which used a role to grant permissions, through a policy, to be run by AWS Config Rules. With our IAM permissions created, and our Lambda function in place, we now need to tie Lambda and Config together.

To enable Config, we can either use the AWS Console, or we can execute the creation of a `Configuration Recorder` using the CLI, as such:

```
aws configservice put-configuration-recorder --configuration-recorder name=default,roleARN=CONFIGROLEARN --recording-group allSupported=true,includeGlobalResourceTypes=true
```

Replace `CONFIGROLEARN` with the ARN of the role you created for Config in Step 1 above.

We also need to create a `Delivery Channel`, which Config uses to deliver results of configuration changes to. This is where we make use of the S3 bucket and SNS topic we created in Step 1 above. To do this, create a file called `deliverychannel.json`, with the following contents:

```
{
  "name": "default",
  "s3BucketName": "S3BUCKETNAME",
  "snsTopicARN": "SNSTOPICARN",
  "configSnapshotDeliveryProperties": {
    "deliveryFrequency": "One_Hour"
  }
}
```

Once saved, run:

```
aws configservice put-delivery-channel --delivery-channel file://deliverychannel.json
```

Config takes a few moments to initially discover resources in an account, after which it will start to become useful as a centralised inventory.

Whilst Config does that, let's add a Config rule. We can add these via the CLI, but the Console gives us a hint as to other rules already made available to us that we can just start using.

1. Navigate to https://console.aws.amazon.com/config
1. Confirm that the region is set to "N. Virginia" at the top right of the page
1. Click Rules on the left.

You'll see a list of managed rules - these are rules made available to customers which are managed by AWS. They take a variety of inputs to provide their intended usefulness. The `approved-amis-by-id` rule, for example, takes a list of AMI IDs.

1. Click Add Custom Rule at the top
1. Enter `GRC338-ContinuousCompliance-CIS-4-3` as the custom rule name
1. Enter the ARN of the Lambda function you created in Step 2

Config Rules can be triggered either by configuration changes, such as Config detecting a VPC's properties have been updated, or periodically. Rules triggered periodically can be checked with a minimum of one hour, up to 24 hours.

1. Choose 'Periodic' for the trigger type, and set the frequency to one hour.
1. Skip the Rule Parameters and Remediation Actions sections.

Were we to wish to remediate automatically, we would set the remediation action to a Systems Manager Automation document, either created for us from an existing template or created by us using Systems Manager.
