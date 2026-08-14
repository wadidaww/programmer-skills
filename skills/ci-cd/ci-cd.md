# Skill: CI/CD Patterns

Patterns and best practices for continuous integration and continuous delivery pipelines.

---

## Pipeline Design Principles

- **Fast feedback**: the CI pipeline for a pull request should complete in < 10 minutes.
- **Fail fast**: put the cheapest checks (lint, format) first; expensive tests later.
- **Parallelise**: run independent jobs concurrently.
- **Idempotent**: running a pipeline twice produces the same result.
- **Auditable**: every deploy is traceable to a commit, PR, and engineer.
- **Reversible**: every deployment can be rolled back in < 5 minutes.

---

## Pipeline Stages Reference

### Stage 1: Lint & Format (< 1 min)
```yaml
lint:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: { node-version: '20', cache: 'npm' }
    - run: npm ci
    - run: npm run lint
    - run: npm run format:check
```

### Stage 2: Unit Tests (< 3 min)
```yaml
unit-test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: { node-version: '20', cache: 'npm' }
    - run: npm ci
    - run: npm test -- --coverage --reporter=junit
    - uses: actions/upload-artifact@v4
      with:
        name: test-results
        path: junit-results.xml
```

### Stage 3: Security Scan (< 2 min)
```yaml
security:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: fs
        severity: HIGH,CRITICAL
        exit-code: '1'
    - name: Run Gitleaks secret scan
      uses: gitleaks/gitleaks-action@v2
```

### Stage 4: Build & Push (merge to main only)
```yaml
build:
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  permissions:
    id-token: write
    contents: read
  steps:
    - uses: actions/checkout@v4
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}
        aws-region: us-east-1
    - uses: aws-actions/amazon-ecr-login@v2
    - name: Build and push
      run: |
        IMAGE_TAG="${{ github.sha }}"
        docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
```

---

## Deployment Strategies

### Blue/Green

```yaml
deploy-green:
  steps:
    - name: Deploy to green environment
      run: |
        # Update green task definition with new image
        aws ecs update-service --service order-api-green --force-new-deployment
    
    - name: Wait for green to be healthy
      run: aws ecs wait services-stable --services order-api-green
    
    - name: Run smoke tests against green
      run: npm run smoke-test -- --base-url $GREEN_URL
    
    - name: Shift traffic to green (ALB listener rule)
      run: |
        aws elbv2 modify-rule --rule-arn $LISTENER_RULE_ARN \
          --actions '[{"Type":"forward","TargetGroupArn":"$GREEN_TG_ARN"}]'
```

### Canary with Automated Rollback

```yaml
canary-deploy:
  steps:
    - name: Deploy canary (5% traffic)
      run: kubectl set image deployment/order-api order-api=$NEW_IMAGE
    
    - name: Monitor canary for 10 minutes
      run: |
        ERROR_RATE=$(./scripts/get-error-rate.sh --last=10m)
        if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
          echo "Error rate $ERROR_RATE exceeds threshold; rolling back"
          kubectl rollout undo deployment/order-api
          exit 1
        fi
    
    - name: Promote to 100%
      run: kubectl rollout status deployment/order-api
```

---

## Environment Promotion Pattern

```
feature-branch → PR pipeline (lint, test, security)
                    ↓ merge
main branch    → CI pipeline + deploy to DEV
                    ↓ auto-promote on green
staging        → integration tests + smoke tests
                    ↓ manual approval or auto on schedule
production     → canary → 100% + smoke tests
```

---

## Dependency Caching

```yaml
# Cache node_modules to speed up installs
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Cache pip packages
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
```

---

## Versioning & Tagging

```yaml
# Automatic semantic versioning via conventional commits
- uses: mathieudutour/github-tag-action@v6
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    default_bump: patch

# Tag Docker images with git SHA + semver tag
IMAGE_TAG="${{ github.sha }}"
SEMVER_TAG="${{ steps.tag.outputs.new_tag }}"
docker tag $IMAGE $REGISTRY/$REPO:$IMAGE_TAG
docker tag $IMAGE $REGISTRY/$REPO:$SEMVER_TAG
docker tag $IMAGE $REGISTRY/$REPO:latest
```

---

## Notifications

```yaml
- name: Notify Slack on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "❌ Pipeline failed on `${{ github.ref_name }}` — ${{ github.run_url }}"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Rollback Procedure

### Kubernetes
```bash
# Instant rollback to previous version
kubectl rollout undo deployment/order-api

# Rollback to a specific revision
kubectl rollout undo deployment/order-api --to-revision=3

# Check rollout history
kubectl rollout history deployment/order-api
```

### ECS
```bash
# Register previous task definition revision and update service
PREV_TASK_DEF=$(aws ecs describe-services --services order-api \
  --query 'services[0].deployments[-2].taskDefinition' --output text)
aws ecs update-service --service order-api --task-definition $PREV_TASK_DEF
```

---

## Pipeline Security

- Pin all action versions to full commit SHAs:
  ```yaml
  # BAD
  - uses: actions/checkout@v4
  
  # GOOD (pinned to SHA, immune to tag mutation)
  - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
  ```
- Use OIDC instead of long-lived access keys for cloud providers.
- Restrict `GITHUB_TOKEN` permissions to the minimum required:
  ```yaml
  permissions:
    contents: read
    id-token: write
  ```
- Never print secrets to logs (`set -x` in bash can leak them).
- Use `${{ secrets.NAME }}` for secrets; never interpolate secrets into shell commands directly — use env variables.
