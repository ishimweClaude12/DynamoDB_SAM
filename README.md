# DynamoDB SAM Lab

An Amazon DynamoDB table provisioned with AWS SAM and deployed to two environments (dev and prod) through two dedicated GitHub Actions pipelines. Each environment has its own SAM pipeline bootstrap resources, including its own S3 artifacts bucket.

## Architecture

```
 feature work
      |
      v
  develop branch ──push──> Deploy DEV workflow  ──OIDC──> dev pipeline role
                                                           |  sam package -> dev artifacts bucket
                                                           |  sam deploy (dev CFN role)
                                                           v
                                                    stack dynamodb-lab-dev  -> table orders-dev

  PR develop -> main
      |
      v
  main branch ──push──> Deploy PROD workflow ──OIDC──> prod pipeline role
                                                           |  sam package -> prod artifacts bucket
                                                           |  sam deploy (prod CFN role)
                                                           v
                                                    stack dynamodb-lab-prod -> table orders-prod
```

No long-lived AWS keys are stored in GitHub. Each workflow assumes its own environment's role through GitHub OIDC, and the role trust policy only accepts tokens from the matching branch.

## Table design

| Item | Value |
|---|---|
| Table name | `orders-dev` / `orders-prod` |
| Partition key | `OrderId` (String) |
| Non-key attributes | `CustomerId` (S), `OrderStatus` (S), `OrderDate` (S), `TotalAmount` (N) |
| GSI 1 | `CustomerOrdersIndex`: `CustomerId` (HASH) + `OrderDate` (RANGE), projection ALL |
| GSI 2 | `OrderStatusIndex`: `OrderStatus` (HASH) + `OrderDate` (RANGE), projection ALL |
| Billing mode | `PAY_PER_REQUEST` (On-Demand) |
| Table class | `STANDARD_INFREQUENT_ACCESS` (not the default `STANDARD`) |
| Prod only | Deletion protection and point-in-time recovery enabled |

Access patterns this supports: get an order by ID, list a customer's orders by date, list all orders in a given status by date.

## Repository layout

```
.
├── template.yaml                         SAM template (the DynamoDB table)
├── .github/workflows/
│   ├── deploy-dev.yml                    dev pipeline (develop branch)
│   └── deploy-prod.yml                   prod pipeline (main branch)
├── .aws-sam/pipeline/pipelineconfig.toml written by sam pipeline bootstrap
├── docs/
│   ├── sam-generated-pipeline.reference.yaml   original sam pipeline init output (not active)
│   └── sample-items.json                 items for console testing
└── README.md
```

## Setup

Prerequisites: AWS CLI v2 configured with an admin profile, AWS SAM CLI, Git, and an empty GitHub repository named `DynamoDB_SAM`.

### 1. Initial commit on main

Push only this README and `.gitignore` to `main` first, so the prod pipeline does not run before dev has been proven.

```bash
git init && git branch -M main
git add README.md .gitignore
git commit -m "chore: initial commit"
git remote add origin git@github.com:ishimweClaude12/DynamoDB_SAM.git
git push -u origin main
git checkout -b develop
```

### 2. Bootstrap the dev stage

This creates the CloudFormation stack `aws-sam-cli-managed-dev-pipeline-resources` containing the dev artifacts bucket, the dev pipeline execution role, the dev CloudFormation execution role, and the GitHub OIDC provider.

```bash
sam pipeline bootstrap \
  --stage dev \
  --region eu-north-1 \
  --permissions-provider oidc \
  --cicd-provider github-actions \
  --oidc-provider github-actions \
  --oidc-provider-url https://token.actions.githubusercontent.com \
  --oidc-client-id sts.amazonaws.com \
  --github-org "ishimweClaude12@77576560" \
  --github-repo "DynamoDB_SAM@1422178740" \
  --deployment-branch develop \
  --no-create-image-repository
```

Running `sam pipeline bootstrap` with no flags and answering the prompts gives the same result.

### 3. Bootstrap the prod stage

Same command with `--stage prod` and `--deployment-branch main`. This creates a second, independent stack `aws-sam-cli-managed-prod-pipeline-resources` with its own bucket and roles. SAM reuses the GitHub OIDC provider created in step 2.

### 4. Generate the SAM pipeline, then split it

```bash
sam pipeline init
```

Choose AWS Quick Start Pipeline Templates, then GitHub Actions, then OIDC, and select `dev` as stage 1 and `prod` as stage 2. SAM writes `.github/workflows/pipeline.yaml`, a single workflow that deploys both stages.

That generated file is replaced by the two dedicated workflows in this repo. Keep it as evidence but take it out of the workflows folder so GitHub never runs it:

```bash
mkdir -p docs
git mv .github/workflows/pipeline.yaml docs/sam-generated-pipeline.reference.yaml 2>/dev/null \
  || mv .github/workflows/pipeline.yaml docs/sam-generated-pipeline.reference.yaml
```

### 5. Fill in the bootstrap values

Copy the role ARNs and bucket names into the `env:` block of each workflow (dev values into `deploy-dev.yml`, prod values into `deploy-prod.yml`). They are in `.aws-sam/pipeline/pipelineconfig.toml`, or:

```bash
aws cloudformation describe-stacks --region eu-north-1 \
  --stack-name aws-sam-cli-managed-dev-pipeline-resources \
  --query "Stacks[0].Outputs" --output table
```

### 6. Deploy dev

```bash
git add .
git commit -m "feat: dynamodb table, sam pipeline config, dev and prod workflows"
git push -u origin develop
```

The Deploy DEV workflow runs and creates `orders-dev`.

### 7. Deploy prod

Open a pull request from `develop` to `main` and merge it. The Deploy PROD workflow runs and creates `orders-prod`.

## Verification in the console (CRUD)

Open DynamoDB in `eu-north-1`, then Tables, then `orders-dev`.

Check configuration on the table overview: capacity mode shows On-demand and table class shows DynamoDB Standard-IA. The Indexes tab lists both GSIs.

**Create.** Explore table items, Create item, switch to JSON view, turn off "View DynamoDB JSON", paste one object from `docs/sample-items.json`, Create item. Repeat for all five.

**Read by primary key.** Scan or query items, choose Query on the table, `OrderId` = `ORD-1001`, Run.

**Read through GSI 1.** Choose index `CustomerOrdersIndex`, `CustomerId` = `CUST-ALICE`. Optionally add a sort key condition such as `OrderDate` between `2026-10-01` and `2026-10-31`.

**Read through GSI 2.** Choose index `OrderStatusIndex`, `OrderStatus` = `PENDING`. Returns ORD-1001 and ORD-1003.

**Update.** Open ORD-1003, change `OrderStatus` to `SHIPPED`, save. Re-run the `PENDING` query: ORD-1003 is gone. Query `SHIPPED`: it appears. This shows the GSI being maintained automatically.

**Delete.** Select ORD-1005, Actions, Delete items, confirm.

## Rubric mapping

| Criterion | Where |
|---|---|
| DynamoDB defined in SAM template | `template.yaml` (`Transform: AWS::Serverless-2016-10-31`, resource `OrdersTable`) |
| On-Demand billing | `BillingMode: PAY_PER_REQUEST` |
| Non-standard table class | `TableClass: STANDARD_INFREQUENT_ACCESS` |
| Primary key, 2+ attributes, 2 GSIs | `KeySchema`, `AttributeDefinitions`, `GlobalSecondaryIndexes` |
| SAM pipeline configured | `.aws-sam/pipeline/pipelineconfig.toml`, the two bootstrap stacks, `docs/sam-generated-pipeline.reference.yaml` |
| GitHub Actions working | Actions tab run history for both workflows |
| Multi-environment | `orders-dev` and `orders-prod`, stacks `dynamodb-lab-dev` and `dynamodb-lab-prod` |
| Environment-scoped artifact buckets | Separate bucket per bootstrap stage, objects under `dynamodb-lab-dev/` and `dynamodb-lab-prod/` prefixes |
| Separate dev and prod pipelines | `deploy-dev.yml` and `deploy-prod.yml`, each with only its own roles, bucket and stack |

## Troubleshooting

**"Not authorized to perform sts:AssumeRoleWithWebIdentity".** The OIDC trust policy is tied to the branch given at bootstrap. Dev must run from `develop`, prod from `main`. Adding `environment:` to a job also breaks this, because GitHub then sends the environment name in the token instead of the branch. If you want a GitHub Environment with a required reviewer on prod, update the prod role trust policy to accept `repo:ishimweClaude12/DynamoDB_SAM:environment:production`.

**Stack stuck in ROLLBACK_COMPLETE.** Delete it (`sam delete --stack-name dynamodb-lab-dev --region eu-north-1`) and push again.

**Changing indexes later.** CloudFormation can only add or remove one GSI per stack update, and changing the key schema replaces the table, which fails when the table has a fixed name. Make one index change per commit.

## Teardown

```bash
sam delete --stack-name dynamodb-lab-dev --region eu-north-1 --no-prompts

aws dynamodb update-table --table-name orders-prod --region eu-north-1 --no-deletion-protection-enabled
sam delete --stack-name dynamodb-lab-prod --region eu-north-1 --no-prompts
```

The bootstrap buckets are versioned, so empty every object version in each before deleting the `aws-sam-cli-managed-<stage>-pipeline-resources` stacks.
