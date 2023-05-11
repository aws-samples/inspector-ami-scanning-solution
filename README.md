# Amazon Inspector V2 AMI Scanning Solution

This repository provides the Cloudformation templates related to the solution.

There are two separate solutions and therefore separate cloudformation templates which can be used;
1. Single-AMI-Scanner - used for passing into the Cfn template a single AMI ID for scanning by Amazon Inspector
2. Scheduled-Multi-AMI-Scanner - used for fetching AMI's to be scanned by on your required tagging preferences. Scans are scheduled using a scheduled Eventbridge rule

### Prerequisites
#### AMI
You will need at least one EC2 AMI which you have created with a supported operating system which Amaon Inspector is able to scan

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


A Lambda function will be used to define and pass parameters to the first AWS Step Functions step and workflow. This workflow step function will launch a temporary EC2 instance from the AMI which has been chosen to be scanned and tag the temporary EC2 instance. An EventBridge rule will be created to listen for the successful Inspector scan event message for the temporary EC2 instance. This is required as the Inspector scanning process of the temporary EC2 instance could take some time to complete depending on a number of factors including instance operating system and account limitations.

Once the Inspector scan has completed successfully, the EventBridge rule will match the successful scan event and trigger the second Step Functions state machine. The second state machine will invoke a Lambda function to export the Inspector findings for the AMI to an Amazon S3Amazon S3 bucket. Another Lambda function is triggered to add a tag to the AMI which has been scanned which includes the location of the Amazon S3 bucket where the Inspector findings are located. A notification is sent to an Amazon SNSAmazon SNS topic with the AMI which was scanned, the status, the S3 bucket location for the report findings and the temporary EC2 instance id which was launched. 

The high level workflow of the solution is as follows;
1.	You provide parameters to the SingleAMI Lambda function as JSON. 
2.	A Lambda function invokes the first AWS Step Functions state machine.
3.	The first AWS Step Functions workflow deploys a temporary EC2 instance from the AMI which is defined.
4.	A Lambda function is invoked to create an EventBridge rule.
5.	An EventBridge rule is created to listen for the successful Inspector scanned event of the temporary EC2 instance.
6.	A Lambda function is invoked to tag the EC2 instance.
7.	The temporary EC2 instance is tagged showing the Inspector scanning is in progress
8.	The first AWS Step Functions workflow will send a notification to a SNS topic.
9.	The EventBridge rule parses the required parameters and triggers the second AWS Step Functions state machine.
10.	A Lambda function is invoked to generate an Inspector report and export the findings to a S3 bucket.
11.	The scanned AMI Inspector results are saved to a S3 bucket.
12.	The AWS Step Functions workflow will terminate the temporary EC2 instance.
13.	A Lambda function is invoked to delete the temporary EventBridge rule.
14.	The temporary EventBridge rule and targets are deleted.
15.	A Lambda function is invoked to tag the AMI
16.	The scanned AMI is updated with tagging metadata.
17.	The second AWS Step Functions workflow will send a notification to a SNS topic.


#### Prerequisite: 
Activate Amazon Inspector in your AWS Account

#### Step 1: Deploy the CloudFormation Template

Make sure you deploy the CloudFormation template provided for Single AMI Scanning within the AWS account and AWS Region where you want to test this solution.
1.	Choose the following Launch Stack button to launch a CloudFormation stack in your account.
The CloudFormation template requires the following parameters to be configured prior to deploying successfully:

InspectorReportFormat – Specifies the report format which can either 'CSV' or 'JSON’

InstanceType - Used to define which instance type to deploy the AMI to for temporary scanning purposes 

InstanceSubnetID - the subnet ID used to launch the temporary EC2 instance into

ScannedAMIID – the id of the AMI which is to be scanned by Inspector

S3ReportBucketName – the name of the S3 bucket to be created as part of the solution

KmsKeyAdministratorRole - Used to define which exiting IAM role needs to have Administrator access to the Kms key created which provides access to encrypt and decrypt the Inspector Report

SnsTopic – A name of a new topic to be created which is used to define which SNS topic notifications are published to 

