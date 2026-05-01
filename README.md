week6
PART 1 — Create VPC (Foundation Layer)
Step 1: Open VPC
Login to AWS Console
Search → VPC
Click → Your VPCs
Click → Create VPC

Step 2: Create VPC
Name: MyVPC
CIDR: 10.0.0.0/16
Tenancy: Default
Click Create VPC
PART 2 — Create Subnets

Step 3: Create Public Subnet (Bastion)
Go → Subnets → Create subnet
VPC: MyVPC
Name: Public-Subnet
CIDR: 10.0.1.0/24
AZ: choose any (e.g., ap-south-1a)

Create

Step 4: Create Private Subnet (DB)
Again → Create subnet
Name: Private-Subnet
CIDR: 10.0.2.0/24

Create

PART 3 — Internet Gateway (Public Access)
Step 5: Create IGW
Go → Internet Gateways
Click → Create
Name: MyIGW

👉 Create → Attach to VPC

Step 6: Route Table for Public Subnet
Go → Route Tables → Create
Name: Public-RT
VPC: MyVPC

👉 Create

Add Route:
Destination: 0.0.0.0/0
Target: Internet Gateway
Associate:
Subnet → Public-Subnet
🔐 PART 4 — Security Groups (VERY CRITICAL)

Step 7: Bastion Security Group
Name: Bastion-SG
Inbound Rules:
SSH (22) → My IP only

Step 8: DB Security Group
Name: DB-SG
Inbound Rules:
MySQL (3306) → 10.0.1.0/24
SSH (22) → Bastion Private IP (later update)

DO NOT ADD 0.0.0.0/0 (or you just broke the whole point)

PART 5 — Launch EC2 Instances
Step 9: Bastion Server
Name: Bastion
AMI: Ubuntu
Subnet: Public
Auto-assign Public IP: ENABLE
SG: Bastion-SG
Key: bastion.pem

Launch

Step 10: DB Server
Name: DB-Server
AMI: Ubuntu
Subnet: Private
Public IP: ❌ DISABLE
SG: DB-SG
Key: dbserver.pem

Launch

PART 6 — Copy Key (CRUCIAL STEP)

From your local machine (PowerShell):

scp -i bastion.pem dbserver.pem ubuntu@<BASTION_PUBLIC_IP>:/home/ubuntu/
🔌 PART 7 — SSH Chain (Access Flow)
Step 11: Connect to Bastion
ssh -i bastion.pem ubuntu@<BASTION_PUBLIC_IP>
Step 12: Fix Permissions

Inside bastion:

chmod 400 dbserver.pem
Step 13: Connect to DB
ssh -i dbserver.pem ubuntu@10.0.2.x

If this fails → it’s ALWAYS:

SG issue
wrong private IP
key permissions
PROBLEM (YOU MENTIONED)
apt update / yum update FAILS

WHY?
Because:

Private subnet = NO internet

PART 8 — NAT GATEWAY (Fix Internet)
Step 14: Allocate Elastic IP
Go → Elastic IPs
Allocate new
Step 15: Create NAT Gateway
Go → NAT Gateway
Create:
Subnet: Public Subnet
Elastic IP: Select above

Create (wait 2–3 mins)

Step 16: Private Route Table
Create Route Table:
Name: Private-RT
Step 17: Associate Private Subnet
Attach → Private-Subnet
Step 18: Add Route
Destination: 0.0.0.0/0
Target: NAT Gateway

Save

FINAL TEST

Inside DB server:

ping google.com
sudo apt update

If it works → you built a real-world production architecture

------------------------------------------------------------------------------------------------------------------------------------------------------

week7
PART 0 — Very Important Setup Rules

Use same AWS Region for everything.

Example:

Region: us-east-1
S3 bucket: us-east-1
Lambda: us-east-1
DynamoDB: us-east-1

Do not create S3 in one region and Lambda in another. That causes avoidable pain.

PART 1 — Create S3 Bucket
Step 1: Open S3
Login to AWS Console.
Search S3.
Click S3.
Click Create bucket.
Step 2: Bucket settings

Fill like this:

Bucket name: my-lambda-upload-bucket-pranathi
Region: same as Lambda region
Object Ownership: ACLs disabled
Block Public Access: keep all checked
Bucket Versioning: Disable
Default encryption: SSE-S3

Bucket name must be globally unique. If name already exists, add random numbers.

Example:

my-lambda-upload-bucket-pranathi-2026

Click Create bucket.

PART 2 — Create DynamoDB Table

DynamoDB tables are created from DynamoDB → Tables → Create table in the console.

Step 3: Open DynamoDB
Search DynamoDB.
Click Tables.
Click Create table.
Step 4: Table details

Use this exact setup:

Table name: S3ObjectMetadata
Partition key: objectKey
Partition key type: String

Keep remaining settings default:

Table settings: Default settings
Capacity mode: On-demand

Click Create table.

Wait until table status becomes:

Active
PART 3 — Create Lambda Function
Step 5: Open Lambda
Search Lambda.
Click Functions.
Click Create function.
Step 6: Function basic details

Choose:

Author from scratch
Function name: S3ToDynamoDBFunction
Runtime: Python 3.12 or Python 3.13
Architecture: x86_64
Step 7: Permissions

Under permissions:

If you are in AWS Academy/Lab:

Use an existing role
Role: LabRole

If not using lab:

Create a new role with basic Lambda permissions

Click Create function.

PART 4 — Add DynamoDB Permission to Lambda Role

Your Lambda must have permission to write into DynamoDB.

If using AWS LabRole

Usually LabRole already has enough permissions. Still verify:

Open Lambda function.
Go to Configuration.
Click Permissions.
Click the role name.
IAM opens.
Check permissions.

You need at least:

dynamodb:PutItem
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents

If your lab role already has admin-like access, fine.

PART 5 — Add S3 Trigger to Lambda
Step 8: Add trigger
Open your Lambda function.
On top diagram, click Add trigger.
Select source:
S3
Step 9: Configure trigger

Fill:

Bucket: your S3 bucket name
Event type: All object create events
Prefix: leave blank
Suffix: leave blank
Recursive invocation warning: tick/check the box if shown

Click Add.

Important: do not make Lambda write files back into the same S3 bucket, or it can trigger itself repeatedly.

PART 6 — Add Lambda Code
Step 10: Open Code tab
Go to Lambda function.
Click Code.
Open file:
lambda_function.py

Delete old code and paste this:

import json
import boto3
import urllib.parse
from datetime import datetime, timezone

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("S3ObjectMetadata")

def lambda_handler(event, context):
    print("Received event:")
    print(json.dumps(event, indent=2))

    if "Records" not in event:
        return {
            "statusCode": 400,
            "body": "No S3 Records found. Use an actual S3 upload event, not a manual empty test."
        }

    for record in event["Records"]:
        bucket_name = record["s3"]["bucket"]["name"]
        object_key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
        object_size = record["s3"]["object"].get("size", 0)
        event_name = record["eventName"]
        event_time = record["eventTime"]

        item = {
            "objectKey": object_key,
            "bucketName": bucket_name,
            "objectSize": object_size,
            "eventName": event_name,
            "eventTime": event_time,
            "processedAt": datetime.now(timezone.utc).isoformat()
        }

        table.put_item(Item=item)

        print(f"Inserted item into DynamoDB: {item}")

    return {
        "statusCode": 200,
        "body": "S3 object metadata inserted into DynamoDB successfully"
    }

