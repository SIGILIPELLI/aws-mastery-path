# Project — Multi-Tier CI/CD Pipeline

This project combines modules 2 (EKS/ECS), 4 (IAM), and 6 (CodePipeline)
into one working release pipeline: a container-based app deployed to
ECS Fargate, built and shipped automatically on every push to `main`,
with a staging environment gating production.

## Architecture

```
GitHub (main branch)
   │  (CodeStar connection webhook)
   ▼
CodePipeline
   ├─ Source stage   → pulls commit
   ├─ Build stage     → CodeBuild: test, docker build, push to ECR
   ├─ Deploy-Staging  → ECS service in `staging` cluster
   ├─ Manual Approval → human clicks "Approve" in console/CLI
   └─ Deploy-Prod     → ECS service in `prod` cluster (blue/green via CodeDeploy)
```

Two ECS clusters (`staging`, `prod`) share one ECR repository, keyed by
commit SHA tags — the exact image tested in staging is what gets
promoted to prod, never a rebuild.

## Step 1 — ECR repository and ECS clusters

```bash
aws ecr create-repository --repository-name training-app

aws ecs create-cluster --cluster-name staging
aws ecs create-cluster --cluster-name prod
```

## Step 2 — IAM roles (least privilege per module 4)

```bash
# CodeBuild role: push to ECR, write logs, read source — nothing else
aws iam create-role --role-name CodeBuildServiceRole \
  --assume-role-policy-document file://trust-codebuild.json
aws iam put-role-policy --role-name CodeBuildServiceRole \
  --policy-name ecr-push --policy-document file://codebuild-ecr-policy.json

# CodePipeline role: start builds, deploy to ECS, read/write the artifact bucket
aws iam create-role --role-name CodePipelineServiceRole \
  --assume-role-policy-document file://trust-codepipeline.json

# ECS task execution role: pull from ECR, write CloudWatch Logs
aws iam create-role --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://trust-ecs-tasks.json
aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

Each role gets only the actions its stage needs — CodeBuild cannot call
`ecs:UpdateService`, and CodePipeline's role cannot push images
directly, matching the boundary discussed in module 4.

## Step 3 — buildspec.yml

```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-8)
  build:
    commands:
      - npm ci && npm test
      - docker build -t $ECR_URI:$IMAGE_TAG .
  post_build:
    commands:
      - docker push $ECR_URI:$IMAGE_TAG
      - printf '[{"name":"training-app","imageUri":"%s"}]' "$ECR_URI:$IMAGE_TAG" > imagedefinitions.json
artifacts:
  files: [imagedefinitions.json]
```

## Step 4 — pipeline with a manual approval gate

```bash
aws codepipeline create-pipeline --cli-input-json file://pipeline.json
```

The pipeline JSON follows module 6's structure with one addition — an
`Approval` action type between staging and prod:

```json
{
  "name": "PromoteToProd",
  "actions": [{
    "name": "ManualApproval",
    "actionTypeId": { "category": "Approval", "owner": "AWS", "provider": "Manual", "version": "1" },
    "configuration": { "CustomData": "Verify staging looks healthy before promoting." }
  }]
}
```

```bash
# Approving from the CLI once staging is verified:
aws codepipeline put-approval-result \
  --pipeline-name training-app-pipeline \
  --stage-name PromoteToProd \
  --action-name ManualApproval \
  --result summary="Staging verified",status=Approved \
  --token <token-from-get-pipeline-state>
```

## Step 5 — blue/green production deploy

```bash
aws deploy create-deployment-group \
  --application-name training-app \
  --deployment-group-name prod \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployECSRole \
  --deployment-config-name CodeDeployDefault.ECSLinear10PercentEvery1Minutes \
  --ecs-services clusterName=prod,serviceName=training-app \
  --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE,DEPLOYMENT_STOP_ON_ALARM
```

Auto-rollback on a CloudWatch alarm means a spike in 5xx errors during
the 10-minute traffic shift reverts to the prior task set automatically
— combining module 8's monitoring with the deployment itself.

## Verifying the pipeline

```bash
aws codepipeline start-pipeline-execution --name training-app-pipeline
aws codepipeline get-pipeline-state --name training-app-pipeline \
  --query 'stageStates[*].{stage:stageName,status:latestExecution.status}'
```

## Gotchas

- **The staging and prod ECS services must use distinct task
  definitions** even though they share an image — different
  environment variables (DB endpoints, feature flags) baked into
  separate task-def revisions, not the container image.
- **A manual approval action blocks the pipeline indefinitely** — set
  up an EventBridge rule on `CodePipeline Stage Execution State Change`
  for `Approval` actions to notify a Slack/SNS channel, or approvals
  get forgotten and staging drifts from prod for days.
- **Blue/green rollback reverts ECS task sets, not any database
  migration** run in the same pipeline — sequence schema migrations to
  be backward-compatible with both old and new code during the shift.
- **The ECR image tag scheme must be deterministic** (commit SHA, not
  `latest`) or you cannot prove which exact build is running in prod
  during an incident.

## Stretch goals

- Add a second manual approval before staging too, gated on automated
  integration test results rather than a human always clicking through.
- Fan out the Build stage to also run `terraform plan` against a
  staging Terraform stack (module 5) and post the plan output as a
  pipeline artifact for review.
- Add an EventBridge rule that triggers a Step Functions workflow
  (module 3) on pipeline failure to page an on-call engineer.
- Extend the deployment group to multiple regions using the DR patterns
  from module 7, so a prod promotion updates a warm-standby region too.
