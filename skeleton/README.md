# ${{ values.system_name }} / ${{ values.component_name }}

${{ values.component_description }}


## New Project Getting Started
This repository was generated from a template intended to get a new RDS database up running quickly. It uses the [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) to define the infrastructure as code.

The [project handbook](docs/handbook/00-index.md) covers features, project creation,
configuration, architecture, CI/CD, observability, and common database tasks.
It is also available in the component's **Docs** tab when Backstage TechDocs is enabled.


## Architecture
This is an Aurora PostgreSQL Serverless v2 database stack. It is built on top of the following AWS services:
* RDS (Aurora PostgreSQL)
* Secrets Manager (admin credential)
* EC2 VPC networking (DB subnet group and security group)
* SSM Parameter Store (connection details for consumers)

The security group starts with no ingress rules; see the handbook's
_Common tasks_ page to grant a service access.
