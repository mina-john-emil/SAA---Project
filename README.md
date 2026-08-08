# Scalable Web Application with ALB and Auto Scaling

AWS Solutions Architect Associate graduation project (Project 1 of 8), based on the
project brief by Ayman Aly Mahmoud.

## Architecture

![Solution architecture diagram](architecture-diagram.svg)

A production-grade web application deployed on EC2 inside a VPC with public and
private subnets across two Availability Zones. High availability and scalability
come from an Application Load Balancer, an Auto Scaling Group, and a CloudFront
distribution in front for caching static assets. A Multi-AZ RDS instance is the
database backend, with all compute in private subnets — nothing app-tier or
database-tier is reachable directly from the internet.

**Request flow:** Users → Route 53 (DNS) → CloudFront (edge caching) → WAF
(OWASP Top 10 rules) → ALB (Layer 7 routing, health checks) → target group →
EC2 instances in private app subnets (across 2 AZs) → Multi-AZ RDS in private
DB subnets.

## Key AWS services

| Service | Role |
|---|---|
| VPC | Public & private subnets across 2 AZs, NAT Gateway, security groups, NACLs |
| EC2 + Auto Scaling | Launch template, target-tracking scaling policy on CPU |
| ALB + WAF | Layer 7 routing, health checks, AWS managed rule group for OWASP Top 10 |
| CloudFront | Caches static assets, reduces latency for global users |
| RDS (Multi-AZ) | MySQL with synchronous standby and automated failover |
| Route 53 | DNS, alias record to the ALB, health checks |
| Systems Manager | Session Manager for instance access — no SSH keys, no bastion host |
| CloudWatch + SNS | Dashboards, CPU and unhealthy-host alarms, email notifications |

## Learning outcomes

- Design VPCs with correct subnet, route table, and NAT Gateway configurations
- Build highly available architectures across multiple Availability Zones
- Configure ALB listener rules and target group health checks
- Implement Auto Scaling with a target-tracking scaling policy
- Secure applications with WAF, security groups, and private subnets
- Use Systems Manager Session Manager as a bastion-free access alternative

## Repository structure

```
.
├── README.md                     # this file
├── architecture-diagram.svg      # solution architecture diagram
└── infrastructure/
    └── main.yaml                 # CloudFormation template (all resources)
```

## Deploying

Prerequisites: an AWS account, the AWS CLI configured with credentials that can
create VPC/EC2/ELB/RDS/IAM/WAF/CloudWatch resources.

```bash
aws cloudformation deploy \
  --template-file infrastructure/main.yaml \
  --stack-name saa-project1 \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      DBUsername=admin \
      DBPassword='<choose-a-strong-password>' \
      AlarmEmail='you@example.com'
```

After the stack finishes (RDS Multi-AZ typically takes 10-15 minutes), confirm
the SNS email subscription, then get the app URL:

```bash
aws cloudformation describe-stacks \
  --stack-name saa-project1 \
  --query "Stacks[0].Outputs[?OutputKey=='ALBDNSName'].OutputValue" \
  --output text
```

Open that DNS name in a browser — you should see a page identifying the
instance that served the request. Refresh a few times or terminate an instance
to see the ASG and health checks in action.

To tear everything down and avoid ongoing charges:

```bash
aws cloudformation delete-stack --stack-name saa-project1
```

## Notes / extensions (not in the base template)

- **HTTPS**: add an ACM certificate and a 443 listener on the ALB; redirect 80 → 443.
- **CloudFront in front of the ALB**: add a `AWS::CloudFront::Distribution` with
  the ALB as a custom origin, and point Route 53 at CloudFront instead of the ALB.
- **Route 53 record**: add an alias record once you own a hosted zone, pointing
  at the ALB (or CloudFront) DNS name.
- **Two NAT Gateways** (one per AZ) for full AZ-independence — the template uses
  one to keep cost down, which is a single point of failure for outbound traffic
  from private subnets, worth calling out explicitly in your write-up.

## Deliverables checklist

- [x] Solution architecture diagram (`architecture-diagram.svg`)
- [x] GitHub repo with documentation (this README + IaC)
- [ ] Live URL or demo video (deploy the stack and add the link/video here)
