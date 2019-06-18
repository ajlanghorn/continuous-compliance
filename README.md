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

Next, install JQ since we're about to use it to interrogate a JSON response privided to us by the IAM API. Installation instructions are available at https://stedolan.github.io/jq/download/, but - in a nutshell - you'll be looking at running...

- `sudo apt-get install jq` on Debian or Ubuntu
- `brew install jq` on a Mac, assuming you have [Homebrew](https://brew.sh) installed

Finally, create an environment variable referencing your AWS account number, as follows:

```
export CC_AWS_ACCOUNT=`aws sts get-caller-identity | jq '.Account' | sed 's/"//g'`
```

## Step 1: Create an evaluatable infrastructure

Our evaluatable infrastructure is the infrastructure we'd be evaluating compliance of against a specific framework. In other words, it's where developers and IT staff would build out infrastructure in AWS. This is in contrast to infrastructure which is designed to do the evaluation of that infrastructure.

For the purposes of this demonstration, we're going to create a fairly rudimentary VPC architecture, comprising a VPC and some Flow Logs.

To do this, visit the VPC console at https://console.aws.amazon.com/vpc.

1. Choose Your VPCs on the left menu, then click Create VPC.
2. Use `evaluatable-infrastructure` as the VPC name.
3. Use `10.0.0.0/16` for the VPC IPv4 CIDR block.
4. Choose not to assign an IPv6 block.
5. Set Tenancy to Default.

Click Create, then close.

Next, visit CloudWatch Logs, at https://console.aws.amazon.com/cloudwatch. Click Logs on the left side. You might need to click 'Let's get started' first to get past the splash screen. Once there, click Create Log Group. Use `GRC338-ContinuousCompliance` as the name. Click Create.

Go back to the VPC console, as above.

Right-click your new VPC, and choose Create Flow Log.

1. Set Filter to All.
2. Select CloudWatch Logs as the destination.
3. Choose the relevant log group from the dropdown. If it doesn't appear, click the Refresh button.
4. Below IAM role, click "Set Up Permissions" to create an appropriately-scoped IAM role.
5. In the pop-up that opens, leave the default settings as they are, and click 'Allow'.
6. Head back to the VPC Flow Logs wizard, and click the refresh button next to the IAM role drop-down.
7. Choose your shiny new IAM role. It'll likely be called `flowlogsRole`.
8. Click Create.

## Step 2: Set up Config

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

Visit the SNS console, and subscribe yourself to the topic you just created. To do this:

1. Visit https://console.aws.amazon.com/sns
2. From the left, choose Topics. If it doesn't appear, choose the hamburger icon.
3. Choose your topic - it's probably called `config`
4. Click the orange 'Create Subscription' button.
5. Select Email for the protocol - if you wanted to send elsewhere, this would be where you do it.
6. Enter a recipient email address.
7. Click Create Subscription.

You should receive an email asking you to confirm your subscription - the link to this will be in a JSON blob.

Finally, for this step, we're going to create an IAM role. This role will allow Config to access the S3 bucket and SNS topic we created above, and get configuration details for services that are supported by Config. Supported services and resources are [documented on the AWS website](https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html). To do this, visit the Config service, at https://console.aws.amazon.com/config. We're also going to set Config up to get it working.

1. In the "Resource Types to Record" section, check "All Resources" to record changes to all supported resources in this region.
2. Next, choose to use the existing Amazon S3 bucket you created above - remember: you created it in your account.
3. Now, choose to stream configuration changes to an Amazon SNS topic, again choosing the SNS topic you previously created.
4. Finally, tell Config to create a service-linked role for us.
5. Skip the section about Rules at this time.
6. Review your settings, and click Confirm.

We'll create a rule in the next step.

## Step 2: Determining if VPC Flow Logs are enabled

Requirement 4.3 of the Center for Internet Security's AWS Fundamentals benchmark is to check whether or not VPC Flow Logs are enabled. A Flow Log allows you to capture, automatically, the IP traffic both orginating and destined from and for network interfaces inside a given VPC. These Flow Logs can then be published to Amazon S3 and Amazon CloudWatch Logs.

1. Start by clicking 'Rules' on the left of the Config console.

You'll see a list of managed rules - these are rules made available to customers which are managed by AWS. They take a variety of inputs to provide their intended usefulness. The `approved-amis-by-id` rule, for example, takes a list of AMI IDs. We're going to make use of the `vpc-flow-logs-enabled` managed rule.

1. Search for the `vpc-flow-logs-enabled` rule.

You can edit the rule name and description, but it's best to keep them as descriptive as possible. Since you can change these names, you'll also see the `Managed Rule Name` field is pre-populated and can't be changed. It's default value is `VPC_FLOW_LOGS_ENABLED`.

2. Next to Trigger Type, ensure Periodic is checked, and set a frequency of 1 hour.

VPC Flow Logs can detect three ACCEPT, REJECT and ALL traffic. We need to tell the rule what traffic we should be detecting and logging in the Flow Logs it detects. For this case, set the value of the `trafficType` tag to `ALL`.

Config Rules can be triggered either by configuration changes, such as Config detecting a VPC's properties have been updated, or periodically. Rules triggered periodically can be checked with a minimum of one hour, up to 24 hours.

3. Choose 'Periodic' for the trigger type, and set the frequency to one hour.
4. Skip the Rule Parameters and Remediation Actions sections.

Were we to wish to remediate automatically, we would set the remediation action to a Systems Manager Automation document, either created for us from an existing template or created by us using Systems Manager.

5. Finally, click Save, and then wait a few seconds. Click the Refresh button a few times, and wait until it's stopped evaluating. You'll then be able to see if you have compliant and non-compliant resources.

6. Click Save.

You'll now notice that Config is evaluating your infrastructure against the rule you've just written! Give it a few moments and you should start to see a response arrive.

## Conclusion

In this workshop, we created:

- a VPC
- a CloudWatch Logs log group
- an S3 bucket
- an SNS topic
- a Config Rule
- associated IAM roles and policies

We now have a situation where detection of VPC Flow Logs is automated for us, and we are alerted when VPCs exist without Flow Logs.

To expand this, we could look at using:

- AWS GuardDuty, to perform machine learning on our security configuration
