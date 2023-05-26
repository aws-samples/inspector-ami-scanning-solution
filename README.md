# Amazon Inspector V2 AMI Scanning Solution

This repository provides the Cloudformation templates related to the solution.

There are two separate solutions and therefore separate CloudFormation templates which can be used;
1. Single-AMI-Scanner - used for passing into the Cfn template a single AMI ID for scanning by Amazon Inspector
2. Scheduled-Multi-AMI-Scanner - used for fetching AMI's to be scanned on your required tagging preferences. Scans are scheduled using a scheduled EventBridge rule

### Prerequisites
#### Activate Inspector
The solution requires that you activate Amazon Inspector in your AWS account - https://docs.aws.amazon.com/inspector/latest/user/getting_started_tutorial.html
This can alternatively be achieved by using the AWS Command Line Interface (CLI) https://aws.amazon.com/cli/ and this GitHub example https://github.com/aws-samples/inspector2-enablement-with-cli. 

#### AMI
You will need at least one EC2 AMI which you have created with a supported operating system which Amazon Inspector is able to scan

#### EBS Encyrption
If you are using customer managed keys (CMK) for encrypting EBS volumes and have a default EC2 configuration set to encrypt EBS volumes, additional key policy permissions will be required to be configured.  For the KMS CMK which is used to encrypt EBS volumes, the following example policy statement can be added to the key policy;

{
            "Sid": "Allow use of the key by AMI Scanner State Machine",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam:: 111122223333:role/service-role/AMIScanner-Statemachine-role"
            },
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        },
        
If this additional policy is not added, launching of EC2 instances by the Step Functions state machine will not be permitted. More info - https://docs.aws.amazon.com/autoscaling/ec2/userguide/key-policy-requirements-EBS-encryption.html#policy-example-cmk-access


## Solution Architecture Overviews

### Single AMI Scanner Solution