Click Deploy.

Do not skip Deploy. Code is not active until deployed.

PART 7 — Upload File to S3
Step 11: Upload object
Go to S3.
Open your bucket.
Click Upload.
Click Add files.
Choose any file, for example:
test.txt
Click Upload.

This upload should automatically trigger Lambda.

PART 8 — Check DynamoDB Output
Step 12: Open table items
Go to DynamoDB.
Click Tables.
Open:
S3ObjectMetadata
Click Explore table items.

You should see one item like:

objectKey: test.txt
bucketName: your-bucket-name
objectSize: file size
eventName: ObjectCreated:Put
eventTime: upload time
processedAt: Lambda processing time
PART 9 — Check CloudWatch Logs

Lambda sends execution logs to CloudWatch; AWS Lambda tutorials use CloudWatch Logs to view Lambda invocation output.

Step 13: Open logs from Lambda
Go to Lambda.
Open your function.
Click Monitor.
Click View CloudWatch logs.
Open latest Log stream.

You should see:

Received event:
...
Inserted item into DynamoDB
PART 10 — If You Get KeyError: 'Records'

This happens when you manually click Test in Lambda using a blank/default test event.

Your Lambda expects an S3 event like:

event["Records"]

But manual test event may not contain Records.

Fix:

Do not test with default event. Instead, upload a real file to S3.

Or use this test event:

{
  "Records": [
    {
      "eventName": "ObjectCreated:Put",
      "eventTime": "2026-05-01T10:00:00.000Z",
      "s3": {
        "bucket": {
          "name": "your-bucket-name"
        },
        "object": {
          "key": "test.txt",
          "size": 123
        }
      }
    }
  ]
}

Replace:

your-bucket-name

with your actual bucket name.

PART 11 — Common Mistakes
Mistake 1: DynamoDB table name mismatch

If your code says:

table = dynamodb.Table("S3ObjectMetadata")

Then your table name must be exactly:

S3ObjectMetadata

Capital letters matter.

Mistake 2: Wrong partition key

Your table partition key must be:

objectKey

Type:

String

If you created another partition key like id, either recreate table or change code.

Mistake 3: Lambda not deployed

After editing code, always click:

Deploy
Mistake 4: S3 trigger not added

Check Lambda page. You should see S3 trigger in the function diagram.

Mistake 5: No permission to write DynamoDB

Error usually looks like:

AccessDeniedException

Meaning Lambda role does not have dynamodb:PutItem.

PART 12 — Final Expected Result

After uploading a file to S3:

S3 bucket contains uploaded file
Lambda runs automatically
CloudWatch shows logs
DynamoDB table contains file metadata

That is the full working flow.

---------------------------------------------------------------------------------------------------------------------------------------------------------

week8
PART A — SNS Topic + Email Notification
Step 1: Open SNS
Login to AWS Console.
In search bar, type SNS.
Open Amazon SNS.
Left side → click Topics.
Click Create topic.
Step 2: Create Topic

Select:

Type: Standard
Name: MyEmailTopic

Leave everything else default.

Click Create topic.

Step 3: Create Email Subscription
Open MyEmailTopic.
Click Subscriptions tab.
Click Create subscription.

Enter:

Protocol: Email
Endpoint: your email address

Click Create subscription.

Step 4: Confirm Email
Open your Gmail/email inbox.
Search for mail from AWS Notifications.
Open it.
Click Confirm subscription.

If you do not confirm, SNS will not send emails. This is the most common mistake.

Step 5: Publish Test Message
Go back to SNS topic.
Click Publish message.
Enter:
Subject: Test Mail
Message body: Hello this is SNS test
Click Publish message.

Now check your email. You should receive the SNS message.

PART B — S3 Upload → SNS → Email
Step 6: Create S3 Bucket
Search S3.
Click Create bucket.

Enter:

Bucket name: my-upload-bucket-2026-yourname
Region: same region as SNS topic

Keep defaults.

Click Create bucket.

Bucket name must be globally unique, so add your name or random numbers.

Step 7: Use/Create SNS Topic

You can reuse:

MyEmailTopic

Or create a new topic:

S3UploadNotification

Make sure email subscription is Confirmed.

Step 8: Add SNS Access Policy for S3
Open SNS topic.
Click Edit.
Go to Access policy.
Choose JSON editor if available.
Replace policy with this.

Change these three values:

SNS_TOPIC_ARN
S3_BUCKET_ARN
ACCOUNT_ID
{
  "Version": "2012-10-17",
  "Id": "s3-publish-to-sns-policy",
  "Statement": [
    {
      "Sid": "AllowS3ToPublishToSNS",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "SNS_TOPIC_ARN",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "S3_BUCKET_ARN"
        },
        "StringEquals": {
          "aws:SourceAccount": "ACCOUNT_ID"
        }
      }
    }
  ]
}

Example:

SNS_TOPIC_ARN = arn:aws:sns:us-east-1:123456789012:S3UploadNotification
S3_BUCKET_ARN = arn:aws:s3:::my-upload-bucket-2026-yourname
ACCOUNT_ID = 123456789012

Click Save changes.

Step 9: Configure S3 Event Notification
Open your S3 bucket.
Go to Properties.
Scroll down to Event notifications.
Click Create event notification.

Enter:

Event name: UploadNotification
Prefix: leave empty
Suffix: leave empty
Event types: All object create events
Destination: SNS topic
SNS topic: MyEmailTopic or S3UploadNotification

Click Save changes.

Step 10: Test S3 → SNS → Email
Open S3 bucket.
Click Upload.
Upload any file, example:
test.txt
Wait a few seconds.
Check email.

You should receive an email containing S3 event details.

PART C — Create SQS Queue and Send Message
Step 11: Open SQS
Search SQS.
Open Amazon Simple Queue Service.
Click Create queue.
Step 12: Create Queue

Enter:

Type: Standard
Name: MyQueue

Keep defaults.

Click Create queue.

Step 13: Send Message
Open MyQueue.
Click Send and receive messages.
In message body, type:
Hello this is SQS test message
Click Send message.
Step 14: Receive Message
Stay on same page.
Click Poll for messages.
You should see your message.
Select message.
Click Delete after reading.

If you do not delete it, the message can appear again later.

PART D — Final Architecture: S3 → SNS → SQS → Lambda

This is the full event-driven flow:

File uploaded to S3
        ↓
S3 sends event to SNS
        ↓
SNS forwards event to SQS
        ↓
Lambda reads message from SQS
        ↓
CloudWatch shows Lambda logs
Step 15: Create New S3 Bucket

Create bucket:

Bucket name: mys3eventbucket123-yourname
Region: same as SNS, SQS, Lambda

Keep defaults and create.

Step 16: Create SNS Topic
Search SNS.
Topics → Create topic.

Enter:

Type: Standard
Name: MyS3SNSTopic

Click Create topic.

Copy the Topic ARN.

Step 17: Create SQS Queue
Search SQS.
Click Create queue.

Enter:

Type: Standard
Name: MyS3Queue

Click Create queue.

Copy the Queue ARN.

Step 18: Subscribe SQS to SNS
Open SNS topic MyS3SNSTopic.
Click Create subscription.

Enter:

Protocol: Amazon SQS
Endpoint: MyS3Queue ARN

