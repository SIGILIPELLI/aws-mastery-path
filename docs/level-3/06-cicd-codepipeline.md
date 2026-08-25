# CI/CD (CodePipeline, CodeBuild, CodeDeploy)

Every module so far deployed manually via CLI or IaC apply. A CI/CD
pipeline automates that: a commit to your source repo triggers a build,
tests run, and (if they pass) the change deploys — with no human typing
`aws deploy` by hand. AWS's native pipeline is three services working
together: **CodePipeline** (orchestration), **CodeBuild** (build/test),
and **CodeDeploy** (deployment).

## The three pieces

| Service | Role |
|---|---|
| CodePipeline | Defines stages (Source → Build → Deploy) and moves an artifact through them |
| CodeBuild | Runs your build/test commands in a managed container, per a `buildspec.yml` |
| CodeDeploy | Performs the actual deployment (to EC2, ECS, or Lambda) with a defined strategy |

## buildspec.yml

CodeBuild reads this from your repo root:

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 20
  pre_build:
    commands:
      - npm ci
  build:
    commands:
      - npm test
      - npm run build
      - docker build -t training-app .
      - docker tag training-app:latest $ECR_REPO_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO_URI
      - docker push $ECR_REPO_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
artifacts:
  files:
    - imagedefinitions.json
```

```bash
aws codebuild create-project \
  --name training-app-build \
  --source type=CODECOMMIT,location=https://git-codecommit.us-east-1.amazonaws.com/v1/repos/training-app \
  --artifacts type=NO_ARTIFACTS \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/standard:7.0,computeType=BUILD_GENERAL1_SMALL \
  --service-role arn:aws:iam::123456789012:role/CodeBuildServiceRole
```

`CODEBUILD_RESOLVED_SOURCE_VERSION` is an environment variable CodeBuild
injects automatically — tagging images with the commit SHA (rather than
`latest`) is what makes rollbacks to a specific prior build possible.

## Wire up the pipeline

```json
{
  "pipeline": {
    "name": "training-app-pipeline",
    "roleArn": "arn:aws:iam::123456789012:role/CodePipelineServiceRole",
    "artifactStore": { "type": "S3", "location": "training-pipeline-artifacts" },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "Source", "actionTypeId": { "category": "Source", "owner": "AWS", "provider": "CodeStarSourceConnection", "version": "1" },
          "outputArtifacts": [{ "name": "SourceOutput" }],
          "configuration": { "ConnectionArn": "arn:aws:codeconnections:us-east-1:123456789012:connection/abc", "FullRepositoryId": "org/training-app", "BranchName": "main" }
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "Build", "actionTypeId": { "category": "Build", "owner": "AWS", "provider": "CodeBuild", "version": "1" },
          "inputArtifacts": [{ "name": "SourceOutput" }],
          "outputArtifacts": [{ "name": "BuildOutput" }],
          "configuration": { "ProjectName": "training-app-build" }
        }]
      },
      {
        "name": "Deploy",
        "actions": [{
          "name": "Deploy", "actionTypeId": { "category": "Deploy", "owner": "AWS", "provider": "ECS", "version": "1" },
          "inputArtifacts": [{ "name": "BuildOutput" }],
          "configuration": { "ClusterName": "training-cluster", "ServiceName": "training-app" }
        }]
      }
    ]
  }
}
```

```bash
aws codepipeline create-pipeline --cli-input-json file://pipeline.json
```

Each stage passes an **artifact** (a zip stored in S3) to the next, not
live state — the Build stage's `imagedefinitions.json` output tells the
ECS deploy action which image tag to roll out.

## Deployment strategies (CodeDeploy)

For ECS/Lambda, CodeDeploy supports **blue/green** deployments: it spins
up the new version alongside the old one, shifts traffic gradually (or
all at once), and can automatically roll back on a CloudWatch alarm.

```json
{
  "deploymentConfigName": "CodeDeployDefault.ECSLinear10PercentEvery1Minutes"
}
```

That config shifts 10% of traffic every minute — a full rollout takes
10 minutes, with automatic rollback if error-rate alarms fire during
the shift. `CodeDeployDefault.ECSAllAtOnce` skips the gradual shift
entirely, trading safety for speed.

## Gotchas

- **CodeBuild's service role needs explicit ECR push permissions and
  VPC config** if your build needs to reach resources inside a VPC
  (e.g., a private npm registry) — CodeBuild runs outside your VPC by
  default.
- **Pipeline stages run in the IAM context of the pipeline's role**, not
  the triggering user's — a pipeline that can deploy to production is a
  high-value target; scope its role tightly and consider requiring a
  manual approval action before the prod stage.
- **CodeStar connections require a one-time manual OAuth handshake in
  the console** — `create-connection` via CLI leaves the connection in
  `PENDING` status until a human completes the handshake.
- **Artifacts expire based on the S3 bucket's lifecycle policy** — a
  pipeline re-run months later can fail if the source artifact was
  already deleted; keep lifecycle rules generous or re-trigger from
  Source.
- **Blue/green rollback undoes traffic shifting, not data migrations**
  — if your deploy step also ran a database migration, CodeDeploy
  rolling back the ECS service does not roll back the schema change.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws codepipeline create-pipeline` | Define a new pipeline |
| `aws codepipeline start-pipeline-execution` | Trigger a run manually |
| `aws codebuild start-build` | Run a build outside a pipeline |
| `aws codebuild batch-get-builds` | Check build status/logs |
| `aws deploy create-deployment` | Trigger a CodeDeploy deployment directly |
| `aws codepipeline get-pipeline-state` | See which stage is currently running |

## Exercise

Write a `buildspec.yml` that runs `npm test`, fails the build if tests
fail (CodeBuild does this automatically on non-zero exit), and produces
an `imagedefinitions.json` artifact. Then sketch a three-stage
CodePipeline JSON (Source → Build → Deploy-to-ECS) referencing it.
