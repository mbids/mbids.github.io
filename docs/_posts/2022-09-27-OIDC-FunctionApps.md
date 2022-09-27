layout: post
title: "Deploying to Function Apps from GitHub with OIDC"
date: 2022-09-27 10:07:00 -0000
categories: CATEGORY-1 CATEGORY-2

**What is OIDC?**



OpenID Connect (OIDC) can be described as a simple identity layer on top of the OAuth 2.0 protocol. It allows you to verify the identity of the End-user based on the authentication performed by an Authorization server.

This protocol can easily be integrated with multiple different platforms. In this article we will walk you through of integrating OIDC with GitHub Actions to deploy our Function app without using any client secrets. 



**Why should I use OIDC with GitHub?**



GitHub Actions are often designed to access and interact with a cloud provider (Azure, AWS, GCP) to deploy software. Before a workflow can access these cloud providers and their resources it will need to authenticate with the cloud provider. Generally, this is done by supplying credentials, a password, certificate, or a token. These are most often stored as secrets in GitHub and the workflow then relies on this secret to authenticate with the cloud provider every time it runs. 



When using secrets this way you will have to create credentials in the cloud provider and then duplicate them in GitHub as a secret. 

With OIDC you can configure your workflow to request a short-lived accesstoken directly from the cloud provider. The cloud provider must support OIDC, which will need to be configured. You need to configure a trust relationship between GitHub and your cloud provider, which workflows can request the access tokens and when, based on specific conditions.



**Configuring Azure**



Firstly, you’ll need to federate your app registration with your GitHub Repository and its environment. For this example, we’ll use the Azure Portal. For production workloads we recommend considering an IaC(Infrastructure-as-Code) approach such as **[Terraform](https://www.terraform.io/)**. The documentation for federated credentials for terraform can be found **[here](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/application_federated_identity_credential).**



1. Navigate to your Azure AD Tenant and add an App Registration, you can find the documentation on how to do that **[here](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)**. 
2. Locate your app registration, in the App Registrations pane of the Azure AD page and select Certificates & Secrets in the left nav pane.  
3. Select the Federated Credentials tab and select Add credential. 
4. In the Federated Credential dropdown select GitHub actions deploying Azure resources. 
5. Insert your Organization and Repository for your GitHub Actions workflow. 
6. For Entity type select Environment and specify the desired name of your environment. Note: it’s very important this is exactly the same name as the environment you’re using in GitHub.
7. Add a Name for the federated credential.
8. The Issuer, Audiences and Subject identifier fields should auto populate based on your entered values.
9. Click Add to save the federated credential.



**Congratulations you did it!**



We have also added the App Registration as a contributor to one of our Function App resources in Azure. 



**GitHub and setting up the workflow**



We’ll still be needing the client ID, Tenant ID and Subscription ID of the relevant azure resources to tie this into our GitHub Actions. The recommendation is to store these as either secrets in the repository or as environment variables in the workflow, you will see an example of this later.



All that’s left now is to build out a workflow that can utilize this to deploy to our Function App in Azure. We’ll have an example of the complete workflow in the bottom of this article.



Start by creating your workflow in the .github/workflows/ folder of your repository, you could aptly name it something similar to deploy_funcapp.yml.



We’ll show an example of the head of a workflow and explain it below.
name: Build and deploy Python project to Azure Function App - mbids-oidc-funcapp
___



```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:



permissions:
      id-token: write
      contents: read
env:
  AZURE_FUNCTIONAPP_PACKAGE_PATH: '.' # set this to the path to your web app project, defaults to the repository root
  PYTHON_VERSION: '3.9' # set this to the python version to use (supports 3.6, 3.7, 3.8)
  AZURE_FUNCTIONAPP_NAME: 'mbids-oidc-funcapp'
  AZURE_FUNCTIONAPP_RG: 'mbids-oidc-funcapp_group'
```
___
\
There are a couple of things to notice here, one that is very important is the permissions, for OIDC to work with GitHub Actions, you must assign the workflow itself the id-token:write and contents:read permissions. Another thing is our env: variables, here you would need to replace $FunctionAppName and $FunctionAppResourceGroup with the appropriate values for your Azure Resources.  



This workflow runs on the event trigger workflow_dispatch and pushes to the branch main, which means you can start the workflow manually and it will be triggered on every push to the main branch. 



Now that we’ve got this set up, we need make sure our application can build before we try to deploy it, and we’ll show an example of this below 



---



``` yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2



     - name: Setup Python version
        uses: actions/setup-python@v1
        with:
          python-version: ${{ env.PYTHON_VERSION }}



     - name: Create and start virtual environment
        run: |
          python -m venv venv
          source venv/bin/activate
      - name: Install dependencies
        run: pip install -r requirements.txt



     - name: Upload artifact for deployment job
        uses: actions/upload-artifact@v2
        with:
          name: python-app
          path: |
            .
            !venv/
```
___
\
This is where we start defining workflow logic, and we’re using a couple of Actions from the GitHub Marketplace to do this for us, and lastly, we’re uploading it as an artifact. 



Now that we’re building the application, we can add another job that will deploy it to Azure, see the example below.



___



``` yaml
steps:
- name: Download artifact from build job
  uses: actions/download-artifact@v2
  with:
    name: python-app
    path: .



- name: 'Az CLI Login via OIDC'
  uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}



- name: "Zip files for deployment"
  run: zip python-app.zip *



- name: Azure CLI script
  uses: azure/CLI@v1
  with:
    azcliversion: 2.30.0
    inlineScript: |
      az functionapp deployment source config-zip -g ${{ env.AZURE_FUNCTIONAPP_RG }} -n ${{ env.AZURE_FUNCTIONAPP_NAME }} --src python-app.zip
```
___
\



This is where OIDC becomes interesting because we can authenticate our Azure CLI with OIDC. 



By extension of this, we can then use the Azure CLI to Deploy our application through OIDC.



Looking at the Az CLI Login via OIDC step we can see that we’re not using a client secret to authenticate.
The actual deployment is done in the step names Azure CLI script and is just the command:



az functionapp deployment source config-zip -g ${{ env.AZURE_FUNCTIONAPP_RG }} -n ${{ env.AZURE_FUNCTIONAPP_NAME }} --src python-app.zip



And you’re done! Congratulations, you’re now deploying with OIDC! 



**Summary**



In this post we’ve learned the basics about what OIDC is, and how to set it up between Azure and GitHub Actions. In this gist below you'll be able to find the workflow.



[![image](https://user-images.githubusercontent.com/7994533/187177942-388e2624-82e6-474c-a0a0-ad6bb07c85c1.png)](https://gist.github.com/mbids/cc3d6df5c1c86d6e6daff4a04a71ada9)



Updating your workflow to use OIDC tokens you can adopt several good security practices.



Such as:
1. No cloud secrets: You won't need to duplicate your cloud credentials as long-lived GitHub secrets.
2. Authentication and authorization management: You have more granular control over how workflows can use credentials, using your cloud provider's authentication (authN) and authorization (authZ) tools to control access to cloud resources.
3. Rotating credentials: With OIDC, your cloud provider issues a short-lived access token that is only valid for a single job, and then automatically expires.



Do keep in mind that it can also come with security concerns by allowing a third party to authenticate users. Also be aware of hostile OpenID providers authenticating spambots and such. Make sure you weigh the pros against the cons before implementing OIDC into your structure.



Hopefully this guide proves helpful in your future GitHub Action adventures. If you’re curious to learn more about this or maybe have questions don’t hesitate to contact us!
