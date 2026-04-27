# Reflection

## Architecture Decisions

I chose Path 1 and Path 2 for this capstone. Path 1 required deploying the full 
application stack locally on minikube, while Path 2 extended this to Azure Kubernetes 
Service using Terraform for infrastructure provisioning.

The decision to use AKS over other Azure compute options was driven by the requirement 
to demonstrate Kubernetes skills end to end. AKS abstracts the control plane management 
while still giving full access to kubectl and Kubernetes primitives like Deployments, 
Services, Secrets, and PersistentVolumeClaims.

For the CI/CD pipeline I chose GitHub Actions because the source code was already on 
GitHub, making it the natural fit with no additional tooling required. The three-job 
structure (build and push, terraform, deploy) mirrors real-world pipelines where 
infrastructure and application deployments are kept separate and ordered deliberately.

Terraform was used for all cloud infrastructure because it provides a declarative, 
repeatable way to provision resources. The empty backend block with -backend-config 
flags keeps connection details out of committed code entirely, satisfying the no 
hardcoding rule. State is stored remotely in Azure Blob Storage so it persists across 
pipeline runs and could be shared across a team.

Docker Hub was chosen as the container registry because it requires no additional Azure 
configuration, keeping the setup simpler for a learning context. Images are tagged with 
the git commit SHA on every pipeline run, ensuring traceability between what is running 
in the cluster and what was committed to source control.

## What Did Not Work

The biggest time sink was Git Bash on Windows mangling forward slashes in Azure CLI 
commands. Any path starting with /subscriptions/ was being converted to a Windows file 
path, causing role assignment commands to fail repeatedly. The fix was to switch to 
PowerShell for Azure CLI commands.

The Terraform VM size selection took multiple attempts. The subscription had strict 
quota limits (only 2 vCPUs available in northeurope) and many standard VM sizes were 
not permitted. The error messages from Azure were helpful but each attempt required a 
full pipeline run to discover, adding significant time.

The Grafana login was unexpectedly difficult. The password set during Helm installation 
was overridden by a Kubernetes secret, and the reset command did not persist because the 
environment variable took precedence. The fix required fetching the actual secret value 
directly from kubectl.

The .terraform provider directory was accidentally committed to git, which caused pushes 
to fail due to GitHub's 100MB file size limit. Removing it required rewriting git history.

## What I Would Improve

Given more time I would add a dedicated ingress controller with a proper domain name 
and TLS certificate rather than exposing services directly via LoadBalancer. This would 
make the application production-ready and demonstrate knowledge of cert-manager and 
nginx ingress.

I would also separate the Terraform state for different environments — having a dev and 
prod workspace — so that infrastructure changes could be tested before reaching 
production. Currently a failed terraform apply in the pipeline could leave the 
production environment in a broken state.

The Grafana dashboard would benefit from alerts configured via Alertmanager, so that 
if the backend goes down or task count drops to zero, an email or Slack notification 
is sent automatically. This closes the loop between observability and response.
