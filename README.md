## Amazon Inspector V2 AMI Scanning Solution

This repository provides all the Cloudformation templates, Lambda functions and Step Functions codes related to the solution.

There are two cloudformation templates which can be used;
1. Single-AMI-Scanner = used for passing into the Cfn template a single AMI ID for scanning by Amazon Inspector
2. Scheduled-Multi-AMI-Scanner = used for fetching AMI's to be scanned by on your required tagging preferences. Scans are scheduled using a scheduled Eventbridge rule

Solution Architecture Overviews
Single AMI Scanner Solution
![Single AMI Scanning - Solution Overview drawio](https://github.com/aws-samples/inspector-ami-scanning-solution/assets/102709027/35ff5f4d-3f70-4241-9df4-9a295ede94aa)


Multi-AMI Scheduled Scanner Solution
![Multiple AMI Scanning - Solution Overview drawio](https://github.com/aws-samples/inspector-ami-scanning-solution/assets/102709027/b271d5d2-9dd7-4df5-a946-3dee082e4b77)

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

