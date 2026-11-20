# Week 9: AWS Fundamentals

Phase 3 begins. General knowledge; AWS consoles and free-tier terms change,
so check current AWS documentation and pricing.

## Goal

Understand the handful of AWS services an AI backend needs and deploy a
"hello" container by the end of the week.

## Cost safety first

1. Create a new account and turn on multi-factor authentication for root.
2. Create a **budget with an email alert** at a small amount before anything
   else.
3. Never use the root user for daily work. Create an IAM user or use IAM
   Identity Center.
4. Never commit access keys. Delete resources when you finish practising.
5. Pick one region and stay in it.

## Services to learn

| Service | What it is | You use it for |
| --- | --- | --- |
| IAM | Users, roles, policies | Who and what may do what |
| VPC | Private network | Isolating services |
| EC2 | Virtual machines | Simplest place to run Docker |
| ECR | Container registry | Storing your images |
| ECS + Fargate | Run containers without managing servers | The API in production |
| S3 | Object storage | Documents, model files, logs |
| RDS | Managed PostgreSQL | App data, pgvector |
| Secrets Manager / Parameter Store | Secret storage | API keys |
| CloudWatch | Logs, metrics, alarms | Monitoring |
| Application Load Balancer | HTTP entry point | HTTPS, routing, health checks |
| Lambda | Functions on demand | Small event-driven jobs |
| Bedrock | Managed foundation models | Calling models inside AWS |

## IAM in one page

- **User**: a person or a long-lived identity.
- **Role**: an identity assumed temporarily by a service or person. Prefer
  roles over users with keys.
- **Policy**: a JSON document allowing or denying actions on resources.
- **Least privilege**: grant only the actions and resources needed.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-docs-bucket/*"
  }]
}
```

An ECS task gets a **task role** (what your code may call) and an
**execution role** (what ECS needs to pull the image and write logs).

## Networking basics

- A VPC has **public subnets** (reachable from the internet) and **private
  subnets** (not).
- Put the load balancer in public subnets; put containers and databases in
  private subnets.
- A **security group** is a firewall on a resource: allow port 443 to the
  load balancer from anywhere; allow the app port only from the load balancer;
  allow the database port only from the app.

## Do

1. Account, MFA, budget alert, admin user.
2. Install and configure the AWS CLI. Run `aws sts get-caller-identity`.
3. Create an S3 bucket; upload and download a file from the CLI and from
   Python with `boto3`.
4. Launch a small EC2 instance, SSH in, install Docker, run your FastAPI
   container, open it in the browser.
5. Store an API key in Secrets Manager and read it from Python.
6. Send application logs to CloudWatch and find them in the console.
7. **Terminate the instance** at the end.

## Checkpoint

- [ ] You can explain user, role and policy.
- [ ] You can explain public and private subnets and security groups.
- [ ] Your container ran on EC2.
- [ ] A budget alert exists.

## Interview questions

1. What is the difference between an IAM user and a role?
2. Why put a database in a private subnet?
3. Where do you keep an LLM API key on AWS?
4. EC2, ECS Fargate or Lambda for a streaming chat API, and why?
