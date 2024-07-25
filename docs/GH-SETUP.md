# Multi-Environment GitHub Setup

This document outlines the steps to set up a multi-environment workflow to deploy infrastructure and services to Azure using GitHub Actions, taking the solution from proof of concept to production-ready.

# Assumptions:

- This example assumes you're using a GitHub organization with GitHub environments
- This is a tightly coupled example, which deploys infrastructure in the same pipeline as all of the services
- This example deploys 3 environments: dev, test, and prod
- This example uses 'azd pipeline config' (https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/configure-devops-pipeline?tabs=GitHub), which as of writing, is in preview. This feature enables rapid GitHub workflow setup and federated identity auth configuration for enhanced security
- All below commands are run as a one-time setup on a local machine by an admin who has access to the GitHub repo and Azure subscription
- This example does not cover configuring any naming conventions
- Original remote versions of the orchestrator, frontend, and ingestion repositories are used; in a real scenario, you would fork these repositories and use your forked versions. This would require updating the repository URLs in the azure.yaml file.

# Decisions required:

- Whether a single or multiple Service Principals will be used for each environment
- Whether to use federated identity or client secret for authentication
- Decisions on which GitHub repo, Azure subscription, and Azure location to use

# Prerequisites:

- Azure CLI (https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli)
- Azure Developer CLI (https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd?tabs=winget-windows%2Cbrew-mac%2Cscript-linux&pivots=os-windows)
- GitHub CLI (https://cli.github.com/)
- PowerShell 7
- GitHub organization with ability to provision environments (e.g., GH Enterprise)
- Bash shell (e.g. Git Bash)
  - Note that all commands are written for bash shell
- Personnel with Azure admin (can create Service Principals) and GitHub admin (owns repo/org) access

# Steps:

## 1. Create Service Principals for each env

<!-- CLI: https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash#create-a-service-principal

Portal: https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal -->

<!-- Note: You will need the app name in later steps

```bash
dev_principal_name='<dev-sp-name>'
test_principal_name='<test-sp-name>'
prod_principal_name='<prod-sp-name>'
``` -->
<!-- 
Given the name of the Service Principal, get the client ID. You will need the client ID for later steps.

Login and query for the DisplayName and AppId and place the value in a variable. In the event that there are multiple Service Principals containing the same name, so you need to pull the correct ID. These can also be retrieved from the Azure portal.

```bash
az login
# example: az ad sp list --display-name $dev_principal_name --query "[].{DisplayName:displayName, AppId:appId}" --output table
dev_client_id='<dev-sp-client-id>'
test_client_id='<test-sp-client-id>'
prod_client_id='<prod-sp-client-id>'
``` -->



## 1. Create azd environments

`cd` to the root of the repo. Create an azd environment per target environment, and configure the pipeline for each environment. Note that these environment names are reused as the GitHub environment names later.


```bash
dev_env='<dev-env-name>' # Example: dev
test_env='<test-env-name>' # Example: test
prod_env='<prod-env-name>' # Example: prod
```

Read more about azd pipeline config: https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/configure-devops-pipeline?tabs=GitHub

- By default, azd pipeline config uses OpenID Connect (OIDC), called federated credentials. If you'd rather not use OIDC, run azd pipeline config --auth-type client-credentials.
Note: If you want to manage and authenticate with a secret rather than using federated identity, you would need to create a secret for each Service Principal, store it as a secret in GitHub, and modify the workflow to use the secret for authentication. This is not covered in this example. If you choose to use a client secret, you may skip step 4.
CLI Doc: https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/reference#azd-pipeline-config

Define the names of the Service Principals that will be used for each environment. You will need the app name in later steps.
Note that azd pipeline config creates a new Service Principal for each environment.
There are a variety of ways to complete the setup below, e.g., you may manually perform all steps below for additional control, you may elect to use a single Service Principal for all environments, etc.

```bash
dev_principal_name='<dev-sp-name>'
test_principal_name='<test-sp-name>'
prod_principal_name='<prod-sp-name>'
```

When running 'azd pipeline config' for each env, choose GitHub as provider, Az subscription, and Az location. When prompted to commit and push your local changes to start the configured CI pipeline, say 'N'.

### Dev

```bash
az login
```

```bash
azd env new $dev_env
azd pipeline config --auth-type federated --principal-name $dev_principal_name
```

### Test

```bash
azd env new $test_env
azd pipeline config --auth-type federated --principal-name $test_principal_name
```

### Prod

```bash
azd env new $prod_env
azd pipeline config --auth-type federated --principal-name $prod_principal_name
```

After performing the above steps, you will see corresponding files to your azd environments in the .azure folder.

If you run 'azd env list', you will see the newly created environments.

You may change the default environment by running `azd env select <env-name>`, for example:

```bash
azd env select $dev_env
```

# 3. Set up GitHub Environments

the below paths may need to be updated if using a GH organization

Set up initial variables:

```bash
org='<your-org-name>'
repo='<your-repo-name>'
```

Run GitHub CLI commands to create the environments:

```bash
gh auth login

gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$dev_env
gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$test_env
gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$prod_env
```

Configure the repository and environment variables: Delete the AZURE_CLIENT_ID and AZURE_ENV_NAME variables at the repository level as they aren't needed and only represent what was set for the environment you created last. AZURE_CLIENT_ID will be reconfigured at the environment level, and AZURE_ENV_NAME will be passed as an input to the deploy job.

```bash
gh variable delete AZURE_CLIENT_ID
gh variable delete AZURE_ENV_NAME
```

Get the client IDs of the Service Principals you created. Ensure you previously ran `az login`. 

```bash
dev_client_id=$(az ad sp list --display-name $dev_principal_name --query "[].appId" --output tsv)
test_client_id=$(az ad sp list --display-name $test_principal_name --query "[].appId" --output tsv)
prod_client_id=$(az ad sp list --display-name $prod_principal_name --query "[].appId" --output tsv)
```

Note:
In the event that there are multiple SPs containing the same name, review the table of results to pull the correct ID. These can also be retrieved from the Azure portal.
```bash
az ad sp list --display-name $dev_principal_name --query "[].{DisplayName:displayName, AppId:appId}" --output table # return results in a table format
dev_client_id='<guid>'
```

Set the variables at the environment level

```bash
gh variable set AZURE_CLIENT_ID -b $dev_client_id -e $dev_env
gh variable set AZURE_CLIENT_ID -b $test_client_id -e $test_env
gh variable set AZURE_CLIENT_ID -b $prod_client_id -e $prod_env
```

Consider setting up deployment protection rules for each environment by going to Settings > Environments > env-name, and setting Deployment protection rules (e.g., required reviewers)

- If using client secret authentication, you would need to set up the client secret in the Azure Portal, configure it as a secret in GitHub as an environment variable and modify the workflow to use the secret for authentication. This is not covered in this example.


# 4. Configure Azure Federated credentials to use newly set up GitHub environments
```bash
issuer="https://token.actions.githubusercontent.com/"
audiences="api://AzureADTokenExchange"

echo '{"name": "'"${org}-${repo}-${dev_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$dev_env"'", "description": "'"${dev_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $dev_client_id --parameters ./federated_id.json

echo '{"name": "'"${org}-${repo}-${test_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$test_env"'", "description": "'"${test_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $test_client_id --parameters ./federated_id.json

echo '{"name": "'"${org}-${repo}-${prod_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$prod_env"'", "description": "'"${prod_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $prod_client_id --parameters ./federated_id.json

rm federated_id.json

```
<!-- # 4. [If using Federated Credentials] Reconfigure Azure Federated credentials to use newly set up GitHub environments

You may skip this step if you are using client secret authentication.

This is a one-time manual step in the Azure portal for each Service Principal

- a) Open Azure Portal and navigate to Microsoft Entra ID
- b) Select App Registrations
- c) Search for the app name you used for your Dev environment
- d) Click on Certificates & secrets
- e) Select the Federated credentials tab
- f) choose 'Add credential'
- g) Choose 'GitHub Actions deploying Azure resources' in the 'Federated credential scenario'
- h) Fill in your organization and repository name
- i) Set the 'Entity type' to 'Environment'
- j) Set the 'GitHub environment name' to the name of the environment you created in GitHub
- k) provide a Name in the Credential Details section, for example: org-repo-env_name-env -->
<!-- - f) choose the credential that ends with '-main'
- g) Update the 'Entity type' to 'Environment'
- h) Edit the 'GitHub environment name' to the name of the environment you created in GitHub
- i) choose Update -->
<!-- - _Perform steps a-i for each Service Principal_ -->

