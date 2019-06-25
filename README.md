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

## Appendix 1: Confirming AWS CloudTrail trails are enabled

[AWS CloudTrail](https://aws.amazon.com/cloudtrail) tracks API usage of AWS services by specific users. It's a particularly useful service for troubleshooting, but also for understanding what happened after a security incident. By default, AWS CloudTrail trails are enabled, but they can be disabled. Requirement 2.1 of the CIS Benchmark, a Level 1 requirement, is to ensure that CloudTrail is enabled in all regions.

To ensure that they haven't been disabled, we can use a Config rule to check that CloudTrail trails have been enabled. We can then go further and check that they are enabled in all regions. This is important to ensure that security issues are able to be caught in more than the primary region you likely consider yourself to be resident in.

The in-built rule we're using for this is called `cloudtrail-enabled`, so go ahead and set that up by clicking Rules on the left of the Config Rules console, finding the rule, and enabling it. Now, either wait a moment or two for the rule to become compliant. If it doesn't, examine why this might be the case - hint: you'll be looking at using CloudTrail's console for that!

Finally, repeat the same steps for the `multi-region-cloudtrail-enabled` rule. Check to see whether the rule is compliant or not. See if you can fix the issue if it's not compliant. You might have to wait a little while before Config picks up on non-compliance, though...

## Appendix 2: Using Config Aggregators

One very powerful feature of Config is aggregation, which allows you to gather together compliance status from various accounts in one single pane of glass. Your auditors and security teams (if you're not already sitting inside one of these!) will love you if you enable this, and give them access.

To create an aggregator, open the Config console, and click Aggregators on the left, under Aggregated View.

1. We need to give Config access to duplicate events from other accounts in to this account. Allow this to occur.
2. Give your aggregator a name.

At this point, if you're running in an Organization, you could add all accounts in your organization with one click, so long as you have a valid IAM role. Since we're not running in this scenario right now, let's ignore that section, and choose instead to provide account IDs.

3. Add `952997488945` as the account ID, alongside any others you wish to use.
4. Optionally, choose regions to monitor. I'd strongly recommend choosing to monitor all regions.
5. Select to include future AWS regions.

Your aggregator should now be created, and you can click through to it to get access to the aggregation dashboard. Data might take a few moments to arrive, as events are replicated from the source AWS account(s) you chose to this one.

## Appendix 3: Using Amazon GuardDuty

[Amazon GuardDuty](https://aws.amazon.com/guardduty) is a threat detection service which continuously monitors for malicious activity and unauthorised behaviour occurring in your AWS account. GuardDuty uses managed rule sets and threat intelligence from the AWS Security teams and trusted third parties, coupled with machine learning, to scan VPC Flow Logs, DNS logs and other sources for malicious behaviour.

We're going to enable GuardDuty and use the `guardduty-enabled-centralized` managed rule in AWS Config. Optionally, this rule takes one parameter - `CentralMonitoringAccount` - which is the account number of a centralised monitoring account which GuardDuty might be pushing findings to. Since we only have one account in our scenario, we're not going to set this parameter.

GuardDuty can be enabled using the AWS CLI. Let's do that by running:

```
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES
```

The `finding-publishing-frequency` parameter takes one of three values: `FIFTEEN_MINUTES`, `ONE_HOUR` and `SIX_HOURS`. You should receive a Detector ID in response. Copy this value as `<DETECTOR_ID>` in the below command:

```
aws guardduty create-sample-findings --detector-id <DETECTOR_ID>
```

Head back to the Config console in your browser, at https://console.aws.amazon.com/config.

1. Click Rules on the left.
2. Search for the `guardduty-enabled-centralized` rule, and enable it.
3. Skip the setting to set the `CentralMonitoringAccount` value.
4. Save, and wait for the finding to evaluate.

Interested in GuardDuty? Go take a look at the GuardDuty console at https://console.aws.amazon.com/guardduty. You should see some sample findings have been created, which lets you explore GuardDuty a little more.

## Thank you!

Thank you for spending some time with me exploring GuardDuty, Config, Config Rules, Config Aggregators and CloudTrail. Services such as this, and others such as Inspector, are helpful in ensuring compliance, especially in regulated or large organisations. I often hear that compliance is a concern with my own customers, but it doesn't have to be difficult using the services we provide natively.
