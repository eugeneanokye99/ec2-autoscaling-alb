# Auto Scaling Web Tier

A single CloudFormation template that provisions a fully working, auto-scaling web tier on AWS.

---

## What gets deployed

| Resource | Details |
|---|---|
| VPC | 10.0.0.0/16, 2 public + 2 private subnets across 2 AZs |
| Internet Gateway | Attached to public subnets |
| NAT Gateway | Regional; sits in Public Subnet A |
| Application Load Balancer | Internet-facing, HTTP:80, round-robin |
| Target Group | HTTP health check on `/` |
| Launch Template | Amazon Linux 2, t3.micro, Apache + `stress` installed via User Data |
| Auto Scaling Group | min 1 / desired 1 / max 4, across both private subnets |
| Scaling Policy | Target Tracking — scale out when avg CPU > 30 % (scale-in automatic) |
| IAM Role / Profile | Grants SSM access for optional in-browser shell (no SSH needed) |

---

## Deploy with CloudFormation GitSync

1. Push this repo to GitHub.
2. In the AWS Console → CloudFormation → **GitSync** → link the repo and point it at `template.yaml`.
3. CloudFormation will create/update the stack automatically on every push.

### Manual deploy (CLI)

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name autoscale-demo \
  --capabilities CAPABILITY_IAM
```

The `ALBDNSName` output gives you the public URL.

---

## How the demo page works

Every instance renders a page showing its own **Instance ID, Private IP, and AZ**.  
Refreshing the ALB URL several times (or using the loop below) proves round-robin distribution.

```bash
ALB="<paste ALBDNSName here>"

# Show instance IDs cycling across requests
for i in $(seq 1 10); do
  curl -s $ALB | grep "Instance ID" | sed 's/<[^>]*>//g'
done
```

---

## Triggering a scale-out event

The template exposes a CGI endpoint that runs `stress` on the instance for 120 seconds.

```bash
# Hammer all instances through the ALB to push avg CPU above 30 %
for i in $(seq 1 20); do
  curl -s "$ALB/cgi-bin/stress.sh" &
done
wait
```

Then watch in **EC2 → Auto Scaling Groups → Activity** or **CloudWatch → Alarms**.  
Within ~2 minutes you should see new instances launch and register with the target group.

---

## Architecture diagram

```
Internet
   │
   ▼
[ALB] ── public-a / public-b
   │
   ▼ (HTTP only, via SG rule)
[EC2 × 1-4] ── private-a / private-b
   │
   ▼ (outbound via NAT)
[NAT Gateway] ── public-a
   │
   ▼
Internet (for yum updates)
```

---

## Security notes

- EC2 instances have **no public IP** and **no inbound SSH**.
- The instance security group only allows port 80 **from the ALB security group**.
- SSM Session Manager is enabled via IAM policy if you need in-instance access without SSH.