 ----------- Project Name: "Flask weather app with Git Flow branching strategy" -----------

👉 Repo structure:

```text
weather-app/
  ├── static/                    # Static files
  ├── templates/                 # Jinja2 / Flask templates
  ├── app.py                     # Main application code
  ├── requirements.txt           # Python dependencies
  ├── Dockerfile                 # Docker image definition
  ├── Jenkinsfile                # CI/CD pipeline
  ├── jenkins-agent-pod.yaml     # Jenkins agent Pod definition to run a CI pipeline in the EKS Cluster 
  └── README.md                  # Project documentation
```

🌿 Git Flow Branching Strategy on GitHub (Long-lived branches)

🎯 Goals of this model:
- Provide a highly structured and systematic approach to managing the software development lifecycle.
- Isolate different stages of work, such as feature development, final release preparation, and urgent production fixes -> to ensure that the main branch remains consistently stable and production-ready

🌿 Long-lived branches (protected)

- These branches exist throughout the entire life of the project. They serve as the central pillars for code integration and production history.

| Branch Name | Primary Purpose                                                                                      | Stability                 |
| ----------- | ---------------------------------------------------------------------------------------------------- | ------------------------- |
| main        | Production-ready code. Every commit here represents a released version of the application.           | High (Strictly Stable)    |
| develop     | Integration branch. New features are merged here for testing before being moved to a release branch. | Moderate (Pre-production) |


🌿 Short-lived branches

- These branches are created for specific tasks and are deleted once their changes have been successfully merged into the long-lived branches.

