#!/bin/bash
# ----------------------------------------------------------
# bash script that works with AWSCLI and jq to obtain the 
# amortized month-to-date cost overview for a specific 
# AWS account or a complete AWS Organization. Running with
# no arguments the costs are retrieved for the current month 
# and year. Dates set in the future are not allowed.
# 
# Arguments can be
# -h for help
# -m for another month as of current month (two digits 01-12)
# -y for another year as of current year (four digits 2000 - 2999)
# -c to source environment variables directly (has priority over 'aws configure' set values)
# 
# ----------------------------------------------------------
# copyright 2018: Markus Oponczewski
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES 
# OR CONDITIONS OF ANY KIND, either express or implied.
# ----------------------------------------------------------

helpscreen()
{
cat << EOF
-----
This script retrieves the AWS month-to-date costs (MTD) for an
AWS account or an AWS Organization with individual accounts
If no argument is given the script queries AWS for the current 
month in the current year.
Dates are only up to now, no dates in the future are supported

Usage of this bash script is: $0 OPTIONS
OPTIONS:
   -h 		Show this message
   -m 		Month other as the current one (two digits e.g. 06)
   -y 		Year other as the current one (four digits e.g. 2017)
   -c 		File location with environment variables AWS_SECRET_ACCESS_KEY and AWS_ACCESS_KEY_ID
EOF
}

# set current date
MONTH=`date +"%m"`
YEAR=`date +"%Y"`

# does an 'aws configure' generated file exist ?
awscredfile="$(eval echo ~$USER)/.aws/credentials"
if [ -f $awscredfile ]; then
	# credentials file from 'aws configure' exist
	awsCredBool=1
else
	# no aws credentials file from 'aws configure'
	awsCredBool=0	
fi


# parse args
while getopts “:hm:y:c:” OPTION
do
	case $OPTION in
		h)
			helpscreen
			exit
			;;
		m)
			if [[ $OPTARG =~ ^(0[1-9]|1[012])$ ]]; then
				MONTH=$OPTARG
 			else
				helpscreen
				exit
			fi
 			;;
		y)  
			if [[ $OPTARG =~ ^([2][0-9][0-9][0-9])$ ]] && [ "$OPTARG" -le "`date +"%Y"`" ]; then
				YEAR=$OPTARG
			else
				helpscreen
				exit
			fi
			;;
		c)
			if [[ -f $OPTARG ]]; then
				# a file given in the -c argument exist... source it!
				sourced=`{ source $OPTARG; } 2>&1`
				if [ $? -ne 0 ]; then
					# something went wrong during sourcing environment variables -> bye
					helpscreen
					exit
				fi

				if [ -z "${AWS_ACCESS_KEY_ID}" ] && [ -z "${AWS_SECRET_ACCESS_KEY}" ]; then
					# sourcing was ok, but no AWS credentials have been sourced (instead sth different)
					if [[ ! $awsCredBool ]]; then
						# additionally 'aws configure' has also not been run -> bye
						helpscreen
						exit
					fi
				else
					awsCredBool=1
				fi
			
			else
				# the file given in -c argument does not exist -> bye
				helpscreen
				exit
			fi
			;;
		?)
			helpscreen
			exit
			;;
		esac
done


if [[ ! $awsCredBool ]]; then
	# no aws credentials, neither from -c Argument nor from 'aws configure'
	helpscreen
	exit
fi


# when it's the current year, check if the month is not in the future
if [ $YEAR -eq "`date +"%Y"`" ] && [ $MONTH -gt "`date +"%m"`" ]; then
   helpscreen
   exit
fi


# independent of what is selected as output-format for AWS CLI
# select json as output format to use in this script
TMPCLIFORMAT=$AWS_DEFAULT_OUTPUT
AWS_DEFAULT_OUTPUT=json


# whoami on AWS CLI, obtain ARN of current user 
# independent if IAM user has IAM rights or not
IAMUSERDATA=`{ aws iam get-user; } 2>&1`

if [ $? -eq 0 ]
then
  # IAM user has the appropriate rights so the IAM-USER-ARN can be drawn 
  # from the JSON output
  echo $IAMUSERDATA | sed -e 's/^.*arn\:aws\:/arn\:aws\:/' -e 's/\".*$//'

else
  # IAM user does not have the appropriate IAM rights
  # so the IAM-USER-ARN must be drawn from the Errormessage
  echo $IAMUSERDATA | sed -e 's/^.*arn\:aws\:/arn\:aws\:/' -e 's/ .*$//' 

fi

# set timeperiod
PERIODSTART=$YEAR-$MONTH"-01"
PERIODEND=$YEAR-$MONTH-`cal $MONTH $YEAR| awk 'NF {eom = $NF}; END {print eom}'`

# query AWS Cost Explorer with selected month as time period
# for the whole account or org (Group-Type: TAG)
awscetag=`aws ce get-cost-and-usage \
--time-period "{\"Start\":\"$PERIODSTART\",\"End\":\"$PERIODEND\"}" \
--granularity MONTHLY \
--group-by "{\"Key\":\"LINKED_ACCOUNT\",\"Type\":\"TAG\"}" \
--metrics AmortizedCost \
| jq -r '.ResultsByTime[].Groups[].Metrics[] | "\(.Unit) \(.Amount)"'`

echo "MTD Cost ${PERIODSTART}-${PERIODEND} is ${awscetag}"

# query AWS Cost Explorer with selected month as time period
# for each individual account in the org (Group-Type: DIMENSION)
awscedim=`aws ce get-cost-and-usage \
--time-period "{\"Start\":\"$PERIODSTART\",\"End\":\"$PERIODEND\"}" \
--granularity MONTHLY \
--group-by "{\"Key\":\"LINKED_ACCOUNT\",\"Type\":\"DIMENSION\"}" \
--metrics AmortizedCost \
| jq --compact-output '.ResultsByTime[].Groups[] | {Account: .Keys[], Amount: .Metrics[].Amount, Currency: .Metrics[].Unit}'`

# format output and query AWS-Account-Name with AWS-Account-ID
for row in $(echo "${awscedim}" | sed 's/ /\n/g'); do
	getAccountId() {
		echo ${row} | jq '.Account' | sed 's/\"//g'
	}
	accountId=$(getAccountId '.Account')
	echo "AWS-Account-ID $(echo ${row} | jq -r '. | "\(.Account) \(.Currency) \(.Amount)"' | column -t)\
	$(aws organizations describe-account --account-id ${accountId} | jq -r '.Account.Name' | sed 's/\"//g')"
done

# restore initial state before leaving the script 
AWS_DEFAULT_OUTPUT=$TMPCLIFORMAT
