# Multi-Specialty Clinic Appointment Booking Platform on AWS

Final project for the **SAA25-S1 – AWS Solution Architect** program.
This repository contains the documented architecture for a medical appointment booking platform, including the diagram and the reasoning behind each design decision.


---

## 1. Overview

A web platform that allows:

- **Patients** to search for a doctor by specialty, view available slots, book or cancel appointments, and receive reminders.
- **Doctors** to manage their schedules and appointments.
- **Admins** to manage specialties, doctors, and reports.

The key technical requirement is **preventing double-booking** of the same doctor at the same time, while protecting patient data.

## 2. Requirements and Assumptions

### Functional requirements
- Sign-in with three roles: Patient / Doctor / Admin.
- Multiple specialties, each with multiple doctors.
- Book, modify, and cancel appointments.
- Confirmation and reminder notifications by SMS and email.
- Upload and view files (lab results, reports).

### Non-functional requirements (assumptions)

 Item | Assumed value 

 Expected users | [e.g. 5,000 monthly active patients] 
 Peak usage | [e.g. 9–11 AM] 
 Target availability | [e.g. 99.9%]
 Online payments | [out of scope for this version / in scope]
 Region | [eu-central-1, based on proximity and compliance]
 Data sensitivity | Personal medical data requiring encryption and auditing 

## 3. Architecture Diagram

![AWS Architecture Diagram](diagrams/architecture.jpg)


## 4. Architecture Walkthrough (Request Flow)

1. The **patient** opens the website through **Route 53**, which resolves to **CloudFront** (protected by **AWS WAF**).
2. The static frontend is served from **S3** through CloudFront and cached at edge locations.
3. Sign-in is handled by **Amazon Cognito**, which returns a JWT containing the user's group (Patient / Doctor / Admin).
4. API calls go to **API Gateway**, which validates the token with a Cognito authorizer.
5. API Gateway reaches the VPC through a **VPC Link** to an **internal Application Load Balancer (ALB)**. User traffic does not pass through the Internet Gateway.
6. The ALB distributes requests across **ECS Fargate tasks** running in private subnets across the three Availability Zones.
7. The application reads and writes to **Amazon RDS (PostgreSQL) Multi-AZ DB cluster** (one writer and two standby readers, one per AZ), and uses **ElastiCache** to cache frequently requested doctor schedules.
8. After a successful booking, the application sends a message to **SQS**.
9. **Lambda** consumes messages from SQS and sends notifications through **SNS** (SMS) and **SES** (email).
10. **EventBridge Scheduler** triggers Lambda to send reminders before appointments.
11. Medical files are stored in **S3**, encrypted with **KMS**.

### Networking (VPC)
- A single VPC spanning **three Availability Zones** (A, B, and C), each with a public, a private app, and a private data subnet.
- **Public subnets:** NAT Gateways (one per AZ).
- **Private app subnets:** ECS Fargate tasks and the internal ALB.
- **Private data subnets:** RDS and ElastiCache, with no direct route to the internet.
- **Internet Gateway (IGW):** attached to the VPC. It is used only for outbound access from the NAT Gateways, since user traffic enters through CloudFront and API Gateway.
- **NAT Gateways:** one per AZ in the public subnets. Private route tables send `0.0.0.0/0` to the NAT Gateway in the same AZ, and the public route table sends `0.0.0.0/0` to the IGW. Outbound access is needed only for real external calls (for example a payment gateway or third-party API).

### VPC Endpoints
- **S3 Gateway Endpoint:** private access to S3 without going through NAT, at no additional cost.
- **Interface Endpoints (PrivateLink):** for services such as SQS, SNS, ECR, CloudWatch Logs, Secrets Manager, and KMS.

### Auto Scaling
- **ECS Service Auto Scaling** (Application Auto Scaling) with target tracking policies:
  - Average CPU at `[60%]`.
  - `ALBRequestCountPerTarget` at `[value determined after load testing]`.
- Minimum 3 tasks (one per AZ) and maximum `[6]`.
-  scheduled scaling to add capacity during peak hours.
- **RDS Storage Auto Scaling** enabled, with the option to add a Read Replica if read traffic grows.

## 5. Design Decisions

