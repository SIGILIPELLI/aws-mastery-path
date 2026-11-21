---
title: "Final Quiz"
description: "Test what you have learned across all four levels of AWS Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "You must give a new teammate access to deploy but not to billing or IAM. What is the best IAM approach?",
    "options": [
      "Share the root credentials",
      "Add them to a group with a least-privilege policy",
      "Give AdministratorAccess to all",
      "Share your access key"
    ],
    "correctIndex": 1,
    "explanation": "Groups carry policies for a team, and least privilege limits blast radius."
  },
  {
    "question": "The root user's credentials leak. Why is that worse than a normal IAM user leak?",
    "options": [
      "Root only works in one region",
      "The root user cannot be restricted by any policy",
      "Root can't access billing",
      "Root users have no MFA option"
    ],
    "correctIndex": 1,
    "explanation": "It has unrestricted access to everything, including billing."
  },
  {
    "question": "A Lambda function needs to read one S3 bucket. Where should that permission live?",
    "options": [
      "In the bucket name",
      "On the function's execution role",
      "In the function code as keys",
      "On the caller's identity"
    ],
    "correctIndex": 1,
    "explanation": "The execution role, not the caller, determines what the function can access."
  },
  {
    "question": "You want SSH to an EC2 instance only from your own IP. Which security group rule?",
    "options": [
      "Inbound TCP 80 from you",
      "Outbound TCP 22 only",
      "Inbound TCP 22 from your IP /32",
      "Inbound TCP 22 from 0.0.0.0/0"
    ],
    "correctIndex": 2,
    "explanation": "Restricting the source reduces who can even attempt to connect."
  },
  {
    "question": "Your RDS database must accept connections only from your app servers. What is the standard pattern?",
    "options": [
      "Open 5432 to 0.0.0.0/0",
      "Allow port 5432 from the app's security group (--source-group)",
      "Make the database public",
      "Use a NAT Gateway"
    ],
    "correctIndex": 1,
    "explanation": "Source-group rules cover every current and future member of the app tier."
  },
  {
    "question": "A private-subnet instance must download updates but never be reachable from the internet. What do you add?",
    "options": [
      "A public IP",
      "A NAT Gateway",
      "An Elastic IP",
      "An Internet Gateway route in its subnet"
    ],
    "correctIndex": 1,
    "explanation": "NAT lets private resources make outbound calls only."
  },
  {
    "question": "Your static site on S3 needs managed HTTPS, global caching and a bucket that stays private. What do you use?",
    "options": [
      "An ALB",
      "A public bucket policy",
      "CloudFront with Origin Access Control",
      "Route 53 alone"
    ],
    "correctIndex": 2,
    "explanation": "OAC lets CloudFront read a private bucket."
  },
  {
    "question": "Traffic to your web tier spikes at night, and one instance is a single point of failure. What combination fixes both?",
    "options": [
      "An Elastic IP",
      "A larger single instance",
      "An Auto Scaling Group behind an Application Load Balancer",
      "A bigger EBS volume"
    ],
    "correctIndex": 2,
    "explanation": "The ASG replaces failed instances and scales; the ALB spreads traffic and health-checks targets."
  },
  {
    "question": "Orders must be processed by one of many workers, and failed messages must not block the queue forever. What do you use?",
    "options": [
      "A CloudWatch alarm",
      "SQS with a dead-letter queue",
      "A Route 53 record",
      "An SNS topic only"
    ],
    "correctIndex": 1,
    "explanation": "SQS delivers each message to one consumer; the DLQ catches repeated failures."
  },
  {
    "question": "One order event must trigger email, inventory and analytics independently. Which pattern?",
    "options": [
      "SNS fan-out to multiple SQS queues",
      "A cron job",
      "A single SQS queue",
      "Direct Lambda chaining"
    ],
    "correctIndex": 0,
    "explanation": "SNS copies the message to every subscriber."
  },
  {
    "question": "A DynamoDB table must return all orders for one customer in one efficient query. How do you model the key?",
    "options": [
      "Partition key customerId and sort key orderId",
      "No key",
      "Sort key only",
      "Partition key orderId only"
    ],
    "correctIndex": 0,
    "explanation": "Items sharing a partition key are stored together and range-queryable."
  },
  {
    "question": "A password for your database must be rotated automatically every 30 days. Which service?",
    "options": [
      "Secrets Manager with rotation",
      "Plain Lambda environment variables",
      "S3",
      "CloudWatch Logs"
    ],
    "correctIndex": 0,
    "explanation": "Secrets Manager supports automatic rotation via a Lambda function."
  },
  {
    "question": "You want to be warned before your monthly spend passes 80% of $50, but a budget alone won't stop resources. What does it do?",
    "options": [
      "Deletes unused resources",
      "Stops everything at 100%",
      "Refunds overspend",
      "Notifies you; stopping resources needs an extra automated action"
    ],
    "correctIndex": 3,
    "explanation": "A budget only notifies unless wired to an action."
  },
  {
    "question": "You describe your infrastructure in a template so it can be recreated and torn down with one command. What is this called and which native service does it?",
    "options": [
      "Infrastructure as Code with CloudFormation",
      "IAM",
      "A cron job",
      "Manual provisioning with the console"
    ],
    "correctIndex": 0,
    "explanation": "The stack is created and deleted as a unit."
  },
  {
    "question": "Several teams share one AWS account and one team's mistake took down another's production. What is the structural fix?",
    "options": [
      "A larger instance",
      "Separate accounts under Organizations",
      "More IAM users",
      "More tags"
    ],
    "correctIndex": 1,
    "explanation": "Accounts give hard blast-radius boundaries for billing, quotas and IAM."
  },
  {
    "question": "You want to stop anyone in an OU from using regions outside us-east-1 and us-west-2 even if they are administrators. What do you use?",
    "options": [
      "An IAM user policy",
      "A security group",
      "A NACL",
      "A Service Control Policy"
    ],
    "correctIndex": 3,
    "explanation": "SCPs set a ceiling for accounts and OUs."
  },
  {
    "question": "A request passes through API Gateway, Lambda, DynamoDB and SQS and is slow somewhere. Which service shows where?",
    "options": [
      "S3 versioning",
      "X-Ray tracing",
      "Route 53",
      "Config"
    ],
    "correctIndex": 1,
    "explanation": "One trace ID follows the request across services."
  },
  {
    "question": "Your RTO is minutes and your RPO is seconds, and you have a moderate budget. Which DR strategy fits best?",
    "options": [
      "Backup and restore",
      "Pilot light with no replica",
      "Warm standby",
      "No DR"
    ],
    "correctIndex": 2,
    "explanation": "Backup and restore takes hours; warm standby recovers in minutes with seconds-minutes of data loss."
  },
  {
    "question": "Ad-hoc queries over years of S3 data should only be billed for what they scan. Which pairing fits?",
    "options": [
      "DynamoDB scans",
      "EC2 with cron",
      "RDS with a big instance",
      "Athena with a partitioned columnar format like Parquet"
    ],
    "correctIndex": 3,
    "explanation": "Athena bills per byte scanned; partitions and columnar formats cut it."
  },
  {
    "question": "Your pipeline builds a Docker image per commit. How should it tag images so you can roll back to a specific build?",
    "options": [
      "Untagged",
      "With latest only",
      "With the date only",
      "With the commit SHA"
    ],
    "correctIndex": 3,
    "explanation": "A SHA tag identifies exactly which build is running."
  }
]
</script>
