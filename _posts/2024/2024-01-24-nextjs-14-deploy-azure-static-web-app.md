---
title: "Deploy NextJs 14 app to Azure static web app with GitHub Actions"
tags: 
- nextjs
- azure
---

# Deploy NextJs 14 app to Azure static web app with GitHub Actions

This post is for my own Archive on my activity but maybe it can be useful for someone else.

For me it was unclear how to find the right parameters to configure the direct deployment to Azure using `azure/static-web-apps-deploy`.

In my case, using the `npx create-next-app@latest` command is the only working configuration is the following:

Notice the "out" is not specified anywhere but the execution of the static-web-apps-deploy will use "out" folder as target where to store the `next build` results.

```
app_location: "/" # App source code path
api_location: "" # Api source code path - optional
output_location: "out" # Built app content directory 
```


```yaml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main

jobs:
  build_and_deploy_job:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v3

      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APP_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }} # Used for GitHub integrations (i.e. PR comments)
          action: "upload"
          ###### Repository/Build Configurations - These values can be configured to match your app requirements. ######
          # For more information regarding Static Web App workflow configurations, please visit: https://aka.ms/swaworkflowconfig
          app_location: "/" # App source code path
          api_location: "" # Api source code path - optional
          output_location: "out" # Built app content directory - optional
          ###### End of Repository/Build Configurations ######

  close_pull_request_job:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    name: Close Pull Request Job
    steps:
      - name: Close Pull Request
        id: closepullrequest
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APP_TOKEN }}
          action: "close"
```
