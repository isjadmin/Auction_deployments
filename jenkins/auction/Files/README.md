# Auction Multibranch Pipelines

This directory contains the Jenkins pipeline definitions used by the Auction project:

- `Jenkinsfile-backend` builds and deploys the changed Java services.
- `Jenkinsfile-frontend` builds and deploys the React website.

Create a separate Jenkins Multibranch Pipeline job for each file. A multibranch job uses one script path for all branches it discovers, so the backend and frontend must not share a job.

## Prerequisites

Before creating the jobs:

1. Make sure the Git repository Jenkins will scan contains these files at the repository-relative paths `auction/Files/Jenkinsfile-backend` and `auction/Files/Jenkinsfile-frontend`.
2. Add an online Jenkins inbound agent with the label `gcp-deploy`. The controller is configured with zero executors, and both pipelines request this agent label, so builds will remain queued without it. The image definition in `jenkins/agent/Dockerfile` includes Java, Node.js, Google Cloud CLI, and common build tools, but the Compose deployment does not start an agent.
3. In **Manage Jenkins > Credentials**, add the Google service-account JSON key as a **Secret file** credential with ID `gcp-deploy-sa`. Both pipelines refer to this exact ID.
4. Ensure the agent can access the repository, Google Cloud project, target Compute Engine VM, and deployment destinations. The frontend pipeline also needs `npm` and `curl`; the backend pipeline downloads Maven during its run.
5. If the source repository is private, add an appropriate GitHub credential in Jenkins and select it when configuring the branch source.

## Create the multibranch jobs

In Jenkins, create a **New Item**, enter the job name, select **Multibranch Pipeline**, then configure **Branch Sources** to use the GitHub organization/user and repository containing the application. Configure GitHub credentials if required, save, and use **Scan Multibranch Pipeline Now** to discover branches.

Create these two jobs, using the repository URL for the Auction application (not this deployment repository unless it also contains the application source):

| Job name           | Script path                          | Intended branch  |
| ------------------ | ------------------------------------ | ---------------- |
| `auction-backend`  | `auction/Files/Jenkinsfile-backend`  | `QA` (uppercase) |
| `auction-frontend` | `auction/Files/Jenkinsfile-frontend` | `qa` (lowercase) |

Set the **Script Path** under the job's **Build Configuration** to the value in the table. Configure branch discovery/filtering so the job builds only its intended QA branch where possible. Branch names are case-sensitive: the backend pipeline fails validation for any branch other than `QA`; the frontend marks non-`qa` builds `NOT_BUILT` and limits its build and deployment stages to `qa`.

## Branch discovery and webhook

Use **Scan Multibranch Pipeline Now** after creating or changing a job to discover branches and create the corresponding child jobs. Configure a GitHub webhook to reach the Jenkins controller for push events if automatic scanning/build triggering is needed; the controller must be reachable from GitHub. The frontend Jenkinsfile declares a `githubPush()` trigger. The backend Jenkinsfile has no explicit push trigger, so ensure branch indexing is triggered by the selected GitHub Branch Source integration or configure a periodic scan. A manual branch scan is always available from the multibranch job.

## What the pipelines do

### Backend (`QA`)

The backend pipeline checks out the branch, compares the current commit with its parent, and looks for changes under `Auction_Master_Api/`, `Participation_Service/`, and `Bidding_Service/`. It builds and deploys only the services whose directories changed. Changes outside these service directories do not trigger a service deployment. Maven builds run with tests skipped (`-DskipTests`). The pipeline deploys to a Google Compute Engine VM and checks service health endpoints.

The first commit has no parent to compare, so the pipeline reports no service changes and does not automatically build or deploy services in that run.

### Frontend (`qa`)

The frontend pipeline runs `npm ci` and `npm run build`, backs up the existing website on the VM, copies the generated `dist/` output into the website directory, then verifies the site responds with HTTP 200. If post-deployment verification fails, it attempts to restore the latest backup.

## Run and monitor

After the initial scan, open the desired branch child job and use **Build Now** for a manual run. For automatic runs, push to the configured branch after confirming the webhook/indexing setup. Open a build to inspect stage status and console output. Builds are timestamped and concurrent runs are disabled in both pipelines.

For the backend, review the detected changed files and service decisions in the console output. For the frontend, check the build, backup, deployment, and post-deployment verification stages. Do not assume a green build means every project path was deployed: backend deployments are directory-change based, and frontend deployment stages run only on `qa`.

## Deployment target warning

The Jenkinsfiles currently use the GCP project `theshortlistd-production` and instance name `shortlistd-prod-march-2026`, even though the pipelines are gated on QA branch names. The frontend also verifies `http://nivasabid.theshortlistd.org/`. Confirm these are the intended QA targets before enabling builds or webhooks; otherwise update the pipeline target values before running a deployment.
