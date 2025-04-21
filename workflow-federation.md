Note: The gsutil command will not use the credentials exported by this GitHub Action. Customers should use gcloud storage instead.

Workload Identity Federation is recommended over Service Account Keys as it obviates the need to export a long-lived credential and establishes a trust delegation relationship between a particular GitHub Actions workflow invocation and permissions on Google Cloud. 

# Prerequisites: 
- Run the actions/checkout@v4 step before this action. Omitting the checkout step or putting it after auth will caus future steps to be unable to authneticate
- To create binaries, containers, pull requests, or other releases, add the following to your .gitignore, .dockerignore and similar files to prevent accidentally committing credentials to your release artifact: 

# Ignore generated credentials from google-github-actions/auth
gha-creds-*.json

This action runs using Node 20. Use a runner version that supports this version of Node or newer. 

STEPS: Complete the GCLOUD COMMANDS and then update  the WORKFLOW AUTH FILE 

- To get the permissions that you need to configure Workload Identity Federation, ask your administrator to grant you the following IAM roles on the project:

- roles/iam.workloadIdentityPoolAdmin
- roles/iam.serviceAccountAdmin


1. Create a Workload Identity Pool

# TODO: replace ${PROJECT_ID} with your value below.

gcloud iam workload-identity-pools create "github" \
  --project="workload-federation-45751}" \
  --location="global" \
  --display-name="GitHub Actions Pool"

2. Get the full ID of the Workload Identity Pool:

# TODO: replace ${PROJECT_ID} with your value below.

gcloud iam workload-identity-pools describe "github" \
  --project="workload-federation-457519" \
  --location="global" \
  --format="value(name)"

Value:
projects/258074141873/locations/global/workloadIdentityPools/github

3. Create a Workload Identity Provider in that pool:

🛑 CAUTION! Always add an Attribute Condition to restrict entry into the Workload Identity Pool. You can further restrict access in IAM Bindings, but always add a basic condition that restricts admission into the pool. A good default option is to restrict admission based on your GitHub organization as demonstrated below. Please see the security considerations for more details.

# TODO: replace ${PROJECT_ID} and ${GITHUB_ORG} with your values below.

gcloud iam workload-identity-pools providers create-oidc "my-repo" \
  --project="workload-federation-457519" \
  --location="global" \
  --workload-identity-pool="github" \
  --display-name="My GitHub repo Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository_owner == 'indopark-tech'" \
  --issuer-uri="https://token.actions.githubusercontent.com"

❗️ IMPORTANT You must map any claims in the incoming token to attributes before you can assert on those attributes in a CEL expression or IAM policy!

4. # TODO: replace ${PROJECT_ID} with your value below.

gcloud iam workload-identity-pools providers describe "my-repo" \
  --project="workload-federation-457519" \
  --location="global" \
  --workload-identity-pool="github" \
  --format="value(name)"

Use this value as the workload_identity_provider value in the GitHub Actions YAML:
projects/258074141873/locations/global/workloadIdentityPools/github/providers/my-repo

- uses: 'google-github-actions/auth@v2'
  with:
    project_id: 'my-project'
    workload_identity_provider: '...' # "projects/123456789/locations/global/workloadIdentityPools/github/providers/my-repo"

❗️ IMPORTANT The project_id input is optional, but may be required by downstream authentication systems such as the gcloud CLI. Unfortunately we cannot extract the project ID from the Workload Identity Provider, since it requires the project number.

It is technically possible to convert a project number into a project ID, but it requires permissions to call Cloud Resource Manager, and we cannot guarantee that the Workload Identity Pool has those permissions.

5. As needed, allow authentications from the Workload Identity Pool to Google Cloud resources. These can be any Google Cloud resources that support federated ID tokens, and it can be done after the GitHub Action is configured.

The following example shows granting access from a GitHub Action in a specific repository a secret in Google Secret Manager.


# TODO: replace ${PROJECT_ID}, ${WORKLOAD_IDENTITY_POOL_ID}, and ${REPO}
# with your values below.
#
# ${REPO} is the full repo name including the parent GitHub organization,
# such as "my-org/my-repo".
#
# ${WORKLOAD_IDENTITY_POOL_ID} is the full pool id, such as
# "projects/123456789/locations/global/workloadIdentityPools/github".

gcloud secrets add-iam-policy-binding "my-secret" \
  --project="${PROJECT_ID}" \
  --role="roles/secretmanager.secretAccessor" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${REPO}"


# Important outputs:
- Workload Identity Pool ID: 
- workload_identity_provider resource name:
