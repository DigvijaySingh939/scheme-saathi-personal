# Docker and CI/CD: how deployment works

## Short viva answer

“Docker packages Scheme Saathi and its runtime dependencies into an image.
GitHub Actions runs the tests, builds that image, pushes it to Docker Hub, and
then asks a self-hosted GitHub Actions runner on AWS EC2 to run that image.
The deployment uses separate Docker Hub push and pull tokens; the EC2 runner
does not need static AWS access keys for this deployment.”

## What Docker does

The [`Dockerfile`](../Dockerfile):

1. Starts from a small Python 3.12 Linux image.
2. Copies the application code, Streamlit settings, and data into the image.
3. Installs the project and its RAG dependencies.
4. Runs Streamlit as a non-root user on port `8501`.

The image is the packaged application. It does not contain `.env` credentials.
The deployment workflow passes runtime settings from the GitHub `production`
environment to the container; they are not baked into the image.

## What the GitHub Actions workflow does

The workflow is [`.github/workflows/cicd.yaml`](../.github/workflows/cicd.yaml).

1. **Trigger:** it runs for pushes to `main`, pull requests to `main`, or a
   manual run. README-only changes are ignored.
2. **Test:** on the GitHub runner, it installs Python 3.12 and project
   dependencies, then runs `python -m pytest -q`.
3. **Deploy gate:** the deploy job runs only after the test job passes and only
   for a push to `main` or a manual run started from `main`.
4. **Preflight:** the workflow checks that the Docker Hub and application
   settings are present. It checks presence, not whether all values are valid.
5. **Docker Hub:** the runner signs in using the Docker Hub username and
   access token, builds the Docker image, and pushes two tags:
   - the full Git commit SHA, for traceability and rollback;
   - `latest`, for convenience.
6. **EC2 deployment:** a self-hosted runner on EC2 logs into Docker Hub with a
   read-only token, pulls the SHA-tagged image, and replaces the running
   container.
7. **Health check:** the workflow checks Streamlit's health endpoint and
   prints recent logs.

## Preventing overlapping deployments

The workflow uses a concurrency group based on the Git reference with
`cancel-in-progress: false`. This prevents two runs for the same branch (in
particular, `main` deployments) from running at the same time, and does not
cancel a deployment already in progress.

## What must be configured

The secrets and variables are entered in the GitHub repository's
**Settings → Environments → production** section. AWS must already have the
EC2 instance, Docker, self-hosted runner, and network access configured. The
exact setup steps and names are in
[`aws-cicd-credentials-setup.md`](aws-cicd-credentials-setup.md).

The GitHub-hosted build job uses the Docker Hub push token. The EC2 runner uses
a separate read-only token to pull the image.

## After a run

Check that tests passed, the image was pushed, and the container health check
passed. Then open the deployed HTTPS address and inspect the container logs on
EC2 if there are startup errors.

The workflow mounts Docker volumes on the EC2 host for uploaded documents,
memory, and workflow checkpoints. Back these volumes up securely; they do not
survive loss of the EC2 instance by themselves.

## Files to mention

- Image recipe: [`Dockerfile`](../Dockerfile)
- Files excluded from the build context:
  [`.dockerignore`](../.dockerignore)
- Tests and deployment workflow:
  [`.github/workflows/cicd.yaml`](../.github/workflows/cicd.yaml)
- Credential setup: [`aws-cicd-credentials-setup.md`](aws-cicd-credentials-setup.md)
- Detailed AWS deployment instructions:
  [`docker-deployment-guide.md`](docker-deployment-guide.md)