![Single AMI Scanning - Solution Overview drawio](https://github.com/aws-samples/inspector-ami-scanning-solution/assets/102709027/8f81e598-821c-40c3-a1c6-09b2b14d1b53)

As shown in Figure 1, the high-level workflow of the solution is as follows:
1.	You provide parameters to the SingleAMI Lambda function as JSON text. 
2.	The Lambda function invokes the first Step Functions state machine.
3.	The first Step Functions workflow deploys a temporary EC2 instance from the AMI that is defined.
4.	A Lambda function is invoked to create an EventBridge rule.
5.	An EventBridge rule is created to listen for the successful Amazon Inspector scanned event of the temporary EC2 instance.
6.	A Lambda function is invoked to tag the EC2 instance.
7.	The temporary EC2 instance is tagged, showing that the Amazon Inspector scanning is in progress.
8.	The first Step Functions workflow sends a notification to an SNS topic.
9.	The EventBridge rule parses the required parameters and invokes the second Step Functions state machine.
10.	A Lambda function is invoked to generate an Amazon Inspector report and export the findings to an S3 bucket.
11.	The scanned AMI results for Amazon Inspector are saved to an S3 bucket.
12.	The Step Functions workflow terminates the temporary EC2 instance.
13.	A Lambda function is invoked to delete the temporary EventBridge rule.
14.	The temporary EventBridge rule and targets are deleted.
15.	A Lambda function is invoked to tag the AMI.
16.	The scanned AMI is updated with tagging metadata.
17.	The second Step Functions workflow sends a notification to an SNS topic.

A Lambda function will be used to define and pass parameters to the first AWS Step Functions step and workflow. This workflow step function will launch a temporary EC2 instance from the AMI which has been chosen to be scanned and tag the temporary EC2 instance. An EventBridge rule will be created to listen for the successful Inspector scan event message for the temporary EC2 instance. This is required as the Inspector scanning process of the temporary EC2 instance could take some time to complete depending on a number of factors including instance operating system and account limitations.

Once the Inspector scan has completed successfully, the EventBridge rule will match the successful scan event and trigger the second Step Functions state machine. The second state machine will invoke a Lambda function to export the Inspector findings for the AMI to an Amazon S3Amazon S3 bucket. Another Lambda function is triggered to add a tag to the AMI which has been scanned which includes the location of the Amazon S3 bucket where the Inspector findings are located. A notification is sent to an SNS topic with the AMI which was scanned, the status, the S3 bucket location for the report findings and the temporary EC2 instance id which was launched. 


#### Step 1: Deploy the CloudFormation Template

Make sure that you deploy the CloudFormation template provided for Single AMI Scanning within the AWS account and Region where you want to test the solution.
1.	Choose the following Launch Stack button to launch a CloudFormation stack in your account.
To deploy the template successfully, you need to configure the following parameters:
•	InspectorReportFormat – the report format, which can be either a CSV or JSON file
•	InstanceType – the instance type to deploy the AMI to for temporary scanning purposes 
•	InstanceSubnetID – the subnet ID to launch the temporary EC2 instance into
•	ScannedAMIID – the ID of the AMI to be scanned by Amazon Inspector
•	S3ReportBucketName – the name of the S3 bucket to be created 
•	KmsKeyAdministratorRole – the existing IAM role that needs to have administrator access to the KMS key created, which provides access to encrypt and decrypt the Amazon Inspector report
•	SnsTopic – the name of the new SNS topic to be created; determines which SNS topic notifications are published to 

2.	Review the stack name and the parameters for the template. 
3.	On the Quick create stack screen, scroll to the bottom and select the I acknowledge that AWS CloudFormation might create IAM resources.
4.	Choose Create stack. The deployment of the CloudFormation stack will take 3–5 minutes. 

After the CloudFormation stack has deployed successfully, you can use the deployed solution. 

#### Step 2: Running the first Step Functions workflow

The first Step Functions state machine requires parameters to be passed in; the SingleAMI Lambda function accomplishes this. You can start the Lambda function by creating a test event and passing the correct JSON text and parameters. The following parameters are available in the output section of the CloudFormation stack that the solution deployed. 
•	AmiId – the ID of the AMI to be used for deploying the EC2 instance. This is the EC2 AMI to be scanned.
•	EC2InstanceProfile – the ARN of the EC2 instance profile that was created by the CloudFormation stack.
•	InstanceType – the type of EC2 instance to use for deployment. 
•	KmsKeyName – the ARN of the KMS key to be used for encrypting and decrypting the Amazon Inspector report that was created by the CloudFormation stack.
•	S3Bucket – the name of the S3 bucket where the Amazon Inspector reports will be exported to. The S3 bucket was created previously by the CloudFormation stack.
•	S3ReportFormat – the report format that Amazon Inspector will use to export the findings report; either JSON or CSV formats are valid.
•	SnsTopc – the ARN of the SNS topic to which notifications will be sent. This SNS topic was created previously by the CloudFormation stack.
•	StateMachineArn – the ARN of the first Step Functions state machine, whichthat the Lambda function will run first.
•	SubnetId – the VPC subnet ID where the EC2 instance will be attached and launched into. This is a required parameter and could be a subnet created specifically for this scanning purpose.


The following is an example parameter configuration and JSON that you can use to run the Lambda function. Make sure to replace each <user input placeholder> with your own information. 
{
"AmiId" : "<AMIami-ABCDEFabcdef01234567890>",
"Ec2InstanceProfile" : "arn:aws:iam:: <111122223333>:instance-profile/Ec2InstanceLaunchRole",
“InstanceType" : "t3.medium",
"KmsKeyName" : "arn:aws:kms:region-name: <111122223333>:key/<a1b2c3d4-5678-90ab-cdef-EXAMPLE11111>",
"S3Bucket" : "<DOC-EXAMPLE-BUCKET-111122223333>",
"S3ReportFormat" : "CSV",
"SnsTopic" : "arn:aws:sns:region-name-2: <111122223333>:InspectorScanner",
"StateMachine": "arn:aws:states:region-name: <111122223333>:stateMachine:AMIScanner-Part1-LaunchEC2",
"SubnetId" : "<SUBNETsubnet-ABCDEFabcdef01234567890>"
}


After the first state machine is finished, the EventBridge rule listens for the successful Amazon Inspector scan event. An SNS notification is sent, similar to the following: 
            
{"AWS Inspector AMI Scan status":"EC2 instance","For AMI":"ami-abcdef01234567890","Temporarily launched AMI using instance":"i-abcdef01234567890"}
            
After Amazon Inspector has finished scanning the EC2 instance, and the second state machine completes successfully, the Amazon Inspector finding report appears in the S3 bucket and notifications appear on the SNS topic that was created. The following is an example of an SNS notification:
            
{"AWS Inspector AMI Scan completed":"Successfully","For AMI":"ami-abcdef01234567890","AWS Inspector report located at S3 Bucket":"DOC-EXAMPLE-BUCKET-111122223333","Temporarily launched AMI using instance":"i-abcdef01234567890"}



### Multi-AMI Scheduled Scanner Solution

![Multiple AMI Scanning - Solution Overview drawio](https://github.com/aws-samples/inspector-ami-scanning-solution/assets/102709027/7afc3368-3c7b-45be-abbc-cfc2142f69f6)

We can extend the solution to handle multiple AMIs and automatic scheduling. The extended solution invokes a Lambda function on a scheduled basis that identifies AMIs with the appropriate tags and passes parameters to the Step Functions workflow. The rest of the solution is the same as that presented in the first part of this solution.
            
The high-level overview of the full solution has the following steps:
1.	A scheduled rule is created with EventBridge to invoke a Lambda function. You can set the rule for daily, weekly, or monthly, depending on your use case.
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
13.	The Step Functions workflow terminates the temporary EC2 instance.
14.	A Lambda function is invoked to delete the temporary EventBridge rule.
15.	The temporary EventBridge rule and targets are deleted.
16.	A Lambda function is invoked to tag the AMI.
17.	The scanned AMI is updated with tagging metadata.
18.	The second Step Functions workflow sends a notification to an SNS topic.


#### AMI Tagging

To use this solution, you need to tag the AMIs that Amazon Inspector will scan. For this blog post, we use the tag InspectorScan with a value of true. This helps ensure that Amazon Inspector can’t scan an AMI unless the intended tagging strategy is configured. With AMI tagging, you can configure automated processes as part of your deployment pipelines to implement the tagging.

#### Deploy the updated solution

#### Step 1: Deploy the CloudFormation Template

Make sure you deploy the CloudFormation template provided for Multi-AMI Scanning within the AWS account and AWS Region where you want to test this solution.

1.	Choose the following Launch Stack button to launch a CloudFormation stack in your account.
Make sure that you configure the following parameters in the CloudFormation template so that it deploys successfully:
•	AMITagName – the AMI tag name to check if the AMI should be scanned by Amazon Inspector
•	AMITagValue – the AMI tag value to check if the AMI should be scanned by Amazon Inspector
•	InspectorReportFormat – the report format, which can be either CSV or JSON
•	InstanceSubnetID – the subnet ID to launch the temporary EC2 instance into
•	InstanceType – the instance type to deploy the AMI to for temporary scanning purposes 
•	KmsKeyAdministratorRole – the existing IAM role that needs to have administrator access to the KMS key created that provides access to encrypt and decrypt the Amazon Inspector report
•	S3ReportBucketName – the name of the S3 bucket to be created 
•	SnsTopic – the name of the new SNS topic to be created; defines the SNS topic that notifications are published to 

2.	Review the stack name and the parameters for the template. 
3.	On the Quick create stack screen, scroll to the bottom and select I acknowledge that AWS CloudFormation might create IAM resources.
4.	Choose Create stack. The deployment of the CloudFormation stack will take 3–4 minutes. 


#### Step 2: Invoking the Lambda function

There are a number of environment variables that will be defined by the CloudFormation template and passed into the AMIScanner- GetAMIs Lambda function by the EventBridge schedule task. The parameter configuration is used as input configuration to the target Lambda function on the EventBridge rule. 
The following is an example JSON parameter configuration that you can use to invoke the Lambda function. Make sure to replace each <user input placeholder> with your own information. 

{
  "INSTANCE_TYPE": "t3.medium",
  "SUBNET_ID": "<SUBNETsubnet-ABCDEFabcdef01234567890>",
  "INSTANCE_PROFILEARN": "arn:aws:iam::< 111122223333>:instance-profile/SingleAMIScanner-EC2InstanceProfile",
  "SNS_TOPICNAME": "arn:aws:sns:<region>:<111122223333>:InspectorScanner",
  "S3REPORTBUCKET": "<DOC-EXAMPLE-BUCKET-111122223333>",
  "AMI_SCANTAG_NAME": "InspectorScan",
  "AMI_SCANTAG_VALUE": "true",
  "KMSKEY_NAME": "arn:aws:kms:<region>: <111122223333>:key/<a1b2c3d4-5678-90ab-cdef-EXAMPLE11111>x",
  "STATE_MACHINEARN": "arn:aws:states:<region>ap-southeast-2: <111122223333>:stateMachine:AMIScanner-Part1-LaunchEC2",
  "INSPECTOR_REPORTFORMAT": "CSV"



## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

