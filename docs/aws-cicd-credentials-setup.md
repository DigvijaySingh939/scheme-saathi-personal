# AWS EC2 CI/CD setup

This guide matches the current [CI/CD workflow](../.github/workflows/cicd.yaml).
GitHub Actions runs tests on GitHub-hosted Linux, pushes the Docker image to
Docker Hub, then deploys it to an AWS EC2 instance using a self-hosted GitHub
Actions runner installed on that instance. It does not deploy to ECS and does
not use an AWS access key or OIDC role.

Never put credentials in the repository, workflow YAML, Docker image, or task
logs. Store CI credentials in GitHub Environment secrets. Store durable
application data in the Docker volumes created on the EC2 host.

## 1. Prepare the EC2 instance

1. Launch a Linux EC2 instance with enough memory and disk for the app, its
   dependencies, and ChromaDB. Select an AWS Region and note the instance's
   public DNS name or configure a domain name.
2. Install Docker Engine and make sure Docker starts automatically on reboot.
3. Configure the GitHub Actions runner as a service on this instance. In
   **GitHub repository → Settings → Actions → Runners → New self-hosted
   runner**, follow GitHub's current Linux installation instructions.
4. Add the custom runner label `scheme-saathi-deploy`. The workflow requires
   the labels `self-hosted`, `linux`, `x64`, and `scheme-saathi-deploy`.
5. Run the runner as a dedicated Linux account that can access the Docker
   daemon. Treat this runner as a privileged deployment machine: only trusted
   maintainers should be allowed to change the protected `main` branch or
   approve the `production` environment.
6. Configure inbound network access through a security group and preferably a
   TLS reverse proxy or load balancer. Expose HTTPS to users; do not make
   container port `8501` publicly accessible. The app container listens on
   host port `8501`.

The workflow creates Docker volumes named
`scheme-saathi-documents`, `scheme-saathi-memory`, and
`scheme-saathi-checkpoints`. These survive container replacement on the same
EC2 instance. They do not survive deletion or loss of the EC2 instance unless
you separately back them up.

## 2. Create Docker Hub repositories and tokens

1. Create a Docker Hub repository for Scheme Saathi. Record its full name as
   `namespace/image`, for example `your-dockerhub-user/scheme-saathi`.
2. Create a Docker Hub access token with permission to push to that repository.
   This token is used by the GitHub-hosted build job.
3. Create a second, read-only Docker Hub token for the EC2 runner to pull the
   built image.
4. Do not use your Docker Hub account password. Keep both tokens private.

The separate tokens limit the impact of a leaked pull credential: it cannot
publish a replacement image.

## 3. Create the GitHub production environment

In the repository, open **Settings → Environments → New environment** and
create an environment named exactly `production`. Add a deployment branch
rule that allows `main`; optionally require a reviewer before each deployment.

In **production → Environment secrets**, add:

| Secret | What to enter |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub account or organization username |
| `DOCKERHUB_TOKEN` | Docker Hub token that can push the image |
| `DOCKERHUB_PULL_TOKEN` | Separate read-only Docker Hub token for the EC2 runner |
| `OPENROUTER_API_KEY` | OpenRouter API key, if using OpenRouter |
| `OPENAI_API_KEY` | OpenAI API key, if using OpenAI; at least one model-provider key is required |
| `MOCK_PORTAL_AUTH_SECRET` | A strong, randomly generated secret used by the mock application portal |

Use **Environment variables** for non-secret configuration:

| Variable | What to enter |
|---|---|
| `DOCKERHUB_REPOSITORY` | Full Docker Hub image name in `namespace/image` format |
| `SUPERVISOR_MODEL` | Model ID for the supervisor |
| `SPECIALIST_MODEL` | Model ID for intake, eligibility, and other specialists |
| `OPENAI_BASE_URL` | Optional OpenAI-compatible API URL; leave empty for the standard OpenAI API |
| `OPENROUTER_BASE_URL` | Optional; normally `https://openrouter.ai/api/v1` |
| `AUTH0_DOMAIN` | Auth0 tenant host name, without `https://` or a path |
| `AUTH0_CLIENT_ID` | Client ID for the Auth0 application |
| `AUTH0_AUDIENCE` | Audience identifier for the protected API |
| `AUTH0_REDIRECT_URI` | Exact public HTTPS application URL, for example `https://app.example.org` |
| `AUTH0_HOUSEHOLD_CLAIM` | Exact custom claim name that Auth0 adds to citizen access tokens |
| `AUTH0_ROLE_CLAIM` | Optional custom claim name for CSC operator roles; leave empty if unused |

The workflow checks required settings before deployment. It needs both Docker
Hub tokens, the image name, model IDs, the mock portal secret, the Auth0
settings, and at least one model-provider API key. It reports missing setting
names only; it does not print their values. Docker Hub login and the health
check provide later validation that credentials and the running container
work.

The Auth0 redirect URI must also be listed in the Auth0 application's allowed
callback URLs. Use the actual public URL, not `localhost`.

## 4. Set up Auth0 and model provider

### Auth0

1. In Auth0, create/configure the application for the UI's Authorization Code
   flow with PKCE.
2. Create or configure the API and its audience.
3. Configure the citizen household claim and set its exact name in
   `AUTH0_HOUSEHOLD_CLAIM`. Configure an Auth0 Action to issue this claim.
4. If CSC operator accounts are used, configure the role claim and set its name
   in `AUTH0_ROLE_CLAIM`.
5. Add the deployed HTTPS URL to the application's allowed callback URLs and
   set the same URL in `AUTH0_REDIRECT_URI`.

The app does not require an Auth0 client secret. Do not use a client secret as
a substitute for the client ID.

### Model provider

Choose at least one:

- **OpenRouter:** add `OPENROUTER_API_KEY` as a GitHub secret and set model IDs
  such as the provider's expected `openai/...` ID in `SUPERVISOR_MODEL` and
  `SPECIALIST_MODEL`.
- **OpenAI:** add `OPENAI_API_KEY` as a GitHub secret and set supported OpenAI
  model IDs in `SUPERVISOR_MODEL` and `SPECIALIST_MODEL`.

If both keys are set, the current model factory prefers OpenAI for agent model
calls. The supervisor's intent/language classifier has an OpenRouter-specific
optional fast path; without an OpenRouter key it uses its built-in fallback.
Use model IDs supported by the selected provider.

## 5. What deployment does

On a push to `main`, or a manual run started from `main`, the workflow:

1. Runs the project's tests with Python 3.12 and `python -m pytest -q`.
2. Checks Docker Hub push settings and logs in from a GitHub-hosted runner.
3. Builds the root `Dockerfile` and pushes `main` and commit-SHA tags.
4. Waits for the self-hosted EC2 runner to log in with the read-only token and
   pull the image for that exact commit.
5. Creates the persistent Docker volumes if they do not already exist.
6. Replaces the `scheme-saathi` container, injects the configured app
   environment, and binds host port `8501`.
7. Checks `http://localhost:8501/_stcore/health` inside the container and
   prints recent logs if the workflow finishes.

The workflow concurrency group serializes runs for the same branch and does
not cancel an in-progress run. This prevents two main-branch deployments from
running at the same time. Pull-request runs use a different branch reference.

The deployment uses a single EC2 instance and replaces its running container;
brief downtime can occur during replacement. The image is tagged with the
commit SHA so a previous version can be pulled and run for rollback.

## 6. Run and troubleshoot

1. Confirm the self-hosted runner is online and has the required custom label.
2. Confirm the `production` environment, secrets, and variables are present.
3. Confirm Docker is running and the runner account can run Docker commands.
4. Push a small change to `main`, or open **Actions → Scheme Saathi AWS
   CI/CD → Run workflow** and select `main`.
5. Confirm tests pass, Docker Hub image upload succeeds, and the deployment
   health check passes.
6. Open the public HTTPS address and check the EC2 firewall, reverse proxy, and
   container logs if the app is unreachable.

To inspect deployment status on EC2, use:

```bash
docker ps --filter name=scheme-saathi
docker logs --tail 100 scheme-saathi
docker volume ls
```

Common problems:

- **Runner is waiting/offline:** check that the runner service is running and
  has the `scheme-saathi-deploy` label.
- **Docker permission denied:** add the runner account to the Docker group,
  restart its login/service, and verify `docker info` as that account.
- **Missing settings:** add the named value to the `production` environment,
  not to a local `.env` file.
- **Docker Hub denied:** verify the username, push token, pull token, and
  repository permissions.
- **Container unhealthy:** inspect `docker logs`; check model credentials,
  Auth0 URL/claim settings, and available EC2 memory/disk.
- **External URL unavailable:** check reverse proxy and security group
  configuration. Only the proxy/load balancer should be publicly reachable.

## 7. Security and persistence notes

- Never commit `.env`, API keys, Docker Hub tokens, or real personal data.
- Restrict who can edit `main`, the production environment, and deployment
  workflow files. A self-hosted runner can access the host's Docker daemon.
- Avoid printing secrets in commands or logs. GitHub masks registered secret
  values, but logs should still not contain them.
- Uploaded files are mounted in a persistent Docker volume. The current
  application stores them locally and deletes them after 30 days; plan secure
  backups and host-disk protection before using real documents.
- Memory and workflow checkpoints are also stored in Docker volumes. Back up
  and protect them according to the project's consent, expiry, and deletion
  policy.
- Docker volumes are tied to the EC2 host. Snapshot or back up the host's
  persistent data using an approved encrypted AWS backup design.