| Branch Name Pattern | Parent Branch | Merges Into    | Purpose                                                                              |
| ------------------- | ------------- | -------------- | ------------------------------------------------------------------------------------ |
| feature/*           | develop       | develop        | Feature Development: Used for adding new functionality or refactoring.               |
| release/*           | develop       | main & develop | Release Preparation: Final bug fixes, documentation, and versioning before a launch. |
| hotfix/*            | main          | main & develop | Emergency Patches: Critical fixes for bugs currently live in production.             |


⌨️ Git Flow Command Guide

1. Working on a New Feature
```
# Start from develop
git checkout develop
git pull origin develop

# Create and switch to feature branch
git checkout -b feature/my-cool-feature

# ... work and commit ...

# Push to trigger CI pipeline
git push origin feature/my-cool-feature
```

2. Starting a Release
```# Branch off develop when it's ready for QA
git checkout develop
git checkout -b release/v1.1.0

# ... final polish/bug fixes ...

# Push to trigger Staging deployment
git push origin release/v1.1.0
```

3. Handling an Emergency (Hotfix)
```
# Branch off main to fix a production bug
git checkout main
git checkout -b hotfix/critical-patch

# ... fix the bug ...

# Push to trigger Prod deployment
git push origin hotfix/critical-patch
```

🔐 Apply Branch Protection Rules (CRITICAL):
- Go to your repository on GitHub.
- Click Settings → Rules → Add ruleset.

🌿 Protect main (production)
- Ruleset name: main
- Bypass list: "Do not allow bypassing the above settings"
- Branch name pattern: main
- Enable:
  - ✅ Require a PR before merging -> Required approvals (at least 1, preferably 2-3)
  - ✅ Require status checks (can be empty for now)
  - ✅ Require linear commit history 
  - ✅ Restrict deletions
  - ✅ Block force pushes
  - ✅ Require signed commits
  
🌿 Protect develop (integration branch)
- Ruleset name: develop
- Bypass list: Repo admin + dev team
- Branch name pattern: develop
- Enable:
  - ✅ Require PR before merging -> Require approvals 0
  - ✅ Restrict deletions
  - ✅ Require linear history 
  - ✅ Block force pushes



🔍 The Complete Weather-App DevSecOps Pipeline

    1. Clone App Repo: Get source code from the current branch.

    2. Security & Linting (Parallel):
        Static Analysis: pylint for code quality.
        Secret Scan: TruffleHog to find leaked credentials.

    3. Pre-Build Scan (Trivy Config & FS):
        Requirements: Scans requirements.txt for vulnerable libraries.
        Dockerfile: Scans Dockerfile for security misconfigurations (e.g., missing USER instructions).

    4. Kaniko Build: Build the image rootlessly in your Jenkins Pod.
       Push the image to Docker Hub repo (depending on the working branch).
       Post-Build Scan (Trivy): Scans the final image layers for OS-level vulnerabilities (CVEs) found in the base image.

    5. Integration / Reachability Test:
        Launch an Ephemeral Pod in Kubernetes.
        curl the internal IP to ensure the app actually starts.
        Cleanup (delete) the test pod.

    6. Push & Sign:
        Sign the image with Cosign (for main and hotfix/* and release/*).

    7. Update GitOps (ArgoCD Promotion):
        Update the specific values-*.yaml file in gitops-repo/weather-app/env/ using yq.
        Commit and push to trigger ArgoCD.
      
    8. Post Stage:
        Always check for test Pods if running and delete them.
        Send a message to Slack channel on success/failure of the pipeline.



🌿 Step-by-step branching flow based on the logic in the pipeline:

👉 1. The Feature Stage (feature/*)
- Trigger: Developer pushes code to a feature/xyz branch.
- Pipeline Behavior: 
  - Security & Linting: Jenkins sees feature/* branch and runs Pylint and TruffleHog.
  - Build & Test: It builds the image with Kaniko and runs a Trivy scan.
  - Smoke Test: It creates a temporary pod in Kubernetes to see if the app actually starts.
- GitOps: Pipeline explicitly skips the GitOps update stage because feature branches aren't meant for shared environments.

👉 2. The Integration Stage (develop)
- Trigger: A Pull Request is merged from feature/* into develop.
- Pipeline Behavior:
  - Security & Linting: Jenkins runs the same tests as the feature branch to ensure the merge didn't break anything.
  - Image Build: A new image is built with the tag ${env.BRANCH_NAME}-${env.BUILD_NUMBER} (e.g., develop-42).
- GitOps Trigger: Jenkins identifies env.BRANCH_NAME == "develop", sets envName = "dev", and updates values-staging.yaml.

- Result: ArgoCD sees the change and deploys to the Dev Environment.

👉 3. The Release Stage (release/*)
- Trigger: When you commit/push to release/*
- Pipeline Behavior:
  - The Check: "$BRANCH_NAME" =~ ^release/ evaluates to True.
  - The Result: The image is built, scanned, and signed. This allows your QA/Staging environment to verify that the image they are testing is authentic and hasn't been tampered with.
- GitOps Trigger: Jenkins identifies env.BRANCH_NAME == "release/", sets envName = "staging", and updates values-staging.yaml.

- Result: ArgoCD sees the change and deploys to the Staging Environment.

👉 4. The Production Stage (main)
- Trigger: The release/* branch is merged into main (after being approved).
- Pipeline Behavior:
  - The Check: "$BRANCH_NAME" == "main" evaluates to True.
  - Final Build: A final production image is re-build.
  - Signing: The image is re-signed as it is now official production code. This ensures your production cluster only runs 'fresh' images and not the ones that was tested in the previous stages.
- GitOps Trigger: Your script identifies env.BRANCH_NAME == "main", sets envName = "prod", and updates values-prod.yaml.

- Result: The code is live in Production.

- Event: realease/* branch is also merged into develop.
- Pipeline Behavior: This triggers the develop pipeline, ensuring the values-dev.yaml also gets updated.

👉 5. The Hotfix Stage (hotfix/*)
- Trigger: Creation & Push of the hotfix/*
- Pipeline Behavior: 
  - Security & Linting: Jenkins sees hotfix/* branch and runs Pylint and TruffleHog.
  - Build & Test: It builds the image with Kaniko and runs a Trivy scan.
  - Smoke Test: It creates a temporary pod in Kubernetes to see if the app actually starts.
  - Hotfix Verification Gate: The pipeline is paused and a notification message is send to Slack to verify the deployment to the staging environment.
  - Manual Approve: Go to the Jenkins UI and approve the deploy.
- GitOps Trigger: Jenkins identifies env.BRANCH_NAME == "hotfix/", sets envName = "hotfix-staging", and updates values-hotfix.yaml.
- Result: ArgoCD sees the change and deploys to the Hotfix Environment to run needed tests before merging hotfix/ to main.

- Trigger: The Hotfix PR is merged into main.
- Pipeline Behavior:
  - The Check: "$BRANCH_NAME" == "main" evaluates to True.
  - Final Build: A final production image is re-build.
  - Signing: The image is re-signed as it is now official production code. This ensures your production cluster only runs 'fresh' images and not the ones that was tested in the previous stages.
- GitOps Trigger: Your script identifies env.BRANCH_NAME == "main", sets envName = "prod", and updates values-prod.yaml.

- Result: The code is live in Production.

- The "Back-Port" (Crucial Step)
- Event: To prevent the bug from reappearing in the next release, the hotfix/* branch is also merged into develop.
- Pipeline Behavior: This triggers the develop pipeline, ensuring the values-dev.yaml also gets the fix.