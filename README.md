# Amazon Inspector AMI Scanning Solution

This repository provides the Cloudformation templates related to the solution. The solution blog post can be found here - https://aws.amazon.com/blogs/security/how-to-scan-ec2-amis-using-amazon-inspector/

Scheduled-Multi-AMI-Scanner CloudFormation Template - used for fetching AMI's to be scanned on your required tagging preferences. Scans are scheduled using a scheduled EventBridge rule which is disabled initially. This can be enabled when you confirm the scheduled day and time you wish to use for the solution.

### Prerequisites
#### Activate Inspector
The solution requires that you activate Amazon Inspector in your AWS account - https://docs.aws.amazon.com/inspector/latest/user/getting_started_tutorial.html
This can alternatively be achieved by using the AWS Command Line Interface (CLI) https://aws.amazon.com/cli/ and this GitHub example https://github.com/aws-samples/inspector2-enablement-with-cli. 

#### Supported AMI
Make sure that the AMI to be scanned by Amazon Inspector is based from one of the operating systems that AWS supports (https://docs.aws.amazon.com/inspector/latest/user/supported.html#supported-os-ec2) for EC2 scanning.

#### AWS Systems Manager SSM Agent
To successfully complete a scan, Amazon Inspector requires the EC2 instance to be a managed instance in AWS Systems Manager that has the Systems Manager Agent installed and running, and has an attached AWS Identity and Access Management (IAM) instance profile that allows Systems Manager to manage the instance. For more information, see https://docs.aws.amazon.com/inspector/latest/user/scanning-ec2.html.

#### EBS Encyrption
If you use customer managed keys to encrypt Amazon Elastic Block Store (Amazon EBS) volumes and you have a default EC2 configuration set to encrypt EBS volumes, you will need to configure additional key policy permissions. For the customer managed key that encrypts EBS volumes, add the following example policy statement to the key policy. Make sure to replace <111122223333> with your own AWS account ID.

            {"Sid": "Allow use of the key by AMI Scanner State Machine",
                        "Effect": "Allow",
                        "Principal": {
                            "AWS": "arn:aws:iam:: <111122223333>:role/service-role/AMIScanner-Statemachine-role"
                        },
                        "Action": [
                            "kms:Encrypt",
                            "kms:Decrypt",
                            "kms:ReEncrypt*",
                            "kms:GenerateDataKey*",
                            "kms:DescribeKey"
                        ],
                        "Resource": "*"}
        
If you don't add this additional policy, the Step Functions state machine won’t allow the EC2 instances to launch. More info - https://docs.aws.amazon.com/autoscaling/ec2/userguide/key-policy-requirements-EBS-encryption.html#policy-example-cmk-access


## Solution Architecture Overview

![Multiple AMI Scanning - Solution Overview drawio](https://github.com/aws-samples/inspector-ami-scanning-solution/assets/102709027/cbad18d5-c4ad-4338-83a4-b02f2e80039e)

The solution invokes a Lambda function on a scheduled basis that identifies AMIs with the appropriate tags and passes parameters to the Step Functions workflow. 
            
The high-level overview of the full solution has the following steps:
1.	You can use   EventBridge to create a scheduled rule to invoke a Lambda function. You can set the rule for daily, weekly, or monthly, depending on your use case.
2.	The Lambda function searches for AMIs with the appropriate tags and passes these as parameters to the Step Functions workflow.
3.	The first Step Functions state machine is invoked for each AMI to be scanned.
4.	The first Step Functions workflow deploys a temporary EC2 instance from the AMI that is defined.
5.	A Lambda function is invoked to create an EventBridge rule.
6.	An EventBridge rule is created to listen for the successful Amazon Inspector scanned event of the temporary EC2 instance.
7.	A Lambda function is invoked to tag the EC2 instance.
8.	The temporary EC2 instance is tagged, showing Amazon Inspector that scanning is in progress.
9.	The first Step Functions workflow sends a notification to an SNS topic.
10.	The EventBridge rule parses the required parameters and invokes the second Step Functions state machine.
11.	A Lambda function is invoked to generate an Amazon Inspector report and export the findings to an S3 bucket.
12.	The scanned Amazon Inspector AMI results are saved to an S3 bucket.
13.	The Step Functions workflow terminates the temporary EC2 instance that can reduce cost   and clean up the process.
14.	A Lambda function is invoked to delete the temporary EventBridge rule.
15.	The temporary EventBridge rule and targets are deleted.
16.	A Lambda function is invoked to tag the AMI.
17.	The scanned AMI is updated with tagging metadata.
18.	The second Step Functions workflow sends a final notification to an SNS topic.


#### AMI Tagging

To use this solution, you need to tag the AMIs that Amazon Inspector will scan, because a Lambda function will use these tags to start the solution orchestration. For this solution, we use the tag InspectorScan with a value of true. With AMI tagging, you can configure automated processes as part of your deployment pipelines to implement the tagging.

#### Deploy the updated solution

#### Step 1: Deploy the CloudFormation Template

Make sure you deploy the CloudFormation template provided for Multi-AMI Scanning within the AWS account and AWS Region where you want to test this solution.

1.	Choose the following Launch Stack button to launch a CloudFormation stack in your account. Note that the stack will launch in the N. Virginia (us-east-1) Region. To deploy this solution into other AWS Regions, download the solution’s CloudFormation template, modify it, and deploy it to the selected Region. Make sure that you configure the following parameters in the CloudFormation template so that it deploys successfully:
            
    AMITagName – the AMI tag name to check if the AMI should be scanned by Amazon Inspector
    AMITagValue – the AMI tag value to check if the AMI should be scanned by Amazon Inspector
    InspectorReportFormat – the report format, which can be either CSV or JSON
    InstanceSubnetID – the subnet ID to launch the temporary EC2 instance into
    InstanceType – the instance type to deploy the AMI to for temporary scanning purposes 
    KmsKeyAdministratorRole – the existing IAM role that needs to have administrator access to the KMS key created that provides access to encrypt and decrypt the Amazon Inspector report
    S3ReportBucketName – the name of the S3 bucket to be created 
    SnsTopic – the name of the new SNS topic to be created; defines the SNS topic that notifications are published to 

2.	Review the stack name and the parameters for the template. 
3.	On the Quick create stack screen, scroll to the bottom and select I acknowledge that AWS CloudFormation might create IAM resources.
4.	Choose Create stack. The deployment of the CloudFormation stack will take 3–4 minutes. 


#### Step 2: Manually run the first Step Functions workflow

The first Step Functions state machine requires parameters to be passed in; the SingleAMI Lambda function accomplishes this. You can start the Lambda function by creating a test event and passing the correct JSON text and parameters. 
            
The following parameters are available in the output section of the CloudFormation stack that the solution deployed:
    •	AmiId – The ID of the AMI to be used for deploying the EC2 instance. This is the EC2 AMI to be scanned.
    •	EC2InstanceProfile – The Amazon Resource Name (ARN) of the EC2 instance profile that the CloudFormation stack created.
    •	InstanceType – The type of EC2 instance to use for deployment. 
    •	KmsKeyName – The ARN of the KMS key to be used for encrypting and decrypting the Amazon Inspector report that the CloudFormation stack created.
    •	S3Bucket – The name of the S3 bucket to which the Amazon Inspector reports will be exported. The S3 bucket was created previously by the CloudFormation stack.
    •	S3ReportFormat – The report format that Amazon Inspector will use to export the findings report; either the JSON or the CSV format is valid.
    •	SnsTopc – The ARN of the SNS topic to which notifications will be sent. This SNS topic was created previously by the CloudFormation stack.
    •	StateMachineArn – The ARN of the first Step Functions state machine, which the Lambda function will run first.
    •	SubnetId – The ID of the VPC subnet to which the EC2 instance will be attached and launched into. This is a required parameter and could be a subnet that is created specifically for this scanning purpose.

The following is an example parameter configuration and JSON that you can use to run the Lambda function. Make sure to replace each <user input placeholder> with your own information. 
```json
            {
            "AmiId" : "<AMI-ABCDEF01234567890>",
            "Ec2InstanceProfile" : "arn:aws:iam:: <111122223333>:instance-profile/Ec2InstanceLaunchRole",
            "InstanceType" : "t3.medium",
            "KmsKeyName" : "arn:aws:kms:region-name: <111122223333>:key/<a1b2c3d4-5678-90ab-cdef-EXAMPLE11111>",
            "S3Bucket" : "<DOC-EXAMPLE-BUCKET-111122223333>",
            "S3ReportFormat" : "CSV",
            "SnsTopic" : "arn:aws:sns:region-name-2: <111122223333>:InspectorScanner",
            "StateMachine": "arn:aws:states:region-name: <111122223333>:stateMachine:AMIScanner-Part1-LaunchEC2",
            "SubnetId" : "<SUBNET-ABCDEF01234567890>"
            }
```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