Click Create subscription.

For SQS subscriptions, confirmation is usually automatic.

Step 19: Add SQS Access Policy
Open MyS3Queue.
Click Access policy.
Click Edit.
Paste this policy.

Replace:

SQS_ARN
SNS_ARN
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSNSToSendMessageToSQS",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "SQS_ARN",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "SNS_ARN"
        }
      }
    }
  ]
}

Click Save.

Step 20: Add SNS Access Policy for S3
Open SNS topic MyS3SNSTopic.
Click Edit.
Go to Access policy.
Paste this.

Replace:

SNS_TOPIC_ARN
S3_BUCKET_ARN
ACCOUNT_ID
{
  "Version": "2012-10-17",
  "Id": "s3-publish-to-sns-policy",
  "Statement": [
    {
      "Sid": "AllowS3ToPublishToSNS",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "SNS_TOPIC_ARN",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "S3_BUCKET_ARN"
        },
        "StringEquals": {
          "aws:SourceAccount": "ACCOUNT_ID"
        }
      }
    }
  ]
}

Click Save changes.

Step 21: Configure S3 Event Notification
Open bucket mys3eventbucket123-yourname.
Go to Properties.
Scroll to Event notifications.
Click Create event notification.

Enter:

Event name: S3UploadEvent
Event types: All object create events
Destination: SNS topic
SNS topic: MyS3SNSTopic

Click Save changes.

Step 22: Test S3 → SNS → SQS
Open S3 bucket.
Upload test.txt.
Open SQS queue MyS3Queue.
Click Send and receive messages.
Click Poll for messages.

You should see S3 event JSON.

Do not worry if the message looks huge. That is normal.

Step 23: Create Lambda Function
Search Lambda.
Click Create function.

Select:

Author from scratch
Function name: SQSConsumerFunction
Runtime: Python 3.12 or Python 3.13

Permissions:

Use existing role: LabRole

If LabRole is not available, choose:

Create a new role with basic Lambda permissions

Click Create function.

Step 24: Add SQS Trigger to Lambda
Open SQSConsumerFunction.
Click Add trigger.
Select source:
SQS
Select queue:
MyS3Queue
Batch size:
10
Click Add.
Step 25: Add Lambda Code

Open Code tab.

Replace existing code with:

import json

def lambda_handler(event, context):
    print("Lambda triggered by SQS")
    print(json.dumps(event, indent=2))

    for record in event["Records"]:
        print("Message received from SQS:")
        print(record["body"])

    return {
        "statusCode": 200,
        "body": "Message processed successfully"
    }

Click Deploy.

Step 26: Final Test
Go to S3 bucket.
Upload a new file:
final-test.txt
Wait 10–20 seconds.
Go to Lambda.
Open SQSConsumerFunction.
Click Monitor.
Click View CloudWatch logs.
Open latest log stream.

You should see:

Lambda triggered by SQS
Message received from SQS

That means your architecture works.

---------------------------------------------------------------------------------------------------------------------------------------------------------
week9
WEEK-9: Elastic Load Balancer + Auto Scaling using EC2
Final Architecture
User Browser    ↓Application Load Balancer    ↓Target Group    ↓EC2 Web Server 1 + EC2 Web Server 2    ↓Auto Scaling Group maintains/replaces instances

PART 1 — Launch Two EC2 Web Servers
Step 1: Open EC2


Login to AWS Console.


Search EC2.


Open EC2 Dashboard.


Click Instances.


Click Launch instances.



Create EC2 Instance 1
Step 2: Name Instance
Enter:
Name: webserver-1

Step 3: Choose AMI
Select:
Amazon Linux 2
or
Amazon Linux 2023
For your lab notes, Amazon Linux 2 is fine.

Step 4: Choose Instance Type
Select:
t2.micro
or free-tier available instance.

Step 5: Select Key Pair
Choose existing key pair or create new one.
Example:
week9-key.pem
Download it safely. You need this for SSH.

Step 6: Network Settings
Click Edit under Network settings.
Use:
VPC: Default VPCSubnet: Any public subnetAuto-assign public IP: Enable
Security group:
Create security groupSecurity group name: webserver-sg
Inbound rules:
SSH   TCP 22   My IPHTTP  TCP 80   Anywhere 0.0.0.0/0

Step 7: Launch Instance
Click:
Launch instance
Wait until instance state becomes:
Running
Status check:
2/2 checks passed

PART 2 — Install Apache on Webserver 1
Step 8: Connect to Instance


Select webserver-1.


Click Connect.


Go to SSH client.


Copy SSH command.


From PowerShell:
ssh -i "week9-key.pem" ec2-user@<WEB_SERVER_1_PUBLIC_IP>
If using Ubuntu AMI, username is:
ubuntu
For Amazon Linux:
ec2-user

Step 9: Install Web Server
Run these commands one by one:
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
echo "This is Server 1" | sudo tee /var/www/html/index.html

Step 10: Test Webserver 1
Open browser:
http://<WEB_SERVER_1_PUBLIC_IP>
Expected output:
This is Server 1
If it does not open, check:
Security group allows HTTP 80 from 0.0.0.0/0Apache is runningInstance has public IP

PART 3 — Create EC2 Instance 2
Repeat the same EC2 launch process.
Step 11: Launch Second Instance
Use:
Name: webserver-2AMI: Amazon Linux 2Instance type: t2.microKey pair: same key pairSecurity group: webserver-sgPublic IP: Enable
Launch instance.

Step 12: Connect to Webserver 2
ssh -i "week9-key.pem" ec2-user@<WEB_SERVER_2_PUBLIC_IP>

Step 13: Install Apache on Webserver 2
Run:
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
echo "This is Server 2" | sudo tee /var/www/html/index.html

Step 14: Test Webserver 2
Open browser:
http://<WEB_SERVER_2_PUBLIC_IP>
Expected output:
This is Server 2

PART 4 — Create Load Balancer Security Group
Step 15: Open Security Groups


Go to EC2.


Left side → Security Groups.


Click Create security group.


Enter:
Security group name: lb-sgDescription: Security group for application load balancerVPC: Default VPC
Inbound rule:
Type: HTTPPort: 80Source: Anywhere IPv4 0.0.0.0/0
Optional HTTPS:
Type: HTTPSPort: 443Source: Anywhere IPv4 0.0.0.0/0
Click Create security group.

PART 5 — Create Target Group
Step 16: Open Target Groups


Go to EC2.


Left side → Target Groups.


Click Create target group.



Step 17: Target Group Settings
Choose:
Target type: InstancesTarget group name: web-servers-tgProtocol: HTTPPort: 80VPC: Default VPCProtocol version: HTTP1
Health check:
Health check protocol: HTTPHealth check path: /
Click Next.

Step 18: Register Targets
Select:
webserver-1webserver-2
Click:
Include as pending below
Then click:
Create target group

PART 6 — Create Application Load Balancer
Step 19: Open Load Balancers


Go to EC2.


Left side → Load Balancers.


Click Create load balancer.


Under Application Load Balancer, click Create.



Step 20: Basic Configuration
Enter:
Load balancer name: my-albScheme: Internet-facingIP address type: IPv4
Listener:
Protocol: HTTPPort: 80

Step 21: Network Mapping
Select:
VPC: Default VPC
Select at least two Availability Zones.
Example:
us-east-1aus-east-1b
Select public subnets for both.
Important: ALB needs at least two subnets in different Availability Zones.

