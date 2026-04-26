# Python Weather App CI Pipeline

This project is a Flask-based weather application packaged as a Docker image and validated through a GitHub Actions CI pipeline. The workflow is defined in `.github/workflows/ci.yml` and is designed to catch code quality issues, detect secrets, scan dependencies and container images, publish the Docker image, and verify that the container starts correctly.

## Repository Context

```text
python_app/
├── .github/workflows/ci.yml   # GitHub Actions pipeline
├── app.py                     # Flask application
├── Dockerfile                 # Container build definition
├── requirements.txt           # Python dependencies
├── templates/                 # HTML templates
├── static/                    # Static assets
└── README.md                  # Project documentation
```

## Workflow Trigger

The CI workflow runs on:

- pushes to `main`
- pull requests targeting `main`

This means every change proposed for the main branch is checked before merge, and every direct update to `main` is validated again.

## Pipeline Overview

The workflow contains five jobs that run in sequence:

1. `lint`
2. `secret-scan`
3. `dependency-scan`
4. `build_scan_and_push`
5. `smoke-test`

Two jobs start immediately in parallel: `lint` and `secret-scan`. After both succeed, the workflow continues with dependency scanning, then image build and push, and finally a smoke test against the pushed container image.

## Step-by-Step Explanation

### 1. Static Code Analysis

Job: `lint`

Implementation details:

- Checks out the repository with `actions/checkout@v4`
- Installs Python `3.11` using `actions/setup-python@v5`
- Upgrades `pip`
- Installs `pylint`
- Runs `pylint app.py`
- Extracts the pylint score from the command output with `awk` and `cut`
- Fails the job if the score is below `7.0`

Purpose:

- Enforces a minimum code quality gate
- Prevents low-quality changes from continuing through the pipeline

### 2. Secret Scanning

Job: `secret-scan`

Implementation details:

- Checks out the full repository history using `fetch-depth: 0`
- Runs `trufflesecurity/trufflehog@main`
- Uses `--results=verified,unknown` so both confirmed and suspicious findings are reported

Purpose:

- Detects hardcoded credentials or leaked secrets before build and release
- Uses full git history because secrets may exist in previous commits, not only in the latest file state

### 3. Dependency and Configuration Scan

Job: `dependency-scan`

Dependencies:

- waits for both `lint` and `secret-scan` through `needs: [secret-scan, lint]`

Implementation details:

- Checks out the code again in a fresh runner
- Runs `aquasecurity/trivy-action` against the repository filesystem
- Uses `scan-type: fs` and `scan-ref: '.'`
- Enables `vuln,config` scanners
- Fails the job with `exit-code: '1'` when `CRITICAL` findings are present

Purpose:

- Scans project files for vulnerable dependencies
- Scans configuration and Docker-related content for high-risk misconfigurations

### 4. Build, Scan, and Push Docker Image

Job: `build_scan_and_push`

Dependencies:

- waits for `dependency-scan`

Implementation details:

- Builds the image locally with:
  - `docker build -t helentam93/python_app:latest .`
- Scans the built image with Trivy
- Logs in to Docker Hub using:
  - `DOCKER_USERNAME`
  - `DOCKER_PASSWORD`
- Builds the image again before push
- Pushes `helentam93/python_app:latest` to Docker Hub

Purpose:

- Verifies that the application builds successfully as a container
- Scans the final image for critical vulnerabilities
- Publishes the image only after earlier quality and security checks pass

Implementation note:

The workflow currently builds the Docker image twice in this job: once before the image scan and once again before pushing. That matches the current `ci.yml` implementation exactly.

### 5. Smoke Test

Job: `smoke-test`

Dependencies:

- waits for `build_scan_and_push`

Implementation details:

- Pulls `helentam93/python_app:latest`
- Starts the container with:
  - `docker run -d --name app -p 8000:8000 helentam93/python_app:latest`
- Waits briefly with `sleep 5`
- Verifies the container is running using `docker ps | grep app`
- Sends an HTTP request to `http://localhost:8000`
- If the request fails, prints container logs and exits with an error
- Stops and removes the container in the cleanup step

Purpose:

- Confirms that the image does not just build, but also starts correctly
- Performs a basic runtime validation of the Flask application served by Gunicorn on port `8000`

## Why the Job Order Matters

The workflow is structured to fail early:

- `lint` catches maintainability issues
- `secret-scan` catches credential leaks
- `dependency-scan` blocks known critical vulnerabilities
- `build_scan_and_push` only runs after earlier checks succeed
- `smoke-test` validates the published image in a running container

This reduces wasted build and push activity when a change is already unsafe or broken earlier in the pipeline.

## Required GitHub Secrets

To push the Docker image successfully, the repository must define these GitHub Actions secrets:

- `DOCKER_USERNAME`
- `DOCKER_PASSWORD`

Without them, the Docker Hub login step will fail.

## Related Implementation Files

- `.github/workflows/ci.yml` contains the full CI definition
- `Dockerfile` defines how the production image is built
- `app.py` exposes the Flask app used during the smoke test
- `requirements.txt` lists the Python dependencies that are scanned by Trivy

## Summary

This CI pipeline implements a simple but practical delivery gate for the project:

- quality check with pylint
- secret detection with TruffleHog
- dependency and config scanning with Trivy
- Docker build and image vulnerability scan
- Docker Hub publish
- runtime smoke test

Together, these steps help ensure that changes merged into `main` are safer, cleaner, and deployable.
