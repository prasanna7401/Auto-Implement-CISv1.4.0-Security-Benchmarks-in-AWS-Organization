# Deploying CISv1.4.0 Security Benchmark recommended controls with Auto-remediation in an AWS Multi-account setup

This implementation performs near real-time "Automatic" remediation of NON-COMPLIANT resources in an AWS Organizations (multi-account) setup, by using AWS services like Security Hub, Lambda functions, EventBridge rules, etc. It will help in increasing the organization-level security compliance score to protect their cloud environment from cyber threats.

```Last updated - Jan 2024```

## Table of Contents
<details>
 <summary> Click me </summary>
 
  ### Contents
  - [Proposed Architecture](#2-proposed-architecture)
  - [Required AWS Services & Components](#3-required-aws-services--components)
  - [Environment Setup](#4-environment-setup)
  - [Remediation Actions](#5-remediation-actions)
     - [Unsupported CIS Controls](#5-1-unsupported-controls)
     - [Supported CIS Controls](#52supported-controls)
         - [Controls that require "Manual" remediation](#521-controls-that-require-manual-remediation)
           - [Setup Custom-Action based EventBridge Rule](#setup-eventbridge-rule-based-on-custom-action)
         - [Controls that support "Auto" remediation](#522-controls-that-support-automatic-remediation)
           - [IAM Controls](#a-iam-controls)
           - [Storage Controls](#b-storage-controls) 
           - [Logging Controls](#c-logging-controls) 
           - [Monitoring Controls](#d-monitoring-controls)
           - [Networking Controls](#d-networking-controls)
           - [Setup Auto-triggered EventBridge Rule](#setup-eventbridge-rule-for-automatic-remediation)
  - [Test Results](#6-test-results)
  - [Conclusion](#7-conclusion)
</details>

## 1. INTRODUCTION

<details>
 <summary> Click here for detailed description </summary>

 ### 1.1. Introduction
In the ever-evolving landscape of cloud computing, ensuring cloud infrastructure security and compliance has become paramount for organizations of all sizes. To address this critical need, the Center for Internet Security (CIS) has developed a set of comprehensive security benchmarks that provide organizations with a structured approach to securing their computer systems. 

By deploying the proposed automatic remediation solution for CIS security benchmarks in the AWS cloud, organizations can proactively fortify their infrastructure against potential threats and ensure adherence to industry-standard security configurations. This comprehensive approach will empower organizations to safeguard their sensitive data, maintain regulatory compliance, and foster a secure environment for their cloud operations.

### 1.2. What are CIS & CIS Benchmarks?

The Center for Internet Security (CIS) is a non-profit organization that develops and promotes best practices for securing IT systems and data, including cloud security. The CIS Benchmarks are globally recognized and consensus-driven guidelines that help organizations protect against emerging cybersecurity risks. These benchmarks, developed with input from a global community of security experts, provide practical guidance for implementing and managing cybersecurity defenses.

### 1.3.	What are CIS AWS Foundations Benchmarks?

The CIS AWS Foundations Benchmark is a set of security best practices for Amazon Web Services (AWS) resources. It provides prescriptive instructions for configuring AWS services to ensure security and integrity. The most recent version is v1.4.0, released in 2021. Following this benchmark helps organizations reduce security risks and maintain compliance with industry regulations.

### 1.4.	Importance of CIS Benchmarks

The CIS Benchmarks are globally recognized and accepted best practice guides for securing IT infrastructure. The benchmarks are freely available for download and implementation and provide up-to-date, step-by-step instructions for organizations to secure their infrastructure. 

The CIS Benchmarks align with major security and data privacy frameworks such as: 
* National Institute of Standards and Technology (**NIST**) Cybersecurity Framework
* Health Insurance Portability and Accountability Act (**HIPAA**)
* Payment Card Industry Data Security Standard (**PCI DSS**)

### 1.5.	CISv1.4.0 Recommended Controls

The CISv1.4.0 Control is composed of 4 sections with a total of 58 controls known as “recommendations.”
Below are the four sections:

- Identity and Access Management – 21 Controls
- Storage – 7 Controls
- Logging – 11 Controls
- Monitoring – 15 Controls
- Networking – 4 Controls

</details>

### 1.6.	Problem Statement

In an AWS Organization setup with hundreds of accounts, enforcing organization-level security regulations for each resource deployed in various regions is a tedious task. An organization's security team will need to put a lot of effort into taking necessary actions to increase the compliance score.

<!-- Document authored by Prasanna Venkatesan Aravindan (prasanna7401@gmail.com) on 12th December 2023 -->

## 2. PROPOSED ARCHITECTURE

### 2.1. Security Hub Setup in AWS Organizations

![Security Hub setup in AWS Organization setup with Delegated Administrator](./screenshots/architecture_securityhub_organization_setup.png)

In an AWS Organizations setup, there will be a [Delegated Administrator Account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_delegate_policies.html) for Security Hub. This account will act as a Centralized Security Dashboard for the entire organization.

### 2.2. Remediation Action Flow based on a Security Hub Finding - Simplified

![Simplified Remediation Action Flow architecture](./screenshots/architecture_remediation_flow_simple.png)

### 2.3. Remediation Action Flow based on a Security Hub Finding - Detailed

![Detailed Remediation Action Flow architecture](./screenshots/architecture_remediation_flow_detailed.png)

The above architecture will be explained in detail in the [Remediation Actions](#5-remediation-actions) section

## 3. REQUIRED AWS SERVICES & COMPONENTS

- [Config](https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html) - the primary source that performs security configuration checks and sends them to AWS Security Hub.

- [Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html) - provides a Centralized Security Dashboard that displays security finding status across all organization member accounts in a prioritized manner. Security Hub currently supports automated checks for standards like, 
    - AWS Foundational Security Best Practices (FSBP) v1.0.0
    - CIS Benchmarks v1.2.0
    - CIS Benchmarks v1.4.0
    - NIST 800-53 Revision 5
- [EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) - helps in setting up rule-based triggers that will deliver events to selected targets.

- [Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) - event-driven serverless compute service that allows us to run our code in response to event triggers like EventBridge rules, without having to provision or manage servers.

- [IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) - it is an identity that has specific permissions. Unlike an IAM user, an IAM role does not have long-term credentials.  when you assume a role, it provides temporary security credentials for the role session. Some AWS Services will assume an IAM role to perform necessary actions.

- [Simple Notification Service](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) - a fully managed distributed publish-subscribe system allowing mass delivery of emails, messages, and notifications.

- [CloudFormation StackSet](https://docs.aws.amazon.com/cloudformation/) - an Infrastructure-as-a-Code solution that helps in quick deployment of resources across multiple accounts and regions under a single operational management framework.

## 4. ENVIRONMENT SETUP

> Pre-requisite: An AWS Organization setup with multiple member accounts, and a management account. Also, Organization-level services like AWS Config, Security Hub, CloudFormation StackSet, CloudTrail, etc. must be enabled as per your requirement, and setup Delegated Administrator accounts for managing these services if needed.
>> NOTE: Due to the AWS Organizations setup, whatever control measure is implemented at the organization level will NOT be enforced on the Management Account (SCPs are applicable to the Management account).

### 4.1. Enable AWS Config

> To ensure that the Security Hub can access its findings, it is necessary to activate AWS Config in the desired regions across all member accounts within the organization.

1. In the Organization’s CloudFormation StackSet Delegated Administrator or the Management account, go to <code>CloudFormation > StackSets > Create StackSet</code> & upload the [Enable_AWS_Config.yml](./CloudFormation_Templates/Enable_AWS_Config.yml) template file.
2. Choose the Parameter values as per your requirements. But let the <code>Include global resource types</code> as <code>FALSE</code>, because AWS Config need not perform redundant checks for Global resources like IAM in each region unnecessarily.

    ![Include global resource type setting](./screenshots/cloudformation_include_global_resource_parameter.png)
3. Set the <code>Deployment options</code> as per your requirement. But for our implementation, we need the deployment targets to be the entire organization.

    ![Deployment Targets](./screenshots/cloudformation_deployment_target.png)
4. Set the <code>Auto-deployment</code> options as <code>Activated</code>, so that when a new member is added, AWS Config is enabled as per the setup.

   ![Deployment Targets](./screenshots/cloudformation_auto_deployment.png)
5. Choose the deployment regions as per your requirement & Click Next > Submit to deploy.

Now, AWS Config will be enabled in all the organization member accounts in the regions you have specified.

### 4.2. Enable AWS Security Hub:

1. To enable AWS Security Hub in the organization member accounts, in the Security Hub Delegated Administrator Account’s Security Hub console of your primary region, go to <code>Configuration > Enable Central Configuration</code>.
2. Choose the Regions which you want to be aggregated, so that all region findings will be shown in the Security Hub dashboard of the primary region. Click Next.
3. For the <code>Configuration type</code>, you can either choose to use the AWS Recommended setting to enable all standards or choose <code>Customize my Security Hub configuration</code> and choose only <code>CIS AWS Foundations Benchmark v1.4.0</code>

    ![Enable Security Hub using the Central Configuration option from Delegated Administrator's Security Hub Console](./screenshots/securityhub_central_enable.png)
4. Choose Deploy to all accounts > Next > Submit.

Now, AWS Security Hub will be enabled in the regions that you have mentioned, with control checks for CIS 1.4.0 enabled. For more information, refer to the [AWS Documentation - Security Hub Central Configuration](https://docs.aws.amazon.com/securityhub/latest/userguide/central-configuration-intro.html)

### 4.3. Create SNS Topic

> To be able to get email notification about the remediation steps taken for an automatically remediated CIS control check or get the steps to perform remediation for a control check that is triggered manually, we need to create an SNS topic in the organization member accounts in whichever region we are performing the remediation. 

1. To do this, use the [CIS_Remediation_Notification_Setup.yml](./CloudFormation_Templates/CIS_Remediation_Notification_Setup.yml) file and deploy using CloudFormation StackSets in all organization members in all regions that you want. Also, let the auto-deployment option be in an Activated state.
2. Now, you can have the necessary email accounts to subscribe to this SNS topic to receive notifications.

### 4.4. Setup Remediation Lambda Function

1. In the Security Hub Delegated Administrator account, create the CIS Control Remediation Lambda function with your preferred name _(say CIS_Remediation_Master)_ and choose the Runtime language as <code>Python 3.11</code>, and create the function with default permissions.
2. Now, upload the codes [lambda_function.py](./main/lambda_function.py) and [cisPlaybook.py](./main/cisPlaybook.py) as a zip file.
3. Go to <code>Configuration > General Configuration</code> and set the <code>Timeout</code> as <code>5 sec</code>.
4. To allow this Remediation Lambda function to be able to assume role <code>CIS_Remediator_Role</code> in the member accounts (we will create after we set this function). We need to give it _sts:AssumeRole_ permission policy. 
5. To do this, Create an IAM Policy with your preferred name _(say, CIS_Remediator_MemberRoleAssumption)_ with the below-mentioned permission, so that the lambda function can assume the _CIS_Remediator_Role_ role in the member accounts to perform remediation.

    ```json
    {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "CrossAccountRemediatorRoleAssumption",
                    "Effect": "Allow",
                    "Action": "sts:AssumeRole",
                    "Resource": "arn:aws:iam::*:role/CIS_Remediator_Role"
                }
            ]
    }
    ```
    
6. Attach this permission policy to the remediation lambda function’s IAM role, in addition to the default lambda permissions.

### 4.5. Create an IAM Role in Member Accounts

> To allow our lambda function to be able to perform remediation action in the organization member accounts, it needs to have sufficient permissions. For this, we need an IAM role in the member accounts, which will be assumed by our lambda function.

1. Using the CloudFormation template [CIS_Remediator_Role_Deployment.yml](./CloudFormation_Templates/CIS_Remediator_Role_Deployment.yml), we will create an IAM role named <code>CIS_Remediator_Role</code> with AWS-managed permission AdministratorAccess with ARN <code>arn:aws:iam::aws:policy/AdministratorAccess</code>.
> If you wish not to give Administrator Access to the assumed member account IAM role, you need to create an IAM policy with necessary permissions that allow the lambda function to perform the necessary remediation actions for all of the CIS Controls. In this case, you can use your own CloudFormation template to create an IAM policy in all the member accounts, and change the ARN of the policy in "CIS_Remediator_Role_Deployment.yml"
2. Since IAM is a global resource, choose only one deployment region.
3. Also, set the Auto-deployment option as Activated, so that this IAM role will be created in new member accounts also.
4. During the deployment, the CloudFormation console will prompt you to provide the <code>ARN of the Remediation lambda function’s IAM role</code>, to create a trust relationship policy in the Member account IAM role, so that our lambda function can assume it successfully.

    ![Member Role deployment parameter requesting Remediation Lambda function's IAM Role ARN](./screenshots/cloudformation_member_role_deployment_parameter.png)

### 4.6. Optional Requirements

For the <code>CIS Control ID – 3.7</code>: CloudTrail Logs should have encryption at rest enabled, we need a KMS key with sufficient permissions. To be able to automatically remediate this control, use CloudFormation StackSets to deploy [CIS_CloudTrail_Encryption_KMS_Key_Deployment.yml](./CloudFormation_Templates/CIS_CloudTrail_Encryption_KMS_Key_Deployment.yml) across all member accounts in the desired regions.

> Additional Resources: If you would like to Disable or Force Enable any of the Security Hub Controls in your organization member accounts, you can implement solutions suggested in the below [AWS blog - Disabling Security Hub Controls in a Multi-Account Environment](https://aws.amazon.com/blogs/security/disabling-security-hub-controls-in-a-multi-account-environment/)

Now, all the requirements to implement our solution have been set up.

## 5. REMEDIATION ACTIONS

Out of 58 different remediation controls suggested by CIS, a few are not supported by AWS. For more information, refer to the [AWS documentation - Center for Internet Security (CIS) AWS Foundations Benchmark v1.2.0 and v1.4.0](https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html)

Also, information required to allow you to customize remediation actions by modifying the variable values in [lambda_function.py](./main/lambda_function.py) has been provided wherever required, for each of the CISv1.4.0 control
<!-- This document has been authored by Prasanna Venkatesan Aravindan on 12th December 2023 -->

### 5.1. Unsupported Controls

- CIS Controls that are not supported for automated checks done by Security Hub: <code>CIS 1.1, 1.2, 1.3, 1.11, 1.13, 1.18, 1.19, 1.20, 1.21, 2.1.1, 2.1.3, 2.1.4, 4.15, 5.4</code>
- CIS Controls that were in CIS v1.2.0, but not supported in CIS v1.4.0, and Controls for which automated control check is disabled by AWS Security Hub for CIS v1.4.0: <code>CIS 1.15, 4.1, 4.2, 5.2</code>
> Note: Here, Control ID 5.2 is covered under 5.1 – Network ACL should not allow ingress from 0.0.0.0/0 to remote administration ports.

### 5.2.	Supported Controls

Among the controls supported by AWS for automated checks done by Security Hub, some need manual intervention for remediation (like setting up MFA, Root account setting modifications), while others can be auto-remediated. 

> As many of the Security Standards recommended and supported by Security Hub overlap, AWS has its own set of control IDs for each matching standard recommended control ID.

Below is the summary of the remediation action done for each CIS control:

#### 5.2.1. Controls that require "Manual" remediation

| CIS Control ID | AWS Control ID | Control Description | Generator ID |
|----------|----------|----------|----------|
|   1.4  |   <code>IAM.4</code>  |   IAM root user access key should not exist  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.4</code>  |
|   1.5  |   <code>IAM.9</code>  |   MFA should be enabled for the root user  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.5</code>  |
|   1.6  |   <code>IAM.6</code>  |   Hardware MFA should be enabled for the root user  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.6</code>  |
|   1.10  |   <code>IAM.5</code>  |   MFA should be enabled for all IAM users that have a console password  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.10</code>  |
|   1.16  |   <code>IAM.16</code>  |   Ensure that Customer-managed IAM policies should not allow full "*:*" administrative privileges  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.16</code>  |
|   2.3.1  |   <code>RDS.3</code> |   RDS DB instances should have encryption at-rest enabled  |   <code>cis-aws-foundations-benchmark/v/1.4.0/2.3.1</code>  |

> The Generator ID mentioned above, is retrieved from the Security Hub Findings. This will be useful in setting up custom EventBridge rules depending on your requirements.

#### Setup EventBridge Rule based on Custom Action

For the above controls, the EventBridge Rule is set to be triggered only upon clicking the <code>Custom Action</code> feature in Security Hub.

1. To Create a Custom Action, in the Security Hub Delegated Administrator, go to <code>Security Hub > Custom Action > Create Custom Action</code> _(say, CIS_Remediation)_
2. Now, go to <code>EventBridge > Rules > Create Rule</code> & Choose your desired rule name _(say, CIS_Remediation_Master_CustomAction_Trigger)_ and give a description.
3. Choose the <code>EventBridge source</code> as “AWS Events or EventBridge partner events”, and <code>Creation method</code> as <code>Use pattern form</code>.\

    ![EventBridge Initial Setup](./screenshots/eventbridge_initial_setup.png)
4. Choose the <code>Event pattern</code> as shown below, with Event type as <code>Security Hub Findings – Custom Action</code> & specify the Custom Action ARN you had created.

    ![EventBridge Pattern](./screenshots/eventbridge_pattern_custom_action.png)
5. Click Next & set the Target as Lambda function & choose the name of the Remediation Lambda function.
6. Click Next & Click on <code>Create Rule</code>.


#### How to Trigger this?
Choose a FAILED compliance control check, Click on <code>Action > Name of the Custom Action</code> you had created. This will trigger the Remediation lambda function to send out an email notification with instructions to perform the necessary remediation action, to the emails subscribed to the SNS topic.

_Sample Email Notification mentioning steps to perform remediation_
![Sample Email Notification mentioning steps to perform remediation](./screenshots/email_manual.png)


> Note: If you prefer to use a different SNS topic for each control, you can simply add the <code>sns_topic_arn</code> variable inside the corresponding <code>“if” condition</code> in the [lambda_function.py](./main/lambda_function.py) code and mention your SNS topic’s ARN.


#### 5.2.2. Controls that support "Automatic" remediation

For the below controls, the impact status has been given based on the performed automatic remediation
| Symbol | Description |
|----------|----------|
|  ❗  |  Impactful  |
|  ⚠️  |  Possible impact (depending on your setup)  |
|  ✅  |  Safe  |

##### A) IAM Controls 

| CIS Control ID | AWS Control ID | Control Description | Generator ID | Action Taken | Impact |
|----------|----------|----------|----------|----------|----------|
|   1.7  |   <code>CloudWatch.1</code>  |   Eliminate the use of 'root' user for administrative and daily tasks  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.7</code>  |   Remediated under CIS Control 4.3: Ensure Log metric filter and alarm should exist for usage of the "Root" user  |  ✅  |
|   1.8  |   <code>IAM.15</code>  |   Ensure IAM password policy requires minimum password length of 14 or greater  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.8</code>  |   Changes password policy  |  ✅  |
|   1.9  |   <code>IAM.16</code>  |   Ensure IAM password policy prevents password reuse  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.9</code>  |   Changes password policy  |  ✅  |
|   1.12  |   <code>IAM.22</code> |   IAM user credentials unused for 45 days should be removed  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.12</code>  |   Keys unused for more than 45 days will be deleted  |  ❗  |
|   1.14  |   <code>IAM.3</code>  |   IAM users' access keys should be rotated every 90 days or less  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.14</code>  |   Keys older than 90 days will be disabled  |  ⚠️  |
|   1.17  |   <code>IAM.18</code>  |   Ensure a support role has been created to manage incidents with AWS Support  |   <code>cis-aws-foundations-benchmark/v/1.4.0/1.17</code>  |   Creates an IAM Role with <code>support:*</code> access   |  ✅  |


> Note for Customization:
> 1. For CIS 1.17 remediation, you can change the name of the IAM role created by modifying the <code>support_role_name</code> variable in [lambda_function.py](./main/lambda_function.py)
> 2. For CIS 1.8 & CIS 1.9 remediation, you can further customize the password policy based on your requirement by modifying the <code>password_policy</code> variable in [cisPlaybook.py](./main/cisPlaybook.py)


##### B) Storage Controls 

| CIS Control ID | AWS Control ID | Control Description | Generator ID | Action Taken | Impact |
|----------|----------|----------|----------|----------|----------|
|   2.1.2  |   <code>S3.5</code>  |   S3 buckets should require requests to use Secure Socket Layer, set to deny HTTP requests  |   <code>cis-aws-foundations-benchmark/v/1.4.0/2.1.2</code>  |   Adds a new statement in the S3 bucket policy to deny HTTP requests  |  ⚠️  |
|   2.1.5.1  |   <code>S3.1</code>  |   S3 Block Public Access setting should be enabled at account level  |   <code>cis-aws-foundations-benchmark/v/1.4.0/2.1.5.1</code>  |   Enables <code>"Block all public access"</code> setting at account-level for S3  |  ❗  |
|   2.1.5.2  |   <code>S3.8</code>  |   S3 Block Public Access Block setting should be enabled at the bucket-level  |   <code>cis-aws-foundations-benchmark/v/1.4.0/2.1.5.2</code>  |   Enables <code>"Block all public access"</code> setting at bucket-level for S3  |  ❗  |
|   2.2.1  |   <code>EC2.7</code> |   EBS default encryption should be enabled  |   <code>cis-aws-foundations-benchmark/v/1.4.0/2.2.1</code>  |   Enables <code>"Always encrypt new EBS volumes"</code> in EC2 Console settings  |  ✅  |

> Note for Customization:
> 1. For CIS 2.2.1 remediation, the Remediation Lambda function’s execution **timeout needs to be 5 seconds**


##### C) Logging Controls 

| CIS Control ID | AWS Control ID | Control Description | Generator ID | Action Taken | Impact |
|----------|----------|----------|----------|----------|----------|
|   3.1  |   <code>CloudTrail.1</code>  |   Ensure that CloudTrail is enabled in all regions & set to log read/write events in CloudTrail S3 bucket  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.1</code>  |   Enabled CloudTrail in compliance failed region with CloudTrail S3 bucket logging set to monitor read/write events  |  ✅  |
|   3.2  |   <code>CloudTrail.4</code>  |   CloudTrail log file validation should be enabled  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.2</code>  |   Enabled <code>Log Validation</code> in compliance failed trail  |  ✅  |
|   3.3  |   <code>CloudTrail.6</code>  |   Ensure the S3 bucket used to store CloudTrail logs is not publicly accessible  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.3</code>  |   Enables <code>Block all public access</code> setting at CloudTrail Bucket  |  ✅  |
|   3.4  |   <code>CloudTrail.5</code> |   CloudTrail trails should be integrated with Amazon CloudWatch Logs  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.4</code>  |   Creates CloudWatch log & IAM role (if not exists) with CloudWatch log writing permissions & integrates CloudTrail with CloudWatch Log group |  ✅  |
|   3.5  |   <code>Config.1</code> |   AWS Config must be enabled in all regions to monitor all resources  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.5</code>  |  No remediation code has been provided for this Control ID. Because, while enabling AWS config at the organization level, we have set up <code>Include Global Resources</code> as <code>FALSE</code> to avoid redundant checks for global resources like IAM. Since AWS Config checks is not allowed for all resources, this control check will be in a FAILED state. You can choose to disable this control check if you wish.  |  -  |
|   3.6  |   <code>CloudTrail.7</code> |   Ensure S3 bucket access logging is enabled on the CloudTrail S3 bucket  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.6</code>  |   Enables <code>Server Access Logging</code> in CloudTrail S3 bucket’s properties  |  ✅  |
|   3.7  |   <code>CloudTrail.2</code> |   CloudTrail Logs should have encryption at-rest enabled  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.7</code>  |   Enabled <code>Log file SSE-KMS encryption</code> using the KMS key created using CloudFormation template [CIS_CloudTrail_Encryption_KMS_Key_Deployment.yml](./CloudFormation_Templates/CIS_CloudTrail_Encryption_KMS_Key_Deployment.yml) earlier.  |  ✅  |
|   3.8  |   <code>KMS.4</code> |   AWS KMS key rotation should be enabled  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.8</code>  |   Enables <code>Automatically rotate this KMS key every year</code> option  |  ⚠️  |
|   3.9  |   <code>EC2.6</code> |   Ensure VPC Flow (reject) logging is enabled in all VPCs  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.9</code>  |   Enables VPC flow logging by creating a new CloudWatch Log group, and an IAM role with log group write permissions, to log <code>REJECT</code> logs for each VPC.  |  ✅  |
|   3.10  |   <code>CloudTrail.1</code> |   Ensure that object-level logging for Write events is enabled for S3 buckets  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.10</code>  |   Remediated under CIS 3.1  |  ✅  |
|   3.11  |   <code>CloudTrail.1</code> |   Ensure that object-level logging for Read events is enabled for S3 buckets  |   <code>cis-aws-foundations-benchmark/v/1.4.0/3.11</code>  |   Remediated under CIS 3.1  |  ✅  |

> Note for Customization: 
> 1. For CIS 3.4 remediation, you can change the name of the IAM role created by modifying the <code>iam_rolename</code>.
> 2. For CIS 3.7 remediation, If you already have a KMS key with necessary permissions, you can add <code>key_alias</code>.
> 3. For CIS 3.8 remediation, you can give a list of keywords in <code>exclusion_keywords</code> variable, so that KMS keys with descriptions containing these keywords will not be rotated.
> > All the above variable changes need to be done in [lambda_function.py](./main/lambda_function.py)

##### D) Monitoring Controls 

| CIS Control ID | AWS Control ID | Control Description | Generator ID | Action Taken | Impact |
|----------|----------|----------|----------|----------|----------|
|   4.3  |   <code>CloudWatch.1</code>  |   Ensure a log metric filter and alarm exist for Usage of 'Root' account  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.3</code>  |   Creates a Log metric & Alarm to monitor "Root" usage <code> { $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" } </code> |  ✅  |
|   4.4  |   <code>CloudWatch.4</code>  |   Ensure a log metric filter and alarm exist for IAM Policy changes  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.4</code>  |   Creates a Log metric & Alarm to monitor IAM Policy changes using the filter <code> {($.eventName=DeleteGroupPolicy) &#124;&#124; ($.eventName=DeleteRolePolicy) &#124;&#124; ($.eventName=DeleteUserPolicy) &#124;&#124; ($.eventName=PutGroupPolicy) &#124;&#124; ($.eventName=PutRolePolicy) &#124;&#124; ($.eventName=PutUserPolicy) &#124;&#124; ($.eventName=CreatePolicy) &#124;&#124; ($.eventName=DeletePolicy) &#124;&#124; ($.eventName=CreatePolicyVersion) &#124;&#124; ($.eventName=DeletePolicyVersion) &#124;&#124; ($.eventName=AttachRolePolicy) &#124;&#124; ($.eventName=DetachRolePolicy) &#124;&#124; ($.eventName=AttachUserPolicy) &#124;&#124; ($.eventName=DetachUserPolicy) &#124;&#124; ($.eventName=AttachGroupPolicy) &#124;&#124; ($.eventName=DetachGroupPolicy)} </code>  |  ✅  |
|   4.5  |   <code>CloudWatch.5</code>  |   Ensure a log metric filter and alarm exist for CloudTrail Configuration changes  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.5</code>  |   Creates a Log metric & Alarm to monitor CloudTrail Configuration changes using the filter <code> { ($.eventName = CreateTrail) &#124;&#124; ($.eventName = UpdateTrail) &#124;&#124; ($.eventName = DeleteTrail) &#124;&#124; ($.eventName = StartLogging) &#124;&#124; ($.eventName = StopLogging) } </code>  |  ✅  |
|   4.6  |   <code>CloudWatch.6</code> |   Ensure a log metric filter and alarm exist for AWS Management Console Authentication Failures  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.6</code>  |   Creates a Log metric & Alarm to monitor Console Authentication Failures using the filter <code> { ($.eventName = ConsoleLogin) && ($.errorMessage = "Failed authentication") } </code>  |  ✅  |
|   4.7  |   <code>CloudWatch.7</code>  |   Ensure a log metric filter and alarm exist for Disabling or Scheduled Deletion of customer created CMKs  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.7</code>  |   Creates a Log metric & Alarm to monitor deletion/disabling of customer-managed keys using the filter <code> { $.eventSource = kms* && $.errorMessage = "* is pending deletion."} </code>  |  ✅  |
|   4.8  |   <code>CloudWatch.8</code>  |   Ensure a log metric filter and alarm exist for S3 Bucket Policy changes  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.8</code>  |   Creates a Log metric & Alarm to monitor S3 bucket policy changes using the filter <code> { ($.eventSource = s3.amazonaws.com) && (($.eventName = PutBucketAcl) &#124;&#124; ($.eventName = PutBucketPolicy) &#124;&#124; ($.eventName = PutBucketCors) &#124;&#124; ($.eventName = PutBucketLifecycle) &#124;&#124; ($.eventName = PutBucketReplication) &#124;&#124; ($.eventName = DeleteBucketPolicy) &#124;&#124; ($.eventName = DeleteBucketCors) &#124;&#124; ($.eventName = DeleteBucketLifecycle) &#124;&#124; ($.eventName = DeleteBucketReplication)) } </code>   |  ✅  |
|   4.9  |   <code>CloudWatch.9</code>  |   Ensure a log metric filter and alarm exist for ‘AWS Config’ configuration changes  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.9</code>  |   Creates a Log metric & Alarm to monitor ‘AWS Config’ configuration changes using the filter <code> { ($.eventSource = config.amazonaws.com) && (($.eventName = StopConfigurationRecorder) &#124;&#124; ($.eventName = DeleteDeliveryChannel) &#124;&#124; ($.eventName = PutDeliveryChannel) &#124;&#124; ($.eventName = PutConfigurationRecorder)) } </code>   |  ✅  |
|   4.10  |   <code>CloudWatch.10</code>  |   Ensure a log metric filter and alarm exist for Security Group changes  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.10</code>  |   Creates a Log metric & Alarm to monitor Security Group changes using the filter <code> { ($.eventName = AuthorizeSecurityGroupIngress) &#124;&#124; ($.eventName = AuthorizeSecurityGroupEgress) &#124;&#124; ($.eventName = RevokeSecurityGroupIngress) &#124;&#124; ($.eventName = RevokeSecurityGroupEgress) &#124;&#124; ($.eventName = CreateSecurityGroup) &#124;&#124; ($.eventName = DeleteSecurityGroup) } </code>   |  ✅  |
|   4.11  |   <code>CloudWatch.11</code>  |   Ensure a log metric filter and alarm exist for changes to Network Access Control Lists  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.11</code>  |   Creates a Log metric & Alarm to monitor NACL changes using the filter <code> { ($.eventName = CreateNetworkAcl) &#124;&#124; ($.eventName = CreateNetworkAclEntry) &#124;&#124; ($.eventName = DeleteNetworkAcl) &#124;&#124; ($.eventName = DeleteNetworkAclEntry) &#124;&#124; ($.eventName = ReplaceNetworkAclEntry) &#124;&#124; ($.eventName = ReplaceNetworkAclAssociation) } </code>   |  ✅  |
|   4.12  |   <code>CloudWatch.12</code>  |   Ensure a log metric filter and alarm exist for changes to Network Gateways  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.12</code>  |   Creates a Log metric & Alarm to monitor Network Gateway changes using the filter <code> { ($.eventName = CreateCustomerGateway) &#124;&#124; ($.eventName = DeleteCustomerGateway) &#124;&#124; ($.eventName = AttachInternetGateway) &#124;&#124; ($.eventName = CreateInternetGateway) &#124;&#124; ($.eventName = DeleteInternetGateway) &#124;&#124; ($.eventName = DetachInternetGateway) } </code>   |  ✅  |
|   4.13  |   <code>CloudWatch.13</code>  |   Ensure a log metric filter and alarm exist for Route Table changes  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.13</code>  |   Creates a Log metric & Alarm to monitor Route Table changes using the filter <code> { ($.eventName = CreateRoute) &#124;&#124; ($.eventName = CreateRouteTable) &#124;&#124; ($.eventName = ReplaceRoute) &#124;&#124; ($.eventName = ReplaceRouteTableAssociation) &#124;&#124; ($.eventName = DeleteRouteTable) &#124;&#124; ($.eventName = DeleteRoute) &#124;&#124; ($.eventName = DisassociateRouteTable) } </code>   |  ✅  |
|   4.14  |   <code>CloudWatch.14</code>  |   Ensure a log metric filter and alarm exist for VPC changes  |   <code>cis-aws-foundations-benchmark/v/1.4.0/4.14</code>  |   Creates a Log metric & Alarm to monitor VPC changes using the filter <code> { ($.eventName = CreateVpc) &#124;&#124; ($.eventName = DeleteVpc) &#124;&#124; ($.eventName = ModifyVpcAttribute) &#124;&#124; ($.eventName = AcceptVpcPeeringConnection) &#124;&#124; ($.eventName = CreateVpcPeeringConnection) &#124;&#124; ($.eventName = DeleteVpcPeeringConnection) &#124;&#124; ($.eventName = RejectVpcPeeringConnection) &#124;&#124; ($.eventName = AttachClassicLinkVpc) &#124;&#124; ($.eventName = DetachClassicLinkVpc) &#124;&#124; ($.eventName = DisableVpcClassicLink) &#124;&#124; ($.eventName = EnableVpcClassicLink) } </code>   |  ✅  |

> Note for Customization: 
> For all the above Monitoring control remediations, you can modify the following input variables under each control’s <code>if condition</code> in the [lambda_function.py](./main/lambda_function.py)
>> 1. <code>log_group_name</code> – Name of the log group that needs to be monitored.
>> 2. <code>alarm_sns_topic</code> – SNS topic ARN that needs to be notified when the Alarm threshold limit is reached.
>> 3. <code>threshold_value</code> – Choose your desired threshold limit `(default = 1)`.

##### D) Networking Controls 

| CIS Control ID | AWS Control ID | Control Description | Generator ID | Action Taken | Impact |
|----------|----------|----------|----------|----------|----------|
|   5.1  |   <code>EC2.21</code>  |   Network ACLs should not allow ingress from 0.0.0.0/0 to remote administration ports  |   <code>security-control/EC2.21</code>  |   Removes the rule that allows connections from <code>ANY sources to port 22 & 3389, and also ANY to ANY</code> *Be cautious while removing ANY-ANY inbound rule, as NACL is stateless* |    ❗  |
|   5.3  |   <code>EC2.2</code>  |   VPC default security groups should not allow inbound or outbound traffic  |   <code>cis-aws-foundations-benchmark/v/1.4.0/5.3</code>  |   Removes all inbound & outbound rules from the Default security group of a VPC  |  ⚠️  |

> Note: 
> 1. For CIS 5.1 remediation, I am working on modifying the code, so that the auto-remediation is done not by removing the non-compliant resource but by replacing the source IP as a private IP range. This will ensure that only users connected to the Organization network directly or via VPN can access services via remote administration ports.

#### Setup EventBridge Rule for Automatic Remediation

> For the above controls, the EventBridge Rule is set to be triggered automatically using an EventBridge rule that needs to be created. Follow the first three steps as indicated, while creating the previous Custom-Action-based EventBridge rule. 
Here, only the Event Pattern will change as given below:

1. Choose the <code>Event type</code> as <code>Security Hub Findings – Imported</code> & Compliance Status as <code>FAILED</code>.
2. Choose the other <code>Event Type Specifications</code> as per your requirements,and modify the <code>Workflow status</code> as <code>NEW</code>
3. Now, enter all the <code>GeneratorId</code> of the controls that you want to be automatically remediated.
4. Click Next & choose the <code>Target</code> as the <code>Remediation Lambda function</code> and <code>Create Rule</code>.

   ![Event Pattern - Autotrigger](./screenshots/eventbridge_pattern_auto.png)

If you want to disable this Automatic Remediation, you can click on the EventBridge Rule you had created, and choose <code>Disable</code>

> Note: 
> For all the controls supporting auto-remediation, once remediation is done, the lambda function will send an email notification to the SNS topic (created earlier using CloudFormation template [CIS_Remediation_Notification_Setup.yml](./CloudFormation_Templates/CIS_Remediation_Notification_Setup.yml)) with information about the actions taken. 

_Sample Email Notification mentioning remediation actions taken_

![Sample Email Notification mentioning remediation actions taken](./screenshots/email_auto.png)

> Also, once a control that is in <code>FAILED</code> state has triggered the remediation action, its workflow state will change from <code>NEW</code> into <code>NOTIFIED</code> until otherwise, it changes to <code>RESOLVED</code> state, to avoid accidental manual triggers for remediation that have already happened.

## 6. TEST RESULTS

> Test Case:
> > CIS Control ID 5.1 - Network ACLs should not allow ingress from 0.0.0.0/0 to Remote Administration ports.

- NACL with non-compliant rules
  
    ![NACL with non-compliant rules](./screenshots/test_bad_nacl.png)
- Security Hub Compliance Status showing <code>FAILED</code>

    ![Security Hub Compliance Status showing FAILED](./screenshots/test_compliancy_fail.png)
- This <code>FAILED</code> & <code>NEW</code> status will automatically trigger the Remediation Lambda function to perform the remediation action in the respective member account, in the region where the non-compliant resource exists. See the below CloudWatch log of the Remediation Lambda for reference.

    ![CloudWatch logs indicating remediation lambda execution](./screenshots/test_cloudwatchlogs.png)
The below email notification has been sent to the emails subscribed to the SNS topic created with the CloudFormation template [CIS_Remediation_Notification_Setup.yml](./CloudFormation_Templates/CIS_Remediation_Notification_Setup.yml).
  
    ![Email notification showing remediation action taken](./screenshots/test_email.png)
- Once auto-remediation is performed, you can confirm that the non-compliant rules have been removed from the NACL.
  
    ![NACL rules after auto-remediation](./screenshots/test_good_nacl.png)
- During the next check done by Security Hub, the Compliancy status will become <code>PASSED</code>.
  
    ![Security Hub Compliance Status showing PASSED after auto-remediation](./screenshots/test_compliancy_pass.png)

### 7. CONCLUSION

#### Future Work Prospectives

Some of the future prospectives of this project include,
- Find a way to perform security checks for controls not supported by AWS Security Hub (e.g., CIS 1.1, 1.2, etc.).
- Design a more adaptable solution, which will also perform remediation for other industry security standards like NIST, PCI DSS, HIPAA, SOC2, etc.
- "Fully" automate the deployment of our solution using CloudFormation templates.
- Add more remediation conditions for control remediations that "may" cause a production impact

#### Disclaimer

All the remediation codes provided in this repository have been tested under a Test AWS Organization Environment setup. Before you try to implement this in your environment, review the entire documentation and the code.

#### Acknowledgements

I want to express my gratitude to the following individuals for their contributions, support, and inspiration in the development of this project:

- Other Contributors:
   - v3.0.0 - [Yang Han](https://github.com/WarrenHan1130), [Yarui Qiu](https://github.com/LottieQ)
   
- Mentor:
  - [Mohammad Reza Bagheri](https://github.com/BagheriReza)

### Issues and discussions

For any issues or concerns in the code or implementation procedure, please post them in the `Issues` or `Discussions` tab of this repository.
