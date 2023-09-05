# databricks/run-notebook v0

# --- DISCLAIMER ---
This is not the original run-notebook action provided by Databricks, but instead a derived version (as allowed under the copyleft license) for testing of additional features. Unless you have a very good reason to use this action, you should use the original one provided by Databricks here: https://github.com/databricks/run-notebook

# Overview
Given a Databricks notebook and cluster specification, this Action runs the notebook as a one-time Databricks Job
run (docs: 
[AWS](https://docs.databricks.com/dev-tools/api/latest/jobs.html#operation/JobsRunsSubmit) |
[Azure](https://redocly.github.io/redoc/?url=https://docs.microsoft.com/azure/databricks/_static/api-refs/jobs-2.1-azure.yaml#operation/JobsRunsSubmit) |
[GCP](https://docs.gcp.databricks.com/dev-tools/api/latest/jobs.html#operation/JobsRunsSubmit)) and awaits its completion:

- optionally installing libraries on the cluster before running the notebook
- optionally configuring permissions on the notebook run (e.g. granting other users permission to view results)
- optionally triggering the Databricks job run with a timeout
- optionally using a Databricks job run name
- setting the notebook output,
  job run ID, and job run page URL as Action output
- failing if the Databricks job run fails

You can use this Action to trigger code execution on Databricks for CI (e.g. on pull requests) or CD (e.g. on pushes
to master).  

# Prerequisites
To use this Action, you need a Databricks REST API token to trigger notebook execution and await completion. The API
token must be associated with a principal with the following permissions:
* Cluster permissions ([AWS](https://docs.databricks.com/security/access-control/cluster-acl.html#types-of-permissions) |
[Azure](https://docs.microsoft.com/en-us/azure/databricks/security/access-control/cluster-acl#types-of-permissions) |
[GCP](https://docs.gcp.databricks.com/security/access-control/cluster-acl.html)): Allow unrestricted cluster creation entitlement,
if running the notebook against a new cluster (recommended), or "Can restart" permission, if running the notebook
against an existing cluster.
* Workspace permissions ([AWS](https://docs.databricks.com/security/access-control/workspace-acl.html#folder-permissions) |
[Azure](https://docs.microsoft.com/en-us/azure/databricks/security/access-control/workspace-acl#--folder-permissions) |
[GCP](https://docs.gcp.databricks.com/security/access-control/workspace-acl.html#folder-permissions)):
  * If supplying `local-notebook-path` with one of the `git-commit`, `git-tag`, or `git-branch` parameters, no workspace
    permissions are required. However, your principal must have Git integration configured ([AWS](https://docs.databricks.com/dev-tools/api/latest/gitcredentials.html#operation/create-git-credential) | [Azure](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/api/latest/gitcredentials) | [GCP](https://docs.gcp.databricks.com/dev-tools/api/latest/gitcredentials.html#operation/create-git-credential)). You can associate git credentials with your principal by creating a git credential entry using your principal's API token.
  * If supplying the `local-notebook-path` parameter, "Can manage" permissions on the directory specified by the
    `workspace-temp-dir` parameter (the `/tmp/databricks-github-actions` directory if `workspace-temp-dir` is unspecified).
  * If supplying the `workspace-notebook-path`  parameter, "Can read" permissions on the specified notebook.

We recommend that you store the Databricks REST API token in [GitHub Actions secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
to pass it into your GitHub Workflow. The following section lists recommended approaches for token creation by cloud.

Note: we recommend that you do not run this Action against workspaces with IP restrictions. GitHub-hosted action runners have a [wide range of IP addresses](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#ip-addresses), making it difficult to whitelist.

## AWS
For security reasons, we recommend creating and using a Databricks service principal API token. You can
[create a service principal](https://docs.databricks.com/dev-tools/api/latest/scim/scim-sp.html#create-service-principal),
grant the Service Principal
[token usage permissions](https://docs.databricks.com/administration-guide/access-control/tokens.html#control-who-can-use-or-create-tokens),
and [generate an API token](https://docs.databricks.com/dev-tools/api/latest/token-management.html#operation/create-obo-token) on its behalf.

## Azure
For security reasons, we recommend using a Databricks service principal AAD token.

### Create an Azure Service Principal
Here are two ways that you can create an Azure Service Principal. 

The first way is via the Azure Portal UI. See the [Azure Databricks documentation](https://docs.microsoft.com/en-us/azure/databricks/administration-guide/users-groups/service-principals#create-a-service-principal). Record the Application (client) Id, Directory (tenant) Id, and client secret values generated by the steps.

The second way is via the Azure CLI. You can follow the instructions below:
* Install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
* Run `az login` to authenticate with Azure
* Run `az ad sp create-for-rbac -n <your-service-principal-name> --sdk-auth --scopes /subscriptions/<azure-subscription-id>/resourceGroups/<resource-group-name> --sdk-auth --role contributor`,
  specifying the subscription and resource group of your Azure Databricks workspace, to create a service principal and client secret.

From the resulting JSON output, record the following values:
* `clientId`: this is the client or application Id of your service principal.
* `clientSecret`: this is the client service of your service princiapl.
* `tenantId`: this is the tenant or directory Id of your service principal.

After you create an Azure Service Principal, you should add it to your Azure Databricks workspace using the [SCIM API](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/api/latest/scim/scim-sp#add-service-principal). Use the client or application Id of your service principal as the `applicationId` of the service principal in the `add-service-principal` payload.

### Use the Service Principal in your GitHub Workflow
* Store your service principal credentials into your GitHub repository secrets. The Application (client) Id should be stored as `AZURE_SP_APPLICATION_ID`, Directory (tenant) Id as `AZURE_SP_TENANT_ID`, and client secret as `AZURE_SP_CLIENT_SECRET`.
* Add the following step at the start of your GitHub workflow.
  This will create a new AAD token for your Azure Service Principal and save its value in the `DATABRICKS_TOKEN`
  environment variable for use in subsequent steps.

  ```yaml
  - name: Generate AAD Token
    run: |
      echo "DATABRICKS_TOKEN=$(curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' \
        https://login.microsoftonline.com/${{ secrets.AZURE_SP_TENANT_ID }}/oauth2/v2.0/token \
        -d 'client_id=${{ secrets.AZURE_SP_APPLICATION_ID }}' \
        -d 'grant_type=client_credentials' \
        -d 'scope=2ff814a6-3304-4ab8-85cb-cd0e6f879c1d%2F.default' \
        -d 'client_secret=${{ secrets.AZURE_SP_CLIENT_SECRET }}' |  jq -r  '.access_token')" >> $GITHUB_ENV
  ```
**Notes**:
  * The generated Azure token has a default life span of **60 minutes**.
  If you expect your Databricks notebook to take longer than 60 minutes to finish executing, then you must create a [token lifetime policy](https://docs.microsoft.com/en-us/azure/active-directory/develop/configure-token-lifetimes)
  and attach it to your service principal.
  * The generated Azure token will work across all workspaces that the Azure Service Principal is added to. You do not need to generate a token for each workspace.

## GCP
For security reasons, we recommend inviting a service user to your Databricks workspace and using their API token.
You can invite a [service user to your workspace](https://docs.gcp.databricks.com/administration-guide/users-groups/users.html#add-a-user),
log into the workspace as the service user, and [create a personal access token](https://docs.gcp.databricks.com/dev-tools/api/latest/authentication.html) 
to pass into your GitHub Workflow.
  
# Usage

See [action.yml](action.yml) for the latest interface and docs.

### (Recommended) Run notebook within a temporary checkout of the current Repo
The workflow below runs a notebook as a one-time job within a temporary repo checkout, enabled by
specifying the  `git-commit`, `git-branch`, or `git-tag` parameter. You can use this to run notebooks that
depend on other notebooks or files (e.g. Python modules in `.py` files) within the same repo.

```yaml
name: Run a notebook within its repo on PRs

on:
  pull_request

env:
  DATABRICKS_HOST: https://adb-XXXX.XX.azuredatabricks.net

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checks out the repo
        uses: actions/checkout@v2
      # The step below does the following:
      # 1. Sends a POST request to generate an Azure Active Directory token for an Azure service principal
      # 2. Parses the token from the request response and then saves that in as DATABRICKS_TOKEN in the
      # GitHub enviornment.
      # Note: if the API request fails, the request response json will not have an "access_token" field and
      # the DATABRICKS_TOKEN env variable will be empty.
      - name: Generate and save AAD Token
        run: |
          echo "DATABRICKS_TOKEN=$(curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' \
            https://login.microsoftonline.com/${{ secrets.AZURE_SP_TENANT_ID }}/oauth2/v2.0/token \
            -d 'client_id=${{ secrets.AZURE_SP_APPLICATION_ID }}' \
            -d 'grant_type=client_credentials' \
            -d 'scope=2ff814a6-3304-4ab8-85cb-cd0e6f879c1d%2F.default' \
            -d 'client_secret=${{ secrets.AZURE_SP_CLIENT_SECRET }}' |  jq -r  '.access_token')" >> $GITHUB_ENV
      - name: Trigger model training notebook from PR branch
        uses: databricks/run-notebook@v0
        with:
          local-notebook-path: notebooks/deployments/MainNotebook
          # If the current workflow is triggered from a PR,
          # run notebook code from the PR's head commit, otherwise use github.sha.
          git-commit: ${{ github.event.pull_request.head.sha || github.sha }}
          # The cluster JSON below is for Azure Databricks. On AWS and GCP, set
          # node_type_id to an appropriate node type, e.g. "i3.xlarge" for
          # AWS or "n1-highmem-4" for GCP
          new-cluster-json: >
            {
              "num_workers": 1,
              "spark_version": "11.3.x-scala2.12",
              "node_type_id": "Standard_D3_v2"
            }
          # Grant all users view permission on the notebook results
          access-control-list-json: >
            [
              {
                "group_name": "users",
                "permission_level": "CAN_VIEW"
              }
            ]
```

### Run a self-contained notebook
The workflow below runs a self-contained notebook as a one-time job.

Python library dependencies are declared in the notebook itself using
notebook-scoped libraries
([AWS](https://docs.databricks.com/libraries/notebooks-python-libraries.html) | 
[Azure](https://docs.microsoft.com/en-us/azure/databricks/libraries/notebooks-python-libraries) | 
[GCP](https://docs.gcp.databricks.com/libraries/notebooks-python-libraries.html)) 
 
```yaml
name: Run a notebook in the current repo on PRs

on:
  pull_request

env:
  DATABRICKS_HOST: https://adb-XXXX.XX.azuredatabricks.net

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v2
      # The step below does the following:
      # 1. Sends a POST request to generate an Azure Active Directory token for an Azure service principal
      # 2. Parses the token from the request response and then saves that in as DATABRICKS_TOKEN in the
      # GitHub enviornment.
      # Note: if the API request fails, the request response json will not have an "access_token" field and
      # the DATABRICKS_TOKEN env variable will be empty.
      - name: Generate and save AAD Token
        run: |
          echo "DATABRICKS_TOKEN=$(curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' \
            https://login.microsoftonline.com/${{ secrets.AZURE_SP_TENANT_ID }}/oauth2/v2.0/token \
            -d 'client_id=${{ secrets.AZURE_SP_APPLICATION_ID }}' \
            -d 'grant_type=client_credentials' \
            -d 'scope=2ff814a6-3304-4ab8-85cb-cd0e6f879c1d%2F.default' \
            -d 'client_secret=${{ secrets.AZURE_SP_CLIENT_SECRET }}' |  jq -r  '.access_token')" >> $GITHUB_ENV
      - name: Trigger notebook from PR branch
        uses: databricks/run-notebook@v0
        with:
          local-notebook-path: notebooks/MainNotebook.py
          # Alternatively, specify an existing-cluster-id to run against an existing cluster.
          # The cluster JSON below is for Azure Databricks. On AWS and GCP, set
          # node_type_id to an appropriate node type, e.g. "i3.xlarge" for
          # AWS or "n1-highmem-4" for GCP
          new-cluster-json: >
            {
              "num_workers": 1,
              "spark_version": "11.3.x-scala2.12",
              "node_type_id": "Standard_D3_v2"
            }
          # Grant all users view permission on the notebook results, so that they can
          # see the result of our CI notebook 
          access-control-list-json: >
            [
              {
                "group_name": "users",
                "permission_level": "CAN_VIEW"
              }
            ]
```

### Run a notebook using library dependencies in the current repo and on PyPI
In the workflow below, we build Python code in the current repo into a wheel, use ``upload-dbfs-temp`` to upload it to a
tempfile in DBFS, then run a notebook that depends on the wheel, in addition to other libraries publicly available on
PyPI. 

Databricks supports a range of library types, including Maven and CRAN. See 
the docs
([Azure](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/api/latest/libraries#--library) |
[AWS](https://docs.databricks.com/dev-tools/api/latest/libraries.html#library) |
[GCP](https://docs.gcp.databricks.com/dev-tools/api/latest/libraries.html#library))
for more information.

```yaml
name: Run a single notebook on PRs

on:
  pull_request

env:
  DATABRICKS_HOST: https://adb-XXXX.XX.azuredatabricks.net
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checks out the repo
        uses: actions/checkout@v2
      # The step below does the following:
      # 1. Sends a POST request to generate an Azure Active Directory token for an Azure service principal
      # 2. Parses the token from the request response and then saves that in as DATABRICKS_TOKEN in the
      # GitHub enviornment.
      # Note: if the API request fails, the request response json will not have an "access_token" field and
      # the DATABRICKS_TOKEN env variable will be empty.
      - name: Generate and save AAD Token
        run: |
          echo "DATABRICKS_TOKEN=$(curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' \
            https://login.microsoftonline.com/${{ secrets.AZURE_SP_TENANT_ID }}/oauth2/v2.0/token \
            -d 'client_id=${{ secrets.AZURE_SP_APPLICATION_ID }}' \
            -d 'grant_type=client_credentials' \
            -d 'scope=2ff814a6-3304-4ab8-85cb-cd0e6f879c1d%2F.default' \
            -d 'client_secret=${{ secrets.AZURE_SP_CLIENT_SECRET }}' |  jq -r  '.access_token')" >> $GITHUB_ENV
      - name: Setup python
        uses: actions/setup-python@v2
      - name: Build wheel
        run: |
          python setup.py bdist_wheel
      # Uploads local file (Python wheel) to temporary Databricks DBFS
      # path and returns path. See https://github.com/databricks/upload-dbfs-tempfile
      # for details.
      - name: Upload Wheel
        uses: databricks/upload-dbfs-temp@v0
        with:
          local-path: dist/my-project.whl
        id: upload_wheel
      - name: Trigger model training notebook from PR branch
        uses: databricks/run-notebook@v0
        with:
          local-notebook-path: notebooks/deployments/MainNotebook
          # Install the wheel built in the previous step as a library
          # on the cluster used to run our notebook
          libraries-json: >
            [
              { "whl": "${{ steps.upload_wheel.outputs.dbfs-file-path }}" },
              { "pypi": "mlflow" }
            ]
          # The cluster JSON below is for Azure Databricks. On AWS and GCP, set
          # node_type_id to an appropriate node type, e.g. "i3.xlarge" for
          # AWS or "n1-highmem-4" for GCP
          new-cluster-json: >
            {
              "num_workers": 1,
              "spark_version": "11.3.x-scala2.12",
              "node_type_id": "Standard_D3_v2"
            }
          # Grant all users view permission on the notebook results
          access-control-list-json: >
            [
              {
                "group_name": "users",
                "permission_level": "CAN_VIEW"
              }
            ]
```

### Run notebooks in different Databricks Workspaces
In this example, we supply the `databricks-host` and `databricks-token` inputs
to each `databricks/run-notebook` step to trigger notebook execution against different workspaces.
The tokens are read from the GitHub repository secrets, `DATABRICKS_DEV_TOKEN` and `DATABRICKS_STAGING_TOKEN` and `DATABRICKS_PROD_TOKEN`.

Note that for Azure workspaces, you simply need to generate an AAD token once and use it across all
workspaces.

```yaml
name: Run a notebook in the current repo on pushes to main

on:
  push
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v2
      - name: Trigger notebook in staging
        uses: databricks/run-notebook@v0
        with:
          databricks-host: https://xxx-staging.cloud.databricks.com
          databricks-token: ${{ secrets.DATABRICKS_STAGING_TOKEN }}
          local-notebook-path: notebooks/MainNotebook.py
          # The cluster JSON below is for AWS workspaces. On Azure and GCP, set
          # node_type_id to an appropriate node type, e.g. "Standard_D3_v2" for
          # Azure or "n1-highmem-4" for GCP
          new-cluster-json: >
            {
              "num_workers": 1,
              "spark_version": "11.3.x-scala2.12",
              "node_type_id": "i3.xlarge"
            }
          # Grant users in the "devops" group view permission on the
          # notebook results
          access-control-list-json: >
            [
              {
                "group_name": "devops",
                "permission_level": "CAN_VIEW"
              }
            ]
      - name: Trigger notebook in prod
        uses: databricks/run-notebook@v0
        with:
          databricks-host: https://xxx-prod.cloud.databricks.com
          databricks-token: ${{ secrets.DATABRICKS_PROD_TOKEN }}
          local-notebook-path: notebooks/MainNotebook.py
          # The cluster JSON below is for AWS workspaces. On Azure and GCP, set
          # node_type_id to an appropriate node type, e.g. "Standard_D3_v2" for
          # Azure or "n1-highmem-4" for GCP
          new-cluster-json: >
            {
              "num_workers": 1,
              "spark_version": "11.3.x-scala2.12",
              "node_type_id": "i3.xlarge"
            }
          # Grant users in the "devops" group view permission on the
          # notebook results
          access-control-list-json: >
            [
              {
                "group_name": "devops",
                "permission_level": "CAN_VIEW"
              }
            ]
```

# Troubleshooting
To enable debug logging for Databricks REST API requests (e.g. to inspect the payload of a bad `/api/2.0/jobs/runs/submit`
Databricks REST API request), you can set the `ACTIONS_STEP_DEBUG` action secret to
`true`.
See [Step Debug Logs](https://github.com/actions/toolkit/blob/master/docs/action-debugging.md#how-to-access-step-debug-logs) 
for further details.

# License

The scripts and documentation in this project are released under the [Apache License, Version 2.0](LICENSE).