| Decision | Reason | Alternative considered |
|---|---|---|
| **RDS PostgreSQL** for booking data | Bookings need transactions and unique constraints (`UNIQUE(doctor_id, slot_start)`) to prevent double-booking | DynamoDB: good for key-value access but harder for relational queries across doctors, specialties, and patients |
| **ECS Fargate** | No server management, automatic scaling, pay-per-use | EC2: more control but higher operational overhead |
| **Cognito** | Ready-made user and role management with native API Gateway integration | Custom authentication: more effort and higher security risk |
| **SQS + Lambda** for notifications | Decouples notifications from the booking path, so a slow notification never delays a booking; built-in retries | Sending directly from the app: slows booking and adds failure points |
| **ElastiCache** | Reduces database load for repeated doctor schedule reads | No cache: higher load on RDS at peak times |
| **CloudFront + S3** for the frontend | Fast, low-cost, highly available | Hosting the frontend on ECS: higher cost and less efficient |
| **API Gateway + VPC Link + internal ALB** | Keeps the ALB private and off the public internet while still exposing a managed API | Internet-facing ALB: larger attack surface |
| **VPC Endpoints** | Keeps traffic to AWS services on the private network and reduces reliance on NAT | Relying entirely on NAT: more exposure and higher data transfer cost |

## 6. Mapping to the Six Well-Architected Pillars

**Operational Excellence**
CloudWatch Logs and Alarms for monitoring, CloudTrail for auditing, CI/CD for deployments `[e.g. CodePipeline or GitHub Actions]`, and infrastructure defined as code `[]`.

**Security**
- Least privilege through IAM roles, with no long-lived access keys.
- Encryption in transit (TLS) and at rest (KMS) for RDS and S3.
- WAF in front of CloudFront and strict Security Groups between tiers.
- Database in isolated subnets with no direct internet access.
- Internal ALB reachable only through the VPC Link.
- Secrets (database credentials) stored in Secrets Manager.

**Reliability**
Three Availability Zones for ECS, the ALB, and the RDS Multi-AZ DB cluster (1 writer + 2 standbys); Auto Scaling; automated RDS backups; retries and a Dead Letter Queue in SQS; one NAT Gateway per AZ.

**Performance Efficiency**
CloudFront for static content, ElastiCache for repeated reads, and automatic scaling based on load.

**Cost Optimization**
Pay-per-use Fargate and Lambda, S3 lifecycle rules for older files, an S3 Gateway Endpoint instead of NAT for S3 traffic, only the necessary Interface Endpoints, and right-sizing of resources.

**Sustainability**
Managed and serverless services for better resource utilization, and automatic scaling to reduce idle capacity.

## 7. Security and Access Management (IAM)

### Identities
- **Application users:** Cognito User Pool with Patients / Doctors / Admins groups.
- **ECS Task Role:** grants the application specific permissions on SQS and S3.
- **Lambda Execution Role:** permissions to read from SQS and send through SNS and SES.

### Example policy for the ECS Task Role (least privilege)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage"],
      "Resource": "arn:aws:sqs:<REGION>:<ACCOUNT_ID>:booking-events"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::<BUCKET_NAME>/patient-files/*"
    }
  ]
}
```

> IAM evaluation logic: an explicit **Deny** always wins, then an explicit **Allow**; if neither applies, the result is an implicit **Deny**. This design relies on that by granting only the minimum required permissions.

## 8. Cost Estimate

The final estimate should be calculated with the [AWS Pricing Calculator](https://calculator.aws/) using the assumptions in Section 2.

| Service | Main cost driver | Monthly estimate |
|---|---|---|
| ECS Fargate | Number and size of tasks | `[ ]` |
| RDS Multi-AZ DB cluster | Instance size (3 instances) and storage | `[ ]` |
| ElastiCache | Node size | `[ ]` |
| ALB | Hours and request count | `[ ]` |
| NAT Gateway | Hours (3 gateways) and data processed | `[ ]` |
| Interface Endpoints | Hours per AZ | `[ ]` |
| CloudFront / S3 | Data transfer and storage | `[ ]` |
| Lambda / SQS / SNS / SES | Requests and messages | `[ ]` |
| **Total** | | `[ ]` |

## 9. Future Improvements

- Multi-Region setup for disaster recovery.
- RDS Proxy to manage connections at larger scale.
- Online payment gateway integration.
- Amazon Pinpoint or WhatsApp Business for richer notifications.
- Analytics and reporting with Athena and QuickSight.

## 10. Repository Structure

```
clinic-booking-aws/
├── README.md
├── diagrams/
│   ├── architecture.png
│   └── architecture.drawio
└── docs/
    └── iam-policies.json   
```

## 11. References

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Reference Architectures](https://aws.amazon.com/architecture/)
- [AWS Pricing Calculator](https://calculator.aws/)

---

Creator: [Ahmed Abdilghany Eldyasty ]
