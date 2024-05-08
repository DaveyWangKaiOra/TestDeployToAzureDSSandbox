# Test the deployment to Azure Data Science Sandbox environment

## Prerequisites
An Azure account with an active subscription.
A GitHub account.
A working container registry (e.g., Azure Container Registry).
An Azure App Service app for containers or Azure Container Instances.

## Step 1: Create a GitHub Actions Workflow
Navigate to your GitHub repository where your Dockerfile is located.
Go to the Actions tab and click on New workflow.
Choose set up a workflow yourself and select YAML.
Name your workflow file (e.g., deploy-to-azure.yml) and save it in the .github/workflows directory of your repository.

## Step 2: Define the Workflow
Here’s an example workflow that builds and deploys a Docker container to Azure App Service. You can modify it according to your needs, especially the AZURE_WEBAPP_NAME, AZURE_WEBAPP_PUBLISH_PROFILE, and other environment-specific details.

name: Build and deploy a container to an Azure Web App

env:
  AZURE_WEBAPP_NAME: MY_WEBAPP_NAME   # set this to your application's name

on:
  push:
    branches:
      - main

permissions:
  contents: 'read'
  packages: 'write'

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to GitHub container registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Lowercase the repo name
        run: echo "REPO=${GITHUB_REPOSITORY,,}" >>${GITHUB_ENV}

      - name: Build and push container image to registry
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ghcr.io/${{ env.REPO }}:${{ github.sha }}
          file:./Dockerfile

  deploy:
    runs-on: ubuntu-latest

    needs: build

    environment:
      name: 'production'
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}

    steps:
      - name: Lowercase the repo name
        run: echo "REPO=${GITHUB_REPOSITORY,,}" >>${GITHUB_ENV}

      - name: Deploy to Azure Web App
        id: deploy-to-webapp
        uses: azure/webapps-deploy@85270a1854658d167ab239bce43949edb336fa7c
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          images: 'ghcr.io/${{ env.REPO }}:${{ github.sha }}'
          
## Step 3: Configure Secrets
GitHub Secrets: You need to add the following secrets to your GitHub repository settings:
AZURE_WEBAPP_PUBLISH_PROFILE: Your Azure App Service publish profile.
AZURE_CREDENTIALS: Your Azure credentials in JSON format, which includes clientId, clientSecret, subscriptionId, and tenantId.

## Step 4: Push Changes
Commit and push your changes to the main branch. The workflow will trigger on push events to the main branch, building and deploying your Docker container to Azure.

Additional Notes
For Azure Container Instances, you might need to adjust the deployment step to use the azure/container-instances-deploy action instead of azure/webapps-deploy.
Ensure your Azure App Service or Azure Container Instances is configured to allow deployments from GitHub Actions.
This process automates the deployment of your Docker container from GitHub to Azure, leveraging GitHub Actions for CI/CD.
