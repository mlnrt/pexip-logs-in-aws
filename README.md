# pexip-logs-on-aws
Pexip Infinity log analysis on the AWS cloud

# What is the purpose of this project?
This is an experiment of using AWS built-in log management, query and analytics tools to help with troubleshooting and searching through the logs of the Pexip Infinity video-conferencing platform.

# How does it work?
I have created a demo which I published on YouTube in two parts:
  - part 1: shows how to get the logs into AWS CloudWatch Logs, how the provided Lambda function wrangles the logs in JSON and another function exports them into back into CloudWatch Logs and/or S3: 
  - part 2: -- coming soon --

# What does the code repo contain?
It contains
  - an AWS CloudFormation YAML template which will provision most of the needed resources (IAM roles, CloudWatch Logs log groups, Lambda Functions, Glue databases and tables...),
  - the code for the Lambda functions,
  - a configuration sample file for the AWS CloudWatch Logs Agent which must be installed on the Syslog server,
  - a log metadata file, containing the data structure of all the types of Pexip Infinity logs I have seen in my lab. It is used by the Lambda function which creates all the AWS Glue tables and partitions based on the processed log files already stored in S3. If some types of logs are missing (and some are for sure, like logs for the integration with Microsoft SfB and Teams, which I do not have in my lab) in this case run the provided Glue Crawler (provisioned through the CloudFormation template) against the corresponding log folder in S3.

# What does the CloudFormation template include?
The CloudFormation template allows you to choose if you want to:
  - process in AWS both the "audit" logs (the OS/process-level application layer logs of the Pexip Infinity servers) and the "support" logs (the higher level application logs, e.g. configuration changes, and communication logs, e.g. SIP, H.323, ICE, REST... communication logs) or just the "support" logs.
  - try to Export the logs after wrangling in S3, to process them in AWS Glue and Athena too, or just use CloudWatch Logs.

Based on your choices, CloudFormation will provision the below resources:
  - The IAM roles, Policies for:
    - the EC2 instance which will host your Syslog server, together with the InstanceProfile to be able to attach the role to the EC2 instance.
    - the Lambda functions
    - the Lambda Permission for the log wrangling function to subscribe to the CloudWatch Logs log stream
    - the Glue databases (if you chose the option #2)
    - the Glue Crawlers (if you chose the option #2)
  - The CloudWatch Logs' log groups for the "support" logs and the "audit logs" (if you chose to process those too)
  - The Lambda functions to wrangle and the Pexip logs
  - The subscription for the log-wrangling Lambda function to the CloudWatch Logs' log groups of the Pexip Infinity "support" and "audit" logs
  - The Lambda functions to create the Glue tables and partitions (if you chose the option #2)
  - The Glue databases and crawlers (if you chose the option #2)

# What the CloudFormation template does NOT include?
The CloudFormation template does not include:
  - the provisioning of an EC2 instance to host the Syslog server
  - configure the Syslog server with the CloudWatch Logs agent. See AWS documentations
    - to install the Agent on a Linux EC2 instance: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html
    - to configure the Agent: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AgentReference.html
  - create all the Glue tables and partitions for all the types of logs. A Lambda function "Pexip_Logs_Create_Glue_Tables" is provided for that, and will have to be run manually with an empty test event "{}" and the environment parameter "CREATE_OR_UPDATE_TABLES_PARTITIONS_AT_THE_SAME_TIME" set to "True"
  
# Lambda Functions logs
All the provided Lambda functions output their own logs into their own CloudWatch Logs' log group in the format "/aws/lambda/function-name" (e.g. "/aws/lambda/Pexip_Logs_Wrangling") for debugging purposes.
For each function there is an environment variable called "DEBUGGING_LOG_LEVEL" which you can use to control the level of logs the function will output to CloudWatch Logs:
  - "Info": will output a very limited amount logs; mainly just the logs about the function start and end, and when it invokes another lambda function (e.g. when the wrangling function evokes the export function),
  - "Detailed": will output, well... detailed logs of each of the steps performed (e.g. when the wrangling function is parsing a log massage)
  - "Debug": will output a lot of data, including the entire JSON event received by the function, the JSON formatted logs after wrangling, the JSON data read in the "pexip_logs_metadata.son" file... This can quickly become a high volume of logs on a production environment so be careful.
If you just want to see what the functions do, use the "Detailed" level. Use the "Debug" level only if you have some logs missing or the functions fail to execute.
By default, the Lambda functions are set in the CloudFormation template with a log level of "Detailed".

# I am missing logs in CloudWatch Logs...
If you find that some logs are missing, here are some possible reasons:
  - adjust on the EC2 Linux Syslog server the CloudWatch Logs Agent "buffer_duration" parameter and the Lambda function "Pexip_Logs_Wrangling" timeout. The buffer_duration controls the time interval during which the CloudWatch Agent will wait and accumulate logs before sending them to CloudWatch Logs. The higher the buffer the more logs are send at once to CloudWatchLogs. If the buffer is too high (one batch of logs will have many, many log messages), and the Lambda function timeout is too low (default=30 seconds), the Lambda function might be terminated, before it was able to wrangle all the logs. Look in the Lambda function's own logs in the CloudWatch Logs' log group /aws/lambda/Pexip_Logs_Wrangling for "Task timed out" events.
  - the "Pexip_Logs_Export" function has timed out before writting all the logs to CloudWatch Logs
  - if you try to use Glue and Athena, there are for sure logs missing in the provided file "pexip_logs_metadata.json"
  - there are bugs in my code
