# AI-Assisted Production Cloud Deployment Platform

Containerized Flask application deployed on AWS using Docker, Terraform, GitHub Actions CI/CD, and secure OIDC-based authentication.

## How it works

```text
git push → CI: build image, run it, test /health
manual run → CD: assume AWS role via OIDC → build arm64 image → push to ECR → deploy to EC2 via SSM
```

| Part | Detail |
|---|---|
| App | Flask, `/` and `/health` endpoints, port 5000 |
| Container | `python:3.12-slim` Docker image |
| CI | On push and PR to `main`: builds the image, runs the container, `curl --fail /health` |
| CD | Manual (`workflow_dispatch`): OIDC login, arm64 build with QEMU/Buildx, push to ECR, deploy over SSM |
| Registry | Amazon ECR |
| Compute | EC2 `t4g.small` (Amazon Linux 2023, arm64), ap-south-1 |
| Infrastructure | Terraform |

## Security

- **OIDC, no long-lived keys.** The CD workflow assumes `ai-cloud-github-actions-role` through GitHub's OIDC provider. The trust policy only accepts tokens from this repo's `main` branch.
- **Scoped pipeline permissions.** The role can push images to this one ECR repository and run `ssm:SendCommand` with the `AWS-RunShellScript` document on this one instance.
- **No SSH.** The instance has no key pair and no port 22. Access is through SSM Session Manager, using an EC2 role with `AmazonSSMManagedInstanceCore` and ECR read-only access.
- **Network.** The security group allows inbound TCP 5000 only.

The OIDC identity provider was created manually in IAM. The role and its policies are in Terraform.

## Infrastructure (Terraform)

The Terraform in [`terraform/`](terraform/) defines:

- ECR repository
- EC2 IAM role, policy attachments and instance profile
- Security group (TCP 5000 only)
- EC2 instance (`t4g.small`, arm64)
- CloudWatch log group `/ai-cloud/app` (7-day retention)
- CloudWatch alarm on EC2 CPU above 70%
- GitHub Actions IAM role with its OIDC trust policy and scoped policies

## Failure test and recovery

I stopped the running container through SSM to simulate a failure, confirmed the outage, then restarted it and verified recovery.

| Step | Evidence |
|---|---|
| 1. Stop and start the container | [View](docs/screenshots/failure-1-container-stopped.png) |
| 2. App unreachable (`ERR_CONNECTION_REFUSED`) | [View](docs/screenshots/failure-2-service-unreachable.png) |
| 3. `/health` returns healthy again | [View](docs/screenshots/failure-3-service-recovered.png) |

**Finding:** nothing restarted the container or notified me. Recovery was manual, because the container has no restart policy.

## Evidence

| Stage | Screenshot |
|---|---|
| App running locally | [View](docs/screenshots/flask-app-running-locally.png) |
| Local `/health` endpoint | [View](docs/screenshots/flask-health-endpoint.png) |
| Local Docker build and run | [View](docs/screenshots/docker-build-and-run.png) |
| CI run passing | [View](docs/screenshots/ci-pipeline-success.png) |
| Terraform apply | [View](docs/screenshots/terraform-apply.png.png) |
| EC2 instance running, 3/3 checks | [View](docs/screenshots/ec2-running.png) |
| ECR image pulled and run on EC2 | [View](docs/screenshots/ecr-pull-and-run.png) |
| CD run passing | [View](docs/screenshots/cd-deploy-success.png) |
| Push, then remote check with SSM `docker ps` | [View](docs/screenshots/cicd-github-to-ec2-deployment.png.png) |
| App live on EC2 | [View](docs/screenshots/live-on-ec2.png) |
| CloudWatch log group | [View](docs/screenshots/cloudwatch-logs.png) |

## What I learned

- **OIDC replaces stored keys.** GitHub gets short-lived credentials by assuming a role, and the trust policy limits which repo and branch can do it.
- **Architecture must match.** The EC2 instance is arm64, so the CD workflow builds an arm64 image with QEMU and Buildx.
- **Infrastructure as code is repeatable.** The whole stack is defined in Terraform, so it can be recreated with one `terraform apply`.
- **Deploy without SSH.** SSM lets the pipeline run commands on the instance, so no port 22 or key pair is needed.
- **Test for failure, not only success.** Stopping the container showed that nothing detects or recovers an outage yet.

## Limitations and next steps

| Limitation | Next step |
|---|---|
| Container has no restart policy | Run with `--restart unless-stopped` |
| The CPU alarm has no notification action | Send it to an SNS topic |
| Redeploys don't ship container logs to CloudWatch | Add the `awslogs` log driver to the `docker run` in CD |
| Public IP changes on redeploy | Attach an Elastic IP |
| Plain HTTP on port 5000, no load balancer | Put an ALB with HTTPS in front |
| CD is triggered by hand | Trigger it after CI succeeds on `main` |
| Image tag is `latest` only | Tag images with the commit SHA |

## Author

**Anjana J** — Electronics & Communication Engineering, Jyothi Engineering College
Aspiring Cloud Engineer | AWS · [GitHub](https://github.com/Anjana-108)