2.	Review the stack name and the parameters for the template. 
3.	Scroll to the bottom of the Quick create stack screen and select the checkbox next to I acknowledge that AWS CloudFormation might create IAM resources.
4.	Choose Create stack. The deployment of this CloudFormation stack will take 3–5 minutes. 

After the CloudFormation stack has deployed successfully, you can use the deployed solution. 

#### Step 2: Running the first step functions workflow

The first Step Functions state machine requires parameters to be passed in which will be done with the SingleAMI Lambda function. The Lambda function can be started by creating a test event and passing the correct JSON text and parameters. The following parameters will be available in the output section of the CloudFormation stack which the solution deployed. 

•	AmiId - the ID of the AMI to be used for deploying the EC2 instance. This is the EC2 AMI to be scanned.

•	EC2InstanceProfile - the ARN of the EC2 instance profile which was created by the CloudFormation stack.

•	InstanceType - the type of EC2 instance to use for deployment. For the purposes of testing and to enable rapid scanning, it is recommended this instance has enough resources available.

•	KmsKeyName - the ARN of the KMS key to be used for encrypting and decrypting the Inspector Report which was created by the CloudFormation stack.

•	S3Bucket - The S3 bucket name where the Amazon Inspector reports will be exported to which was created by the CloudFormation stack.

•	S3ReportFormat - either JSON or CSV report formats are valid.

•	SnsTopc - the ARN of the SNS topic which was created earlier for sending notifications to which was created by the CloudFormation stack.

•	StateMachineArn – the ARN of the first Step Functions state machine which will be executed first by the Lambda function.

•	SubnetId - the VPC subnet ID where the EC2 instance will be attached and launched into. This is a required parameter and could be a subnet created specifically for this scanning purpose.

An example parameter configuration and JSON which can be used to execute the Lambda function is as follows:

{
"AmiId" : "ami-abcdef01234567890",
"Ec2InstanceProfile" : "arn:aws:iam:: 111122223333:instance-profile/Ec2InstanceLaunchRole",
“InstanceType" : "t3.medium",
"KmsKeyName" : "arn:aws:kms:region-name: 111122223333:key/a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
"S3Bucket" : "DOC-EXAMPLE-BUCKET-111122223333",
"S3ReportFormat" : "CSV",
"SnsTopic" : "arn:aws:sns:region-name-2: 111122223333:InspectorScanner",
"StateMachine": "arn:aws:states:region-name: 111122223333:stateMachine:AMIScanner-Part1-LaunchEC2",
"SubnetId" : "subnet-abcdef01234567890"
}


Once the first state machine is finished, the EventBridge rule will listen for the successful Inspector scan event. A SNS notification will appear similar to this example;

{"AWS Inspector AMI Scan status":"EC2 instance","For AMI":"ami-abcdef01234567890","Temporarily launched AMI using instance":"i-abcdef01234567890"}

Once Inspector has finished scanning the EC2 instance and the second state machine completes successfully, the Inspector finding report will appear in the S3 bucket and notifications will appear on the SNS topic which was created. The SNS notification will appear similar to this example;

{"AWS Inspector AMI Scan completed":"Successfully","For AMI":"ami-abcdef01234567890","AWS Inspector report located at S3 Bucket":"DOC-EXAMPLE-BUCKET-111122223333","Temporarily launched AMI using instance":"i-abcdef01234567890"}


### Multi-AMI Scheduled Scanner Solution

![Multiple AMI Scanning - Solution Overview drawio](https://github.com/aws-samples/inspector-ami-scanning-solution/assets/102709027/7afc3368-3c7b-45be-abbc-cfc2142f69f6)

We can extend the solution to handle multiple AMIs and automatic scheduling. The extended solution triggers a Lambda function on a scheduled basis which identifies AMIs with the appropriate tags and passes parameters to the Step Functions workflow. The rest of the solution is unchanged from the first part of the solution.

The high-level overview of the full solution is as follows;
1.	A scheduled rule is created with Amazon EventBridge which could be daily, weekly or monthly depending on your use case to trigger a Lambda function.
2.	The Lambda function searches for AMIs with the appropriate tags and passes these as parameters to the Step Functions workflow.
3.	The first Step Functions state machine  is invocated for each AMI to be scanned.
4.	The first AWS Step Functions workflow deploys a temporary EC2 instance from the AMI which is defined.
5.	A Lambda function is invoked to create an EventBridge rule.
6.	An EventBridge rule is created to listen for the successful Inspector scanned event of the temporary EC2 instance.
7.	A Lambda function is invoked to tag the EC2 instance.
8.	The temporary EC2 instance is tagged showing the Inspector scanning is in progress
9.	The first AWS Step Functions workflow will send a notification to a SNS topic.
10.	The EventBridge rule parses the required parameters and triggers the second AWS Step Functions state machine.
11.	A Lambda function is invoked to generate an Inspector report and export the findings to a S3 bucket.
12.	The scanned AMI Inspector results are saved to a S3 bucket.
13.	The AWS Step Functions workflow will terminate the temporary EC2 instance.
14.	A Lambda function is invoked to delete the temporary EventBridge rule.
15.	The temporary EventBridge rule and targets are deleted.
16.	A Lambda function is invoked to tag the AMI
17.	The scanned AMI is updated with tagging metadata.
18.	The second AWS Step Functions workflow will send a notification to a SNS topic.

#### AMI Tagging

To use this solution, AMIs which will be scanned by Amazon Inspector need to be tagged. For the purposes of this blog post, we have used the tag “InspectorScan” with a value of “true”. This ensures an AMI cannot be scanned unless the intended tagging strategy is configured. AMI tagging allows you to configure automated processes as part of your deployment pipelines to implement the tagging.

#### Deploy the updated solution

#### Step 1: Deploy the CloudFormation Template

Make sure you deploy the CloudFormation template provided for Multi-AMI Scanning within the AWS account and AWS Region where you want to test this solution.

1.	Choose the following Launch Stack button to launch a CloudFormation stack in your account.
The CloudFormation template requires the following parameters to be configured prior to deploying successfully:

AMITagName - the AMI tag name that will be used for checking if the AMI should be Inspector scanned

AMITagValue - the AMI tag value that will be used for checking if the AMI should be Inspector scanned

InspectorReportFormat – Specifies the report format which can either 'CSV' or 'JSON’

InstanceSubnetID - the subnet ID used to launch the temporary EC2 instance into

InstanceType - Used to define which instance type to deploy the AMI to for temporary scanning purposes 

KmsKeyAdministratorRole - Used to define which exiting IAM role needs to have Administrator access to the Kms key created which provides access to encrypt and decrypt the Inspector Report

S3ReportBucketName – the name of the S3 bucket to be created as part of the solution

SnsTopic – A name of a new topic to be created which is used to define which SNS topic notifications are published to 

2.	Review the stack name and the parameters for the template. 
3.	Scroll to the bottom of the Quick create stack screen and select the checkbox next to I acknowledge that AWS CloudFormation might create IAM resources.
4.	Choose Create stack. The deployment of this CloudFormation stack will take 3–4 minutes. 

#### Step 2: Invoking the Lambda function

There are a number of environment variables which will be defined by the CloudFormation template and passed into the “AMIScanner- GetAMIs” Lambda function by the EventBridge schedule task. The parameter configuration is used as input constant configuration to the target Lambda function on the EventBridge rule. 

{
  "INSTANCE_TYPE": "t3.medium",
  "SUBNET_ID": "subnet-abcdef01234567890",
  "INSTANCE_PROFILEARN": "arn:aws:iam:: 111122223333:instance-profile/SingleAMIScanner-EC2InstanceProfile",
  "SNS_TOPICNAME": "arn:aws:sns:region:111122223333:InspectorScanner",
  "S3REPORTBUCKET": "DOC-EXAMPLE-BUCKET-111122223333",
  "AMI_SCANTAG_NAME": "InspectorScan",
  "AMI_SCANTAG_VALUE": "true",
  "KMSKEY_NAME": "arn:aws:kms:region: 111122223333:key/a1b2c3d4-5678-90ab-cdef-EXAMPLE11111x",
  "STATE_MACHINEARN": "arn:aws:states:ap-southeast-2: 111122223333:stateMachine:AMIScanner-Part1-LaunchEC2",
  "INSPECTOR_REPORTFORMAT": "CSV"
}


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