Step 22: Security Group
Choose:
lb-sg
Remove default security group if not needed.

Step 23: Listener and Routing
For default action, select:
Forward to target groupTarget group: web-servers-tg

Step 24: Review and Create
Click:
Create load balancer
Wait a few minutes until ALB state becomes:
Active

PART 7 — Verify Load Balancer
Step 25: Get ALB DNS Name


Go to Load Balancers.


Select my-alb.


Copy DNS name.


Example:
my-alb-123456789.us-east-1.elb.amazonaws.com
Open in browser:
http://my-alb-123456789.us-east-1.elb.amazonaws.com
Expected output:
This is Server 1
or
This is Server 2
Refresh multiple times. You may see traffic distributed between servers.

PART 8 — Check Target Health
Step 26: Open Target Group


Go to EC2.


Click Target Groups.


Open web-servers-tg.


Click Targets tab.


Expected status:
webserver-1: healthywebserver-2: healthy

If Target Is Unhealthy
Check these:
Check 1: Apache running
SSH into instance:
sudo systemctl status httpd
If stopped:
sudo systemctl start httpd

Check 2: HTTP security group
Your EC2 security group must allow:
HTTP 80 from lb-sg
or for lab simplicity:
HTTP 80 from 0.0.0.0/0
Better practice:
HTTP 80 from Load Balancer security group

Check 3: Health check path
Target group health check path should be:
/
Your page must exist:
cat /var/www/html/index.html

PART 9 — Create AMI from Existing EC2
Now we prepare Auto Scaling.
Step 27: Create AMI


Go to EC2 → Instances.


Select webserver-1.


Click Actions.


Click Image and templates.


Click Create image.


Enter:
Image name: my-webserver-AMIDescription: Apache web server AMI for Auto Scaling
Keep default settings.
Click:
Create image

Step 28: Wait for AMI


Go to EC2 → AMIs.


Select Owned by me.


Wait until status becomes:


Available
Do not continue until AMI is available.

PART 10 — Create Launch Template
Step 29: Open Launch Templates


Go to EC2.


Left side → Launch Templates.


Click Create launch template.



Step 30: Configure Launch Template
Enter:
Launch template name: my-launch-templateDescription: Launch template for webserver ASG

Step 31: Select AMI
Under AMI:
My AMIsSelect: my-webserver-AMI

Step 32: Instance Type
Choose:
t2.micro

Step 33: Key Pair
Select:
week9-key.pem

Step 34: Security Group
Choose the EC2 webserver security group:
webserver-sg
Important: Do not choose only lb-sg for EC2 instances.
Correct setup:
ALB security group: lb-sgEC2 security group: webserver-sg
The EC2 security group should allow HTTP 80 from the ALB.

Step 35: User Data
Since AMI already has Apache installed, user data is optional.
You can leave it blank.
Click:
Create launch template

PART 11 — Create Auto Scaling Group
Step 36: Open Auto Scaling Groups


Go to EC2.


Left side → Auto Scaling Groups.


Click Create Auto Scaling group.



Step 37: Choose Launch Template
Enter:
Auto Scaling group name: webserver-asgLaunch template: my-launch-templateVersion: Default / Version 1
Click Next.

Step 38: Network Settings
Choose:
VPC: Default VPCAvailability Zones and subnets: select at least two public subnets
Click Next.

Step 39: Attach Load Balancer
Choose:
Attach to an existing load balancer
Select:
Choose from your load balancer target groupsTarget group: web-servers-tg
Health checks:
Turn on Elastic Load Balancing health checksHealth check grace period: 300 seconds
Click Next.

Step 40: Group Size
Set:
Desired capacity: 2Minimum capacity: 1Maximum capacity: 4
Meaning:
Desired = normally maintain 2 instancesMinimum = never go below 1Maximum = never go above 4
Click Next.

Step 41: Scaling Policy
For simple lab, choose:
No scaling policies
or choose target tracking:
Target tracking scaling policyMetric type: Average CPU utilizationTarget value: 70
For viva answer, CPU 70% scaling is a good example.
Click Next.

Step 42: Notifications
Skip notifications.
Click Next.

Step 43: Tags
Add tag:
Key: NameValue: ASG-WebServer
Tick:
Tag new instances
Click Next.

Step 44: Review and Create
Check:
Launch template: my-launch-templateTarget group: web-servers-tgDesired: 2Min: 1Max: 4Health check: ELB enabled
Click:
Create Auto Scaling group

PART 12 — Verify Auto Scaling
Step 45: Check ASG Instances


Open webserver-asg.


Click Instance management.


You should see instances launched by Auto Scaling.


Wait until lifecycle state becomes:
InService

Step 46: Check Target Group Again


Go to Target Groups.


Open web-servers-tg.


Click Targets.


Expected:
2 healthy instances

Step 47: Test Load Balancer Again
Open ALB DNS:
http://<ALB-DNS-NAME>
Expected:
This is Server 1
or Apache page from AMI.
If all ASG instances came from webserver-1 AMI, they may all show:
This is Server 1
That is normal because AMI copied Server 1’s content.

PART 13 — Test Auto Recovery
Step 48: Terminate One ASG Instance


Go to EC2 → Instances.


Find one instance created by ASG.


Select it.


Click Instance state.


Click Terminate instance.


Wait 1–3 minutes.
Auto Scaling should automatically launch a replacement instance.

Step 49: Confirm Replacement
Go back to:
EC2 → Auto Scaling Groups → webserver-asg → Instance management
You should again see desired capacity maintained.
Example:
Desired capacity = 2Running instances = 2
That proves Auto Scaling is working.

PART 14 — Common Error: 503 Service Temporarily Unavailable
If ALB URL shows:
503 Service Temporarily Unavailable
It means:
Load balancer has no healthy targets
Fix checklist:


Go to Target Groups.


Open web-servers-tg.


Check Targets.


If targets are unhealthy, check Apache:


sudo systemctl status httpd


Check EC2 security group allows HTTP 80.


Check target group port is 80.


Check health check path is /.


Open instance public IP directly.


If instance public IP also fails, Apache/security group is wrong.


If instance public IP works but ALB fails, target group/security group routing is wrong.



PART 15 — Final Cleanup to Avoid Billing
After lab, delete in this order:
Step 50: Delete Auto Scaling Group


EC2 → Auto Scaling Groups.


Select webserver-asg.


Click Delete.


Confirm.



Step 51: Terminate EC2 Instances


EC2 → Instances.


Select webserver instances.


Instance state → Terminate.



Step 52: Delete Load Balancer


EC2 → Load Balancers.


Select my-alb.


Actions → Delete load balancer.



Step 53: Delete Target Group


EC2 → Target Groups.


Select web-servers-tg.


Actions → Delete.



Step 54: Delete Launch Template


EC2 → Launch Templates.


Select my-launch-template.


Actions → Delete template.



Step 55: Deregister AMI


EC2 → AMIs.


Select my-webserver-AMI.


Actions → Deregister AMI.


Also delete snapshot:


EC2 → Snapshots.


Find snapshot created by AMI.


Delete it.


This matters because snapshots can cost money.

------------------------------------------------------------------------------------------------------------------------------------------------------------

week10
WEEK-10 — Elastic Beanstalk Step-by-Step Guide