<!-- If you have multiple Service Principals: -->
<!-- If you have only one Service Principal/multiple environments using the same Service Principal:

- f) choose 'Add credential'
- g) Choose 'GitHub Actions deploying Azure resources' in the 'Federated credential scenario'
- h) Fill in your organization and repository name
- i) Set the 'Entity type' to 'Environment'
- j) Set the 'GitHub environment name' to the name of the environment you created in GitHub
- k) provide a Name in the Credential Details section, for example: org-repo-env_name-env
- _Perform steps f-k for each environment_ -->

Note: The existing/unmodified credentials created by Azure Developer CLI may be deleted.

# 5. Modify the workflow files as needed for deployment

- The following files in the .github/workflows folder are used to deploy the infrastructure and services to Azure:
  - azure-dev.yml
    - This is the main file that triggers the deployment workflow. The environment names are passed as inputs to the deploy job, **which needs to be edited to match the environment names you created**.
    - You may edit the workflow_dispatch to suit your workflow trigger needs.
  - deploy-template.yml
    - This is a template file that is used to deploy the infrastructure and services to Azure. This file needs to be edited if you are using client secret authentication.


# TODOs and potential improvements:

- Add a step to package the application code (in azure-dev.yml) and upload it as a GitHub artifact to do a package deploy (in deploy.yml)

  - azure-dev.yml suggested addition:

    ```
    build:
        runs-on: ubuntu-latest

        steps:
        - name: Checkout
          uses: actions/checkout@v3

        - name: Install azd
          uses: Azure/setup-azd@v0.1.0

        - name: Package (may need to package by service)
          run: azd package --output-path ./dist/soln.zip --environment NONE --no-prompt
          working-directory: ./

        - name: Upload Package
          uses: actions/upload-artifact@v3
          with:
            name: package
            path: ./dist/soln.zip
            if-no-files-found: error
    ```

  - deploy-template.yml suggested addition:

    ```
    - name: Download Package
      uses: actions/download-artifact@v3

    - name: Deploy Application
      run: azd deploy --from-package ./package/soln.zip --no-prompt
      env:
        AZURE_ENV_NAME: ${{ inputs.AZURE_ENV_NAME }}
        AZURE_LOCATION: ${{ vars.AZURE_LOCATION }}
        AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    ```

- Provide client credential authentication example
- Provide example of how to configure naming conventions
- Provide example of setting up self-hosted runners for network-restricted deployments
- Consider decoupling infrastructure and app code deployments
- Offer branching strategy guidance
- Update deploy.yml to only deploy service(s) which have changed (with `azd deploy <service>`) (https://github.com/MicrosoftDocs/azure-dev-docs/blob/main/articles/azure-developer-cli/reference.md#azd-deploy)
  - this would require a more complex workflow to determine which services have changed, and each service is in a separate repository