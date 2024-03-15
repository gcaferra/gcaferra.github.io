---
title: "Deploy Blazor website to Azure Static web App"
tags: 
- blazor
- dotnet
- azure
---


# Deploy Blazor website to Azure Static web App

## Prepare secrets for the deployment

This is the YAML I've used to deploy the application to Azure.

Just remember to download the Publish Profile from the azure portal
![Azure Portal publish profile export button](/assets/images/download-publish-profile-button.png "Click on Download publish profile")

After downloaded the file, you can create a new secret in github Secrets called 

```
AZURE_DEPLOYMENT_TOKEN
```

![Github new secret creation](/assets/images/new-deployment-secret.png "New secret creation")

And paste the whole content of the downloaded file into it.

Once you have completed this, you can use the following configuration to deploy:

## YAML 

``` YAML

name: Deploy web app to Azure Static Web Apps

on:
  push:
    branches: [ "main" ]

# Environment variables available to all jobs and steps in this workflow
env:
  AZURE_WEBAPP_NAME: your-web-app-name-here


permissions:
  contents: read

jobs:
  build_and_deploy_job:
    permissions:
      contents: read # for actions/checkout to fetch code
      pull-requests: write # for Azure/static-web-apps-deploy to comment on PRs
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
      - name: Set up .Net Core
        uses: actions/setup-dotnet@v3
        with:
            dotnet-version: '8.0.x'

      - name: Publish .NET Core Project
        run: dotnet publish path-to-the-project.csproj -c Release -o publish

      - name: Deployment
        id: buildanddeploy
        uses: azure/static-web-apps-deploy@v1
        with:
          action: 'upload'
          repo_token: ${{ secrets.GITHUB_TOKEN }} # Used for Github integrations (i.e. PR comments)
          azure_static_web_apps_api_token: ${{ secrets.AZURE_DEPLOYMENT_TOKEN }}
          app_location: publish/wwwroot
          api_location: ''
          app_artifact_location: publish/wwwroot
```
