# Github Actions and Azure

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0069.png)

> This is the article created at Nov 16, 2019 and moved from Medium.

A few days ago (November 13, 2019) GitHub announced the general availability of GitHub Actions. CI/CD solution from GitHub. I tested it with two of my projects and I have to say I’m impressed. But first thing first.

I will describe below how I configured two of my projects (Angular 8 and .NET Core 3) with GitHub Actions.
<!--more-->

---

### Continuous Integration

After each commit we would like to have a feedback about build result, linter and unit/e2e tests etc. For that reason we have to create `yml` files in folder: `.github/workflows` (for example: `.github/workflows/build.yml`) with steps definition.

**Angular 8**

For angular application our file should look like on below snippet.

```yaml
name: Build
on:
  push:
    branches-ignore: master
env:
  NODE_VERSION: '10.x'
jobs:
  build:
    name: Build Angular
runs-on: ubuntu-latest
steps:
    - uses: actions/checkout@master
    - name: Use Node.js ${{ env.NODE_VERSION }}
      uses: actions/setup-node@v1
      with:
        node-version: ${{ env.NODE_VERSION }}
    - name: Install dependencies
      run: npm install
    - name: Lint
      run: npm run lint
    - name: Build
      run: npm run build -- --prod
    - name: Unit tests
      run: npm run test
    - name: E2E tests
      run: npm run e2e
```

We have to define NodeJS version. We will run our build on ubuntu server. And we have a few steps that we should recognize if we are familiar with Angular apps development.

Also we run that configuration only for branches other then `master`. For `master` branch we have separate configuration (with deployment to Azure).

**.NET Core 3**

GitHub Actions definition file for .NET Core is also very simple.

```yaml
name: Build
on:
  push:
    branches-ignore: master
jobs:
  build:
    name: Build .NET Core
runs-on: ubuntu-latest
steps:
    - uses: actions/checkout@v1
    - name: Setup .NET Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.0.100
    - name: Build
      working-directory: ./src/Application.Api
      run: dotnet build --configuration Debug
    - name: Unit tests
      working-directory: ./tests/Application.Api.Tests
      run: dotnet test
```

We also defined that particular configuration should be run on branches different then `master`. We are using Ubuntu. And all steps should be familiar to .NET developers.

---

### Continuous Deployment

Continuous deployment will deploy our applications to our hosting provider. Below there is example how to configure GitHub with Azure.

#### Prerequisite

First of all we have to export publish profiles from Azure (from our WebApps home pages).

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0070.png)

Then we have to create in GitHub Secrets page new secrets (in my case separate secrets for both applications, because they are separate WebApps in Azure). It’s important to save the secret name, because we will use it in our CD configuration.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0071.png)

**That’s all**. Now we can configure GitHub Actions for deployment to Azure.

#### GitHub Actions for CD

We will configure two continuous deployments configurations.

**Angular 8**

For Angular application we have `yml` file like on below snippet.

```yaml
name: Deploy to Azure
on:
  push:
    branches:
      - master
env:
  AZURE_WEBAPP_NAME: application-web
  AZURE_WEBAPP_PACKAGE_PATH: './dist/ApplicationWeb'
  NODE_VERSION: '10.x'
jobs:
  build-and-deploy:
    name: Build and Deploy
runs-on: ubuntu-latest
steps:
    - uses: actions/checkout@master
    - name: Use Node.js ${{ env.NODE_VERSION }}
      uses: actions/setup-node@v1
      with:
        node-version: ${{ env.NODE_VERSION }}
    - name: Install dependencies
      run: npm install
    - name: Build
      run: npm run build -- --prod
    - name: 'Deploy to Azure WebApp'
      uses: azure/webapps-deploy@v1
      with:
        app-name: ${{ env.AZURE_WEBAPP_NAME }}
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
```

First steps are similar to CI build configuration (we have to build our application first). But the last step is step created with `azure/webapps-deploy` workflow. We have to configure:

- `app-name` — application name in Azure
- `publish-profie` — name of the secret from GitHub
- `package` — path to directory which we would like to deploy (in above example: `./dist/ApplicationWeb`.

And that’s it. Really clear and simple!

**.NET Core 3**

For .NET Core our configuration file it’s very similar.

```yaml
name: Deploy to Azure

on:
  push:
    branches:
      - master

env:
  AZURE_WEBAPP_NAME: application-api
  AZURE_WEBAPP_PACKAGE_PATH: './src/Application.Api/bin/Release/netcoreapp3.0'
  NODE_VERSION: '10.x'

jobs:
  build-and-deploy:
    name: Build and Deploy

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v1
    - name: Setup .NET Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.0.100
    - name: Build
      working-directory: ./src/Application.Api
      run: dotnet build --configuration Release
    - name: 'Deploy to Azure WebApp'
      uses: azure/webapps-deploy@v1
      with:
        app-name: ${{ env.AZURE_WEBAPP_NAME }}
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
```

First we need to build our application and then we can deploy our assemblies to Azure.

---

### Summary

After create our workflows in GitHub actions I have on GitHub page like on below image.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0072.png)

Thus I have really clean information about my CI/CD results. And I have to say that it’s really fast. For sure GitHub Action will have positive impact on my GitHub repositories and my work. **Thanks GitHub!**