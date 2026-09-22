# Backstage AWS SAM RDS Template

This repository is a Backstage scaffolding template for creating a new repo for deploying an AWS RDS database using AWS SAM and GitHub Actions.

## What this template creates

When used from Backstage, the template generates a new project that includes:

- An AWS SAM template defining an Aurora PostgreSQL cluster
- A Secrets Manager secret for the database admin password
- VPC/subnet and security group configuration for the database
- SSM parameters for cluster host, port, username, and admin credentials
- GitHub Actions workflows for validation and deployment
- A generated repository skeleton ready to be published to GitHub and registered back into Backstage

## Template structure

- `template.yaml` — the Backstage Scaffolder template definition
- `skeleton/` — the generated project skeleton used by the template
- `pipeline/` — GitHub Actions workflow templates added to the generated repository

## Included infrastructure

The generated SAM stack provisions:

- Aurora PostgreSQL database cluster
- DB instances with serverless v2 scaling
- DB subnet group and security group
- Secret in AWS Secrets Manager
- CloudWatch log exports for Postgres logs
- SSM configuration values for runtime integration

This is useful when a service needs a managed database but you want the environment to be created consistently and repeatably from a Backstage-driven workflow.

## How it works

The scaffold collects basic metadata from Backstage, including:

- component name and description
- owner and domain/system context
- deployment account and environment metadata
- database settings like engine, version, username, port, and scaling capacity

It then:

1. fetches the relevant catalog entities from Backstage
2. renders the project skeleton from `skeleton/`
3. adds CI/CD workflow files from `pipeline/`
4. publishes the repo to GitHub
5. registers the new component in Backstage

## Requirements

Before using this template in a Backstage installation, ensure the following are available:

- Backstage with the Scaffolder plugin enabled
- GitHub publisher integration configured for repository creation
- Catalog entities for domain, system, owner, environment, and cloud account
- AWS account and VPC/subnet configuration for the target deployment environment
- AWS SAM and related deployment tooling available in the generated project pipeline

## Notes

This template focuses specifically on a PostgreSQL-backed RDS deployment flow and is best suited for infrastructure teams standardizing database provisioning through Backstage.
