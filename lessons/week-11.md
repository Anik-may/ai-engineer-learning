# Week 11: CI/CD with GitHub Actions

General knowledge; check action versions and AWS docs before copying.

## Goal

Every push tests the code, runs a small evaluation, builds the image and
deploys it, with no manual steps.

## Concepts

- **CI** (continuous integration): on every change, lint, test and build.
- **CD** (continuous delivery or deployment): the tested build goes to an
  environment automatically.
- **Workflow → job → step**: a workflow file holds jobs; jobs hold steps.
- **Runner**: the machine that executes a job.
- **Secrets**: encrypted values available to workflows.
- **Environments**: named targets (staging, production) with their own
  secrets and optional manual approval.

## A pipeline for an AI service

```
push / PR
   |
   +-- lint + type check         (seconds)
   +-- unit tests                (seconds)
   +-- eval smoke set            (minutes, costs tokens)
   +-- build image
   +-- push to ECR               (main branch only)
   +-- deploy to staging         (main branch only)
   +-- full eval against staging (nightly or before release)
   +-- deploy to production      (manual approval)
```

## Workflow

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

permissions:
  id-token: write      # for OIDC login to AWS
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run pytest tests/unit -q

  evals:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --frozen
      - run: uv run pytest tests/evals -q
        env:
          LANGSMITH_API_KEY: ${{ secrets.LANGSMITH_API_KEY }}
          LANGSMITH_TRACING: "true"
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}

  deploy:
    if: github.ref == 'refs/heads/main'
    needs: [test, evals]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ap-southeast-1
      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr
      - run: |
          IMAGE=${{ steps.ecr.outputs.registry }}/rag-api:${{ github.sha }}
          docker build -t $IMAGE .
          docker push $IMAGE
      - run: |
          aws ecs update-service --cluster rag --service rag-api --force-new-deployment
```

## Log in to AWS without stored keys

Use **OIDC**: GitHub proves its identity to AWS and assumes a role for the
length of the job. No long-lived access keys in GitHub secrets. The role's
trust policy should allow only your repository and branch.

## Good practice

| Practice | Why |
| --- | --- |
| Tag images with the commit SHA | You always know what is running; rollback is redeploying an old tag |
| Run evals on a small smoke set in PRs | Fast and cheap |
| Run the full set nightly | Catches slow regressions |
| Fail the build on a score threshold | Quality gate |
| Cache dependencies | Faster runs |
| Protect `main`; require checks to pass | No untested code reaches production |
| Manual approval for production | A human sees the eval report first |
| Separate staging and production secrets | Limits blast radius |

## Rolling back

Redeploy the previous image tag. Practise it once so you know it works.
Prompts stored in LangSmith or Langfuse roll back by moving a label, with no
deploy.

## Do

1. Add lint and unit-test jobs to project 1.
2. Add the eval smoke job with a score threshold.
3. Set up OIDC between GitHub and AWS with a least-privilege deploy role.
4. Build and push the image tagged with the commit SHA.
5. Break a test on purpose; confirm the deploy does not run.
6. Add a status badge to the README.

## Checkpoint

- [ ] A push to `main` produces a new image in ECR automatically.
- [ ] A failing test or eval blocks the deploy.
- [ ] No AWS keys are stored in GitHub.

## Interview questions

1. What is different about CI for an LLM application?
2. How do you authenticate GitHub Actions to AWS safely?
3. How do you roll back a bad deploy? A bad prompt?
4. How do you keep evaluation cost under control in CI?
