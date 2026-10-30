#### flask_docker_lab

> **About this fork:** a copy of [cloudshiftstrategies/flask_docker_lab](https://github.com/cloudshiftstrategies/flask_docker_lab) used to work through the lab. Fork-specific commits change `SITE_TITLE` in `app/config.py` (to see a new version roll out) and re-encrypt `ecr.secrets.enc` for this fork's Travis CI build.

This is a basic Flask web application that will be deployed to AWS ECS using docker

It is used as part of an AWS Meetup Lab

[Cloud Shift Strategies - Travis Ci/Cd Lab](https://www.cloudshiftstrategies.com/travisLab.html)

Go there for instructions

## Features

- Flask site with a Bootstrap UI: a home page (`/`) and a CPU load generator page (`/loadgen`, 300,000 loop iterations, shows the elapsed time)
- Docker image and `docker-compose.yaml` for local runs on port 5000
- Travis CI: builds the image, runs `linkchecker` against the running site, and on git tags pushes the image to Amazon ECR
- Terraform for the AWS side: VPC (public and app subnets across three AZs, NAT gateway), ECR repository, ECS cluster on EC2 Auto Scaling, Application Load Balancer, ECS service and task definition, CloudWatch log groups and an SNS topic for stopped-task events
- Optional developer desktop: a Packer-built Ubuntu AMI with Docker, Terraform, AWS CLI, Travis CLI, Atom and a VNC server, deployed with Terraform

## Tech stack

- Python 2.7 (`python:2.7` image) and Flask
- Docker, Docker Compose
- Travis CI
- Terraform (0.11-style syntax), AWS ECS (EC2 launch type), ECR, ALB
- Packer (developer desktop)

## Project structure

```
.
├── app/
│   ├── Dockerfile               # python:2.7, runs run.py
│   ├── requirements.txt         # flask
│   ├── config.py                # Site title, static URL, nav pages, CSS/JS includes
│   ├── run.py                   # Starts Flask on 0.0.0.0:5000 (debug mode)
│   └── app/                     # Flask app: routes and templates
├── docker-compose.yaml          # Builds ./app, maps 5000:5000, mounts ./app at /code
├── .travis.yml                  # CI: build, link check, push on tag
├── create_ecr_secrets.sh        # Writes ecr.secrets from the AWS CLI and encrypts it for Travis
├── docker_push.sh               # Tags and pushes the image to ECR (used by Travis)
├── ecr.secrets.enc              # Travis-encrypted ECR login file
├── infrastructure/
│   ├── requirements-testsuite.txt   # requests, linkchecker
│   ├── travis.yml.example
│   └── terraform/               # VPC, ECR, ECS, ALB, ASG, CloudWatch, SNS
│       └── scripts/             # create_s3bucket.py + s3state.template for remote state
├── developer_desktop/           # Packer template, Terraform and setup scripts (see its readme.md)
└── todo.txt
```

## Prerequisites

- Docker and Docker Compose (local runs)
- For CI/CD: a Travis CI account and the Travis CLI, the AWS CLI with credentials
- For AWS: Terraform 0.11-era (the code uses `"${var.x}"` interpolation), Python 2 with `boto3` and `pyhcl` for the state bucket script

## Run locally

```bash
docker-compose up --build
# open http://localhost:5000
```

`./app` is mounted into the container, so code edits reload through Flask's debug server.

Without Docker (Python 2.7):

```bash
cd app
pip install -r requirements.txt
python run.py
```

## Test

Travis runs a link check against the running site. To run it yourself:

```bash
pip install -r infrastructure/requirements-testsuite.txt
docker-compose up -d --build
linkchecker --anchors --cookies --check-html --check-css http://localhost:5000
```

## Configuration

Site settings live in `app/config.py`: `SITE_TITLE`, `STATIC_URL` (S3 location of static images), `FAV_ICON`, `PAGES` (nav menu), `CSS_INCLUDES`, `JS_INCLUDES`.

Environment variables and secrets, by name only:

| Name | Where | Meaning |
|---|---|---|
| `ECR_USER`, `ECR_PASS`, `ECR_URL`, `ECR_DNS` | `ecr.secrets` (git-ignored) | ECR login, written by `create_ecr_secrets.sh`, read by `docker_push.sh` |
| `encrypted_<id>_key`, `encrypted_<id>_iv` | Travis repository settings | Keys Travis uses to decrypt `ecr.secrets.enc` |
| `TRAVIS_TAG` | Travis | Extra image tag pushed on tagged builds |
| `AWS_PROFILE` | Your shell | AWS CLI profile for Terraform, Packer and the scripts |

Terraform inputs are in `infrastructure/terraform/variables.tf`. Main ones: `region` (default `us-west-2`), `availZones`, `projectName`, `stageName`, `projectAcronym`, `containerName`, `containerTag`, `containerPort`, `containerCount`, `containerMem`, `ecsInstanceType`, `tgtGrp*Size`, `vpcCidr`, `publicCidrs`, `appCidrs`.

## Deploy

1. Create the infrastructure:
   ```bash
   cd infrastructure/terraform
   # optional: remote state in a new S3 bucket (writes ../s3state.tf from s3state.template)
   (cd scripts && pip install -r requirements.txt && python create_s3bucket.py)
   terraform init
   terraform plan
   terraform apply
   ```
   Outputs: `LOADBALANCER_DNS` and `ECR_REPOSITORY`.
2. Create and encrypt the ECR login for Travis (needs AWS and Travis CLI logins):
   ```bash
   ./create_ecr_secrets.sh
   ```
   It uses `aws ecr get-login`, which only exists in AWS CLI v1, and hardcodes `us-west-2`.
3. Push a git tag. Travis builds the image and `docker_push.sh` pushes `:latest` and `:<tag>` to ECR.
4. Set `containerTag` in `variables.tf` to the new tag and run `terraform apply` to roll the ECS service.
5. Open `http://<LOADBALANCER_DNS>`.

Clean up with `terraform destroy` in `infrastructure/terraform`.

The developer desktop has its own steps in [developer_desktop/readme.md](developer_desktop/readme.md).

## Notes

- `ecr.secrets` holds a short-lived ECR password. It is git-ignored; only the encrypted `ecr.secrets.enc` is committed.
- The ECR login token expires after 12 hours, so `create_ecr_secrets.sh` must be re-run before later deploys.
- The developer desktop's IAM role has an admin policy and a fixed VNC password. Use it only in a lab account and destroy it afterwards.
