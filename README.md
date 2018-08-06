# awshelper - helpful tools to work better with AWS

## Month-to-date (MTD) costs of an AWS Account or an AWS Organization
bash script that works with AWSCLI and jq to obtain the amortized month-to-date cost overview for a specific AWS account or a complete AWS Organization. Running withput arguments the costs are retrieved for the current month and year. Dates set in the future are not taken into account.

### Script arguments can be
- -h for help
- -m for another month as of current month (two digits 01-12)
- -y for another year as of current year (four digits 2000 - 2999)
- -c to source environment variables directly (has priority over 'aws configure' set values)

**Usage** is

`$ ./aws-mtd-cost`

or

`$ ./aws-mtd-cost -h -m MM -y YYYY -c PATH-TO-CREDFILE`

---
**PREREQUISITES in AWS**

IAM User in AWS exists

- with programmatic access
- with granted right to query AWS Cost Explorer (`$ aws ce ...`) (Policy: Cost Explorer Service with Full Access)
- with granted right to describe-accounts (Create Inline Policy from AWS Managed Policy AWSConfigRoleForOrganizations
  
**AWS MTD COST EXPLORER: NECESSARY POLICIES ATTACHED TO IAM USER**\
```
{
	"Version": "2012-10-17",
	"Statement": [
		{	 
		"Effect": "Allow",
 		"Action":[
					"ce:*",
					"organizations:ListAccounts",
					"organizations:DescribeAccount",
					"organizations:DescribeOrganization",
					"organizations:ListAWSServiceAccessForOrganization"
				],
 		"Resource": "*"
		}
    ]
}
```

**PREREQUISITES on Client**

- awscli is installed (https://github.com/aws/aws-cli)
- jq is installed (https://stedolan.github.io/jq/)

**Bash Environment Variables are set**

either source them or 'aws configure' the credentials in .aws/credentials

the following content is the minimum setting in the file
```
export AWS_ACCESS_KEY_ID=<YOUR-AWS-ACCESS-KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR-AWS-SECRET-ACCESS-KEY>
```





