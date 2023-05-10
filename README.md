## Amazon Inspector V2 AMI Scanning Solution

This repository provides all the Cloudformation templates, Lambda functions and Step Functions codes related to the solution.

There are two cloudformation templates which can be used;
1. Single-AMI-Scanner = used for passing into the Cfn template a single AMI ID for scanning by Amazon Inspector
2. Scheduled-Multi-AMI-Scanner = used for fetching AMI's to be scanned by on your required tagging preferences. Scans are scheduled using a scheduled Eventbridge rule


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