Elastic Beanstalk deploys your app and automatically provisions EC2, load balancing, health monitoring, and scaling resources for the environment.

PART 0 — Create a Simple App First

Since you may not already have a .war file, use a simple Node.js zip app.

Step 1: Create project folder

On your laptop:

mkdir my-eb-app
cd my-eb-app
Step 2: Create package.json

Create a file named:

package.json

Paste:

{
  "name": "my-eb-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
Step 3: Create app.js

Create a file named:

app.js

Paste:

const express = require("express");
const app = express();

const PORT = process.env.PORT || 8080;

app.get("/", (req, res) => {
  res.send("Hello from AWS Elastic Beanstalk!");
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
Step 4: Zip the files correctly

Important: zip the files inside the folder, not the folder itself.

Your zip should contain:

app.js
package.json

Not:

my-eb-app/app.js
my-eb-app/package.json

Create zip:

my-eb-app.zip
PART 1 — Open Elastic Beanstalk
Login to AWS Console.
Search Elastic Beanstalk.
Open Elastic Beanstalk.
Make sure your region is correct, for example:
us-east-1
PART 2 — Create Application
Click Create application.
Enter:
Application name: MyApp
Description: Elastic Beanstalk demo application
Choose:
Environment tier: Web server environment

Elastic Beanstalk has a Create Environment wizard where you choose the environment tier, such as Web server environment.

PART 3 — Configure Environment

Use:

Environment name: MyApp-env
Domain: leave default or enter myapp-demo
Platform: Node.js
Platform branch: latest recommended Node.js branch
Platform version: recommended

If you do not see Platform and Branch, look for a section called:

Platform

or

Managed platform

Then select:

Node.js
PART 4 — Upload Application Code

Under Application code, choose:

Upload your code

Then:

Version label: v1
Source code origin: Local file
File: my-eb-app.zip

Upload the zip file.

PART 5 — Configure Service Access

This part is important.

Elastic Beanstalk uses a service role to call AWS services like EC2, Elastic Load Balancing, and Auto Scaling on your behalf.

Use:

Service role: LabRole

For EC2 instance profile:

EC2 instance profile: LabInstanceProfile

or:

aws-elasticbeanstalk-ec2-role

The instance profile is applied to EC2 instances so they can retrieve application versions from S3 and write logs to S3.

If you are in AWS Academy, usually choose:

LabRole
LabInstanceProfile
PART 6 — Configure Network

Select:

VPC: Default VPC
Public IP address: Enabled

For subnets:

Instance subnets: select public subnets
Load balancer subnets: select public subnets

For a simple lab, select at least two public subnets if load balancer is enabled.

PART 7 — Configure Instance and Scaling

Choose:

Instance type: t2.micro or t3.micro
Min instances: 1
Max instances: 2

Capacity:

Environment type: Load balanced

or for cheaper/simple lab:

Single instance

Use Single instance if your lab account has limits or you want less billing risk.

Use Load balanced if your lab specifically requires load balancer and auto scaling.

PART 8 — Security Group

If Elastic Beanstalk creates one automatically, keep it.

Required inbound rule:

HTTP 80 from 0.0.0.0/0

Optional:

SSH 22 from My IP

SSH is only needed if you want to connect to EC2 manually.

PART 9 — Review and Create
Click Review.
Check:
Application: MyApp
Environment: MyApp-env
Platform: Node.js
Code: my-eb-app.zip
Service role: LabRole
Instance profile: LabInstanceProfile
Instance type: t2.micro/t3.micro
Min: 1
Max: 2
Click Create environment.

Wait until status becomes:

Health: Ok / Green

This can take several minutes.

PART 10 — Access Your Application
Open Elastic Beanstalk.
Click your environment:
MyApp-env
Copy the environment URL.

Example:

http://myapp-env.eba-xxxxx.us-east-1.elasticbeanstalk.com
Open it in browser.

Expected output:

Hello from AWS Elastic Beanstalk!
PART 11 — Check Resources Created Automatically

Elastic Beanstalk may create:

EC2 instance
Security group
Auto Scaling Group
Load Balancer
Target Group
CloudWatch logs/metrics
S3 application version storage

To check:

Go to EC2 → Instances.
Look for instance created by Elastic Beanstalk.
Go to EC2 → Load Balancers if you selected load balanced.
Go to EC2 → Auto Scaling Groups.
PART 12 — Update Application
Step 1: Edit app.js

Change:

res.send("Hello from AWS Elastic Beanstalk!");

to:

res.send("Updated version deployed successfully!");
Step 2: Create new zip

Zip again:

app.js
package.json

Name:

my-eb-app-v2.zip
Step 3: Upload and Deploy

AWS docs describe this as opening the environment, choosing Upload and deploy, uploading the source bundle, and choosing Deploy.

In console:

Go to Elastic Beanstalk.
Open environment.
Click Upload and deploy.
Upload:
my-eb-app-v2.zip
Version label:
v2
Click Deploy.

Refresh your app URL.

Expected:

Updated version deployed successfully!
Common Errors and Fixes
Error 1: Application shows 502 Bad Gateway

Possible causes:

Wrong port
App not starting
Missing package.json start script
Zip folder structure wrong

Fix:

Use:

const PORT = process.env.PORT || 8080;

Make sure package.json has:

"start": "node app.js"
Error 2: Deployment failed

Check:

Elastic Beanstalk → Environment → Events

Then:

Logs → Request logs → Last 100 lines
Error 3: No platform/branch visible

You may be on the new wizard page.

Look for:

Platform
Managed platform
Application code
Presets
Configure more options

Select:

Node.js

For a .war file, choose:

Tomcat

For Python:

Python
Error 4: EC2 instance profile missing

If you do not see LabInstanceProfile, choose another available EC2 instance profile.

In non-lab accounts, AWS commonly uses:

aws-elasticbeanstalk-ec2-role

AWS documents that Elastic Beanstalk EC2 roles commonly include policies such as AWSElasticBeanstalkWebTier.


---------------------------------------------------------------------------------------------------------------------------------------------------

week11
WEEK-11 — Amazon Lex Hotel Booking Bot

Amazon Lex V2 bots use intents, sample utterances, and slots. Slots collect required user inputs before the bot completes the intent. Amazon Lex also supports conditional branching based on slot values, such as {age} < 18.

Final Bot Flow
User: Book a hotel
Bot: What is your age?
User: 25
Bot: Which city do you want?
User: Hyderabad
Bot: What is your check-in date?
User: Tomorrow
Bot: How many nights will you stay?
User: 2
Bot: Select room type
User: Double
Bot: Do you want to confirm booking in Hyderabad for 2 nights?
User: Yes
Bot: Booking confirmed.
PART 1 — Open Amazon Lex
Login to AWS Console.
In the search bar, type:
Amazon Lex
Open Amazon Lex.
Make sure you are using Amazon Lex V2.
Click Create bot.
PART 2 — Create Bot
Step 1: Creation Method

Choose:

Create a blank bot

or:

Create

depending on your console screen.

Step 2: Bot Configuration

Enter:

Bot name: HotelBookingBot
Description: Bot for hotel room booking
Step 3: IAM Permissions

Choose:

Create a role with basic Amazon Lex permissions

or:

Create new role

If your lab asks for LabRole and it appears, choose it. Otherwise, create a new Lex role.

Step 4: COPPA

For:

Is this bot subject to COPPA?

Choose:

No
Step 5: Idle Session Timeout

Keep default, usually:

5 minutes

Click Next.

PART 3 — Add Language
Step 6: Language Settings

Choose:

Language: English (US)
Voice interaction: None

Voice is optional.

For text-only bot, choose:

No voice

or keep voice default if required.

Click:

Done

Now the bot is created.

PART 4 — Create Intent
Step 7: Open Intents
Inside your bot, go to English (US).
Left side → click Intents.
Click Add intent.
Choose:
Add empty intent
Step 8: Name Intent

Enter:

Intent name: BookHotel

Click:

Add
PART 5 — Add Sample Utterances

Sample utterances are example sentences users type to start this intent.

In Sample utterances, add these one by one:

I want to book a hotel
Book a room
Reserve hotel
I need a room
Book a hotel
I want a hotel room
Can you reserve a room for me
Hotel booking

Click Save intent after adding them.

PART 6 — Create Age Slot

Your notes have one small mistake: for age slot, the prompt should be:

What is your age?

not:

Which city do you want?
Step 9: Add Age Slot
In the BookHotel intent, scroll to Slots.
Click Add slot.

Enter:

Name: age
Slot type: AMAZON.Number
Prompt: What is your age?

Turn on:

Required

Click:

Add
PART 7 — Add Conditional Logic for Age

Goal:

If age < 18, bot says user is not eligible.
If age >= 18, bot continues booking.
Step 10: Open Advanced Options
In the age slot, click the slot name.
Find Advanced options.
Open Slot capture.
Find Slot capture success response.
Enable or open Conditional branching.

Amazon Lex V2 supports conditional branching to control conversation paths based on slot values.

Step 11: Add Condition

Click:

Add condition

Condition:

{age} < 18

Response:

You are not eligible for hotel booking.

Next step/action:

Close intent

or:

End conversation

Default branch:

Continue to next step

Click Save.

Tiny correction: write “You are not eligible for hotel booking”, not “Your not eligible for hoot booking”.

PART 8 — Create Location Slot
Step 12: Add Location Slot

Go back to Slots.

Click Add slot.

Enter:

Name: location
Slot type: AMAZON.City
Prompt: Which city do you want?

Enable:

Required

Click Add.

PART 9 — Create Check-in Date Slot
Step 13: Add Check-in Slot

Click Add slot.

Enter:

Name: checkin
Slot type: AMAZON.Date
Prompt: What is your check-in date?

Enable:

Required

Optional retry prompt:

Please provide a valid date, for example 2026-03-25.

Click Add.

PART 10 — Create Number of Nights Slot
Step 14: Add Nights Slot

Click Add slot.

Enter:

Name: nights
Slot type: AMAZON.Number
Prompt: How many nights will you stay?

Enable:

Required

Click Add.

PART 11 — Create Custom Slot Type for Room Type
Step 15: Go to Slot Types
Left side → click Slot types.
Click Add slot type.
Choose:
Add blank slot type
Step 16: Create RoomType Slot Type

Enter:

Slot type name: RoomType

Add values:

Single
Double
Suite

Click:

Save slot type
PART 12 — Add RoomType Slot to Intent
Step 17: Go Back to Intent
Left side → click Intents.
Open:
BookHotel
Scroll to Slots.
Click Add slot.

Enter:

Name: roomType
Slot type: RoomType
Prompt: Which room type do you want?

Enable:

Required

Click Add.

PART 13 — Add Response Card Buttons

Response cards show quick buttons like Single, Double, Suite.

Step 18: Open Room Type Slot Prompt
Click the roomType slot.
Go to Prompt section.
Click More prompt options.
Click Add.
Choose Card group or Response card.
Step 19: Add Card Details

Enter:

Card title: Select Room Type
Subtitle: Choose one room type

Add buttons:

Button text: Single
Button value: Single

Button text: Double
Button value: Double

Button text: Suite
Button value: Suite

Save the slot.

PART 14 — Arrange Slot Order

Your slot order should be:

1. age
2. location
3. checkin
4. nights
5. roomType

If Lex has slot priority/order settings, arrange them exactly like this.

This matters because Lex asks slot questions in this order.

PART 15 — Configure Initial Response

Initial response means what bot says after identifying the intent.

Step 20: Add Initial Response

Inside BookHotel, find:

Initial response

Add:

Welcome to Hotel Booking! I will help you reserve a room.

Do not ask name unless you create a name slot. Your sample flow does not collect name, so keep it simple.

PART 16 — Configure Confirmation
Step 21: Add Confirmation Prompt

Find:

Confirmation

Enable confirmation.

Confirmation prompt:

Do you want to confirm booking in {location} for {nights} nights in a {roomType} room?

Decline response:

Okay, your booking has not been confirmed.
PART 17 — Configure Closing Response
Step 22: Add Closing Response

Find:

Closing response

Add:

Booking confirmed.

Better version:

Booking confirmed for {location} from {checkin} for {nights} nights in a {roomType} room.

Click:

Save intent
PART 18 — Build Bot
Step 23: Build
Top-right, click:
Build
Wait until build finishes.

Expected status:

Build succeeded

If build fails, check:

all required slots have prompts
custom slot type is saved
intent is saved
no invalid condition syntax
PART 19 — Test Bot
Step 24: Open Test Panel
Click Test.
Test panel opens on the right side.

Type:

Book a hotel

Expected conversation:

Bot: What is your age?
You: 25
Bot: Which city do you want?
You: Hyderabad
Bot: What is your check-in date?
You: Tomorrow
Bot: How many nights will you stay?
You: 2
Bot: Which room type do you want?
You: Double
Bot: Do you want to confirm booking in Hyderabad for 2 nights in a Double room?
You: Yes
Bot: Booking confirmed.
PART 20 — Test Underage Condition

Type:

Book a hotel

Then answer:

17

Expected:

You are not eligible for hotel booking.

If it still continues, your conditional branch is not connected to Close intent / End conversation.

--------------------------------------------------------------------------------------------------------------------------------------------------------

week12
WEEK-12 — IAM User + Console Access + CLI Access
Goal

You will create an IAM user with S3-only access, test that S3 works, test that EC2 fails, then configure AWS CLI and verify the same permission behavior from your terminal.

PART A — Create IAM User from Root/Admin Account
Step 1: Login as Root/Admin
Open AWS Console.
Login using your root email or admin account.
Search:
IAM
Open IAM.
Step 2: Create IAM User
Left side menu → click Users.
Click Create user.
Enter username:
S3_Specialist
Tick:
Provide user access to the AWS Management Console
Choose:
I want to create an IAM user
Console password:
Custom password

Example:

User123$
Untick this if shown:
Users must create a new password at next sign-in

Only untick for lab convenience. In real security practice, keep it enabled.

Click Next.

Step 3: Attach S3 Permission

Choose:

Attach policies directly

Search:

AmazonS3FullAccess

Tick:

AmazonS3FullAccess

Click Next.

Step 4: Review and Create

Check:

Username: S3_Specialist
Access type: Console access
Policy: AmazonS3FullAccess

Click:

Create user
Step 5: Download Login Details

On the final page:

Click Download .csv file.
Save it safely.
It contains:
Sign-in URL
Username
Password

Do not share this file publicly.

PART B — Login as IAM User
Step 6: Copy Account ID

From top-right account dropdown, copy your:

12-digit AWS Account ID

Example:

123456789012

Now click:

Sign out
Step 7: Login as IAM User

You have two options.

Option 1: Use Sign-in URL from CSV

Open the URL from the downloaded .csv.

Option 2: Use AWS IAM Login Page
Open AWS sign-in page.
Choose:
IAM user
Enter:
Account ID: your 12-digit account ID
IAM username: S3_Specialist
Password: User123$

Click Sign in.

PART C — Permission Testing in Console
Test 1: EC2 Access Should Fail
Search:
EC2
Open EC2.
Try to view instances.

Expected result:

Access Denied

or:

API Error

This is correct because the user only has S3 permission.

Test 2: S3 Access Should Work
Search:
S3
Open S3.
Click Create bucket.

Enter:

Bucket name: s3-specialist-bucket-yourname-2026
Region: your current region

Keep defaults.

Click:

Create bucket

Expected result:

Bucket created successfully

This proves IAM policy is working correctly.

PART D — Create CLI Access Keys

Now you will access AWS from CMD/Terminal.

Step 8: Go Back to Admin/Root

Sign out from IAM user.

Login again as:

Root/Admin

Open:

IAM → Users → S3_Specialist
Step 9: Create Access Key
Click your IAM user:
S3_Specialist
Open tab:
Security credentials
Scroll to:
Access keys
Click:
Create access key
Select use case:
Command Line Interface (CLI)
Tick acknowledgement checkbox.
Click Next.
Description tag optional:
cli-access-key
Click Create access key.
Click:
Download .csv file

This CSV contains:

Access Key ID
Secret Access Key

Critical: Secret access key is shown only once.

PART E — Install AWS CLI
Step 10: Download AWS CLI

For Windows:

https://awscli.amazonaws.com/AWSCLIV2.msi
Download the MSI installer.
Double-click it.
Click Next.
Accept license.
Click Install.
Finish setup.
Step 11: Verify Installation

Open Command Prompt or PowerShell.

Run:

aws --version

Expected output:

aws-cli/2.x.x

If command not recognized, close terminal and reopen it.

PART F — Configure AWS CLI
Step 12: Run Configure Command

In CMD/PowerShell:

aws configure

It asks four things.

Enter from CSV:

AWS Access Key ID: paste Access Key ID
AWS Secret Access Key: paste Secret Access Key
Default region name: ap-south-1
Default output format: json

If your lab uses another region, use that region.

Example:

us-east-1
PART G — Verify CLI Permissions
Step 13: Test EC2 Access

Run:

aws ec2 describe-instances

Expected result:

AccessDenied

This is correct.

Why? Because the IAM user has only:

AmazonS3FullAccess

No EC2 access.

Step 14: Test S3 Access

Run:

aws s3 ls

Expected result:

List of S3 buckets

If no buckets exist, output may be blank. Blank is okay if there is no error.

Step 15: Create S3 Bucket from CLI

For ap-south-1, use:

aws s3 mb s3://s3-specialist-cli-bucket-yourname-2026 --region ap-south-1

For us-east-1, use:

aws s3 mb s3://s3-specialist-cli-bucket-yourname-2026 --region us-east-1

Bucket name must be globally unique.

Example:

aws s3 mb s3://s3-specialist-cli-bucket-pranathi-2026 --region ap-south-1

Expected:

make_bucket: s3-specialist-cli-bucket-pranathi-2026
Step 16: Check Bucket in Console
Open AWS Console.
Go to S3.
Refresh bucket list.

You should see the bucket created from CLI.

PART H — Useful IAM CLI Commands
Create IAM User
aws iam create-user --user-name TestUser
Attach S3 Read-Only Policy
aws iam attach-user-policy --user-name TestUser --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
Create Access Key for User
aws iam create-access-key --user-name TestUser
List IAM Users
aws iam list-users
List S3 Buckets
aws s3 ls
PART I — Safe Cleanup

Do this after lab to avoid security risk.

Step 17: Delete CLI Access Key
IAM → Users.
Open S3_Specialist.
Security credentials.
Access keys.
Deactivate key.
Delete key.
Step 18: Delete S3 Buckets

Before deleting bucket, empty it.

Console method:

S3 → bucket → Empty → Delete

CLI method:

aws s3 rm s3://your-bucket-name --recursive
aws s3 rb s3://your-bucket-name
Step 19: Delete IAM User

Before deleting user:

Remove access keys.
Remove attached policies.
Remove from groups if any.
Delete user.

Console path:

IAM → Users → S3_Specialist → Delete
Scenario-Based Questions
1. Root user for daily work — what is wrong?

Using root for daily work is unsafe because root has full unrestricted access to all AWS services and billing. Use root only for initial setup and critical account tasks. Enable MFA and use IAM users or roles for daily work.

2. Sudden bill increase, root compromise suspected

Do this immediately:

Change root password
Enable MFA
Delete/rotate access keys
Check EC2 for unknown instances
Check IAM for unknown users
Review CloudTrail logs
Stop/delete unknown resources
Contact AWS Support
3. Forgot root password

Use:

Forgot password

on AWS sign-in page. If root email access is lost, contact AWS Support for account recovery.

4. Why avoid root access keys?

Root access keys give full permanent programmatic access. If leaked, the whole AWS account can be controlled. Delete root access keys if they exist.

5. New developer joining

Create a dedicated IAM user or IAM Identity Center user, assign them to a group, attach required permissions, and enable MFA. Never share credentials.

6. User needs only S3 read access

Attach:

AmazonS3ReadOnlyAccess

Do not give full access.

7. Contractor needs access for 2 weeks

Best approach:

Temporary access using IAM role and AWS STS

Less ideal:

Create IAM user and delete it after 2 weeks
8. Access Denied while accessing EC2

Possible causes:

No EC2 permissions
Explicit deny policy
Wrong role/profile
Region mismatch
Resource-level restriction
9. 50 users need same permissions

Use:

IAM Group

Attach policy to group, then add users to the group.

10. Configure CLI

Run:

aws configure

Enter:

Access Key ID
Secret Access Key
Default region
Output format
11. Create IAM user using CLI
aws iam create-user --user-name username
12. Give S3 read access using CLI
aws iam attach-user-policy --user-name username --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
13. Generate access keys
aws iam create-access-key --user-name username

Store the secret key safely. It is shown only once.

14. List IAM users
aws iam list-users
15. Safely delete IAM user

Remove these first:

Access keys
Attached policies
Group memberships
Login profile
MFA devices if any

Then delete user.

------------------------------------------------------------------------------------------------------------------------------------------------------------
week13
WEEK-13 — IAM Roles for EC2
Goal

You will create an IAM role with S3 access, attach it to an EC2 instance, then verify from inside EC2 that:

aws s3 ls works
aws ec2 describe-instances fails

This proves the role gives only the permissions attached to it.

PART 1 — Create IAM Role for EC2
Step 1: Login to AWS Console
Open AWS Console.
Login as root/admin user.
Search:
IAM
Open IAM.
Step 2: Go to Roles
Left side menu → click Roles.
Click Create role.
Step 3: Select Trusted Entity

Choose:

Trusted entity type: AWS service

For service/use case, select:

Service or use case: EC2

This means EC2 is allowed to use this role.

Click Next.

Step 4: Add Permissions

Search:

AmazonS3FullAccess

Tick:

AmazonS3FullAccess

Click Next.

Step 5: Name the Role

Enter:

Role name: EC2-S3-Role

Description optional:

Allows EC2 instance to access S3

Click:

Create role

Role is now ready.

PART 2 — Launch EC2 Instance
Step 6: Open EC2
Search:
EC2
Open EC2 Dashboard.
Click Instances.
Click Launch instances.
Step 7: Instance Name

Enter:

Name: Role-Test-EC2
Step 8: Choose AMI

Choose:

Amazon Linux 2023

or:

Amazon Linux 2

Amazon Linux is best here because AWS CLI is usually available or easy to install.

Step 9: Instance Type

Choose:

t2.micro

or free-tier instance type.

Step 10: Key Pair

Choose existing key pair or create new.

Example:

week13-key.pem

Download it safely if creating new.

Step 11: Network Settings

Use:

VPC: Default VPC
Subnet: Public subnet
Auto-assign public IP: Enable

Security group:

Create security group
Security group name: role-test-sg

Inbound rules:

SSH 22 from My IP

No need to allow HTTP for this lab.

Click:

Launch instance
PART 3 — Attach IAM Role to EC2

You can attach the role during launch or after launch. Your notes use after launch.

Step 12: Select Running Instance
Go to EC2 → Instances.
Select:
Role-Test-EC2
Click:
Actions
Go to:
Security
Click:
Modify IAM role
Step 13: Select Role

Choose:

EC2-S3-Role

Click:

Update IAM role

Now EC2 has temporary credentials through the IAM role.

PART 4 — Connect to EC2
Step 14: Connect Using EC2 Instance Connect

Simplest method:

Select EC2 instance.
Click Connect.
Choose EC2 Instance Connect.
Click Connect.

If this works, continue inside the browser terminal.

Step 15: Connect Using SSH from PowerShell

If using .pem file:

ssh -i "week13-key.pem" ec2-user@<EC2_PUBLIC_IP>

If permission error occurs on Windows, run:

icacls week13-key.pem /inheritance:r
icacls week13-key.pem /grant:r "$env:USERNAME:R"

Then try SSH again.

PART 5 — Check AWS CLI Inside EC2
Step 16: Check CLI Version

Inside EC2, run:

aws --version

If it shows version, good.

If command not found, install AWS CLI.

For Amazon Linux:

sudo yum update -y
sudo yum install awscli -y

For Amazon Linux 2023:

sudo dnf install awscli -y
PART 6 — Verify S3 Access Works
Step 17: Run S3 Command

Inside EC2:

aws s3 ls

Expected result:

List of S3 buckets

If no buckets exist, output may be blank. Blank is okay.

The important thing is: no AccessDenied error.

PART 7 — Verify EC2 Access Fails
Step 18: Run EC2 Command

Inside EC2:

aws ec2 describe-instances

Expected result:

UnauthorizedOperation

or:

AccessDenied

This is correct because the role has only:

AmazonS3FullAccess

It does not have EC2 permissions.

PART 8 — Important Concept

You did not run:

aws configure

You did not paste:

Access Key ID
Secret Access Key

Why?

Because IAM role automatically provides temporary security credentials to the EC2 instance.

That is the whole point of IAM roles.

PART 9 — Troubleshooting
Problem 1: aws s3 ls gives AccessDenied

Check:

IAM role attached?
Correct role selected?
Role has AmazonS3FullAccess?
Wait 1–2 minutes after attaching role

Then retry:

aws s3 ls
Problem 2: Unable to locate credentials

This means EC2 does not have usable role credentials.

Fix:

EC2 → Instances.
Select instance.
Actions → Security → Modify IAM role.
Attach EC2-S3-Role.
Wait one minute.
Retry.
Problem 3: SSH not connecting

Check:

Instance is running
Public IP exists
Security group allows SSH 22 from My IP
Correct username: ec2-user for Amazon Linux
Correct key pair
Problem 4: aws ec2 describe-instances works

That means your role has EC2 permissions too.

For this lab, it should fail. Check if you attached a broad policy like:

AdministratorAccess

Remove broad permissions and keep only:

AmazonS3FullAccess
PART 10 — Cleanup

After lab:

Step 19: Detach Role
EC2 → Instances.
Select instance.
Actions → Security → Modify IAM role.
Choose:
No IAM role
Update.
Step 20: Terminate EC2
EC2 → Instances.
Select instance.
Instance state → Terminate instance.
Step 21: Delete IAM Role
IAM → Roles.
Search:
EC2-S3-Role
Select role.
Delete role.
Scenario-Based Questions
1. New developer needs EC2 and S3 access

Create an IAM user or IAM Identity Center user, add them to a Developers group, and attach only required EC2 and S3 permissions. Enable MFA. Never share root credentials.

2. User needs access only for 2 days

Use IAM role with temporary credentials through AWS STS. Temporary access expires automatically. Creating an IAM user and deleting it later is less secure.

3. Access keys exposed on GitHub

Immediately deactivate and delete the exposed keys. Create new keys only if needed. Review CloudTrail logs, check for suspicious resources, rotate credentials, and reduce permissions.

4. User needs programmatic access

Create IAM user access keys and configure them using:

aws configure

Permissions must be controlled through IAM policies.

5. 50 employees need same S3 access

Create an IAM group, attach S3 policy to the group, and add users to that group.

6. Revoke access from 20 users

Remove users from the IAM group or detach the policy from the group. Group-based access is easier than editing each user.

7. HR, Dev, Admin need different access

Create separate IAM groups:

HR-Group
Developer-Group
Admin-Group

Attach different policies based on each department’s responsibilities.

8. EC2 needs S3 access without credentials

Attach an IAM role to EC2 with S3 permissions. This avoids storing access keys on the instance.

9. Cross-account access

Create an IAM role in Account B with a trust policy allowing Account A to assume it. Users/services in Account A then assume the role securely.

10. Lambda needs DynamoDB access

Attach an IAM role to Lambda with DynamoDB permissions such as:

dynamodb:GetItem
dynamodb:PutItem
dynamodb:UpdateItem
11. Temporary admin access

Use role assumption with STS and limit the session duration. Do not permanently assign admin access.

12. Restrict S3 access to one bucket

Create a custom IAM policy with the specific bucket ARN only.

Example resources:

arn:aws:s3:::my-bucket-name
arn:aws:s3:::my-bucket-name/*
13. Deny delete actions

Use explicit deny for delete actions.

Example:

Deny s3:DeleteObject

Explicit deny always overrides allow.

14. Time-based access

Use IAM condition:

aws:CurrentTime

This allows access only during specified time windows.

15. IP-based access

Use IAM condition:

aws:SourceIp

Allow only office IP ranges.

16. Secure permission management

Use:

IAM groups
IAM roles
Least privilege policies
MFA
Credential rotation
CloudTrail auditing
IAM Access Analyzer
17. When should root user be used?

Only for critical account-level tasks like billing, account recovery, or initial setup. Never for daily work. Enable MFA.

18. User needs EC2 + S3 + DynamoDB

Attach multiple policies or create one custom least-privilege policy containing only required EC2, S3, and DynamoDB actions.

19. Track user activity

Use:

AWS CloudTrail

CloudTrail records API activity for auditing and investigation.

20. User has too much permission

Review attached policies, remove unnecessary permissions, apply least privilege, and use IAM Access Analyzer to detect overly broad access.
