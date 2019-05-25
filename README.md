# pexip-logs-on-aws
Pexip Infinity log analysis on the AWS cloud

# What  is the purpose of this project ?
This is an experiment of using AWS built-in log management, query and analytics tools to help with troubleshooting and searching through the logs of the Pexip Infinity video-conferencing platform.

# What does the code repo contains ?
It contains
  - an AWS CloudFormation YAML template which will provision most of the needed resources (IAM roles, CloudWatch Logs log groups, Lambda Functions, Glue databases and tables...),
  - the code for the Lambda functions,
  - a configuration sample file for the AWS CloudWatch Logs Agent which must be installed on the Syslog server,
  - a log metadata file, containing the data structure of all the types of Pexip Infinity logs I have seen in my lab. It is used by the Lambda function which creates all the AWS Glue tables and partitions based on the processed log files already stored in S3. If some types of logs are missing (and some are for sure, like logs for the integration with Microsoft SfB and Teams, which I do not have in my lab) in this case run the provided Glue Crawler (provisioned through the CloudFormation template) against the corresponding log folder in S3.
  
# What do I need to do ?

# What does the CloudFormation template includes ?
The CloudFormation template allows you to choose if you want to:
  - process in AWS both the "audit" logs (the OS/process-level application layer logs of the Pexip Infinity servers) and the "support" logs (the higher level application logs, e.g. configuration changes, and communication logs, e.g. SIP, H.323, ICE, REST... communication logs) or just the "support" logs.
  - try to Export the logs after wrangling in S3, to process them in AWS Glue and Athena too, or just use ClouWatch Logs.

Based on your choices, CloudFormation will provision the below resources:
  - The IAM roles, Policies for:
    - the EC2 instance which will host your Syslog server, together with the InstanceProfile to be able to attach the role to the EC2 instance.
    - the Lambda functions
    - the Lambda Permission for the log wrangling function to subscribe to the CloudWatch Logs log stream
    - the Glue databases (if you chose the option #2)
    - the Glue Crawlers (if you chose the option #2)
  - The CloudWatch Logs log groups for the "support" logs and the "audit logs" (if you chose to process those too)
  - The Lambda functions to wrangle and the Pexip logs
  - The subsciption for the log-wrangling Lambda function to the CloudWatch Logs log groups of the Pexip Infinity "support" and "audit" logs
  - The Lambda functions to create the Glue tables and partitions (if you chose the option #2)
  - The Glue databases and crawlers (if you chose the option #2)

# What the CloudFormation template does NOT includes ?
The CloudFormation template does not include:
  - the provisioning of an EC2 instance to host the Syslog server
  - configure the Syslog server with the CloudWatch Logs agent. See AWS documentation https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html
  - create all the Glue tables and partitions for all the types of logs. A Lambda function "Pexip_Logs_Create_Glue_Tables" is provided for that, and will have to be run manually with an empty test event "{}" and the environment parameter "CREATE_OR_UPDATE_TABLES_PARTITIONS_AT_THE_SAME_TIME" set to "True"

# I am missing logs in CloudWatch Logs...
If you find that some logs are missing, here are some possible reasons:
  - adjust the 
