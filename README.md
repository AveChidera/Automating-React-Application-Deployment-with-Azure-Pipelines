# **Automating React Application Deployment with Azure Pipelines**

---

## **Why This Lab?**

So far, we’ve deployed a static HTML file using Azure Pipelines to an Ubuntu VM running NGINX. That lab gave us a glimpse of what’s possible when infrastructure meets automation. But in real-world applications, deployment usually involves more than copying a file. There’s source code to build, tests to run, and dependencies to install.

In this lab, we're stepping into a realistic workflow. We’ll automate the deployment of a real React application. Along the way, you’ll learn how to integrate **build**, **test**, and **deployment** stages into a single CI/CD pipeline. The goal is to show you how professional development teams ensure code is built, validated, and deployed—automatically, reliably, and safely.

---

## **The Big Picture**

Here’s what we want to achieve:

1. Host a real React app using NGINX on an existing Ubuntu VM.
2. Push code changes to Azure Repos and have Azure Pipelines automatically:

   * Install dependencies
   * Build the app
   * Run unit tests
   * Deploy it to the VM if tests pass

We’ll begin by getting the application code into our Azure DevOps project, then configure the pipeline to bring the deployment to life.

---

## **Pre-Requisites**

Before starting, make sure you have:

* An Azure VM running Ubuntu or Debian with:

  * NGINX installed and running
  * Port 80 open to the internet
* Azure DevOps project ready
* SSH access to the VM (username and password)
* An SSH service connection named `ssh-to-nginx-vm` in Azure DevOps using **password-based authentication**

---

## **Step 1: Fork the React App into Azure Repos**

Our app lives on GitHub, but we want to bring it into Azure DevOps for better integration and control. Let’s start by forking it.

1. Open your terminal and run the following:

   ```bash
   git clone https://github.com/pravinmishraaws/my-react-app.git
   cd my-react-app
   ```

2. In Azure DevOps, create a new **empty repository** inside your project.

3. In your terminal, point the app to the new Azure Repo:

   ```bash
   git remote rename origin github
   git remote add origin <your-azure-repo-url>
   git push -u origin main
   ```

Now your React app is hosted in Azure Repos. Any change you push here will trigger your pipeline later.

---

## **Step 2: Understand the Deployment Workflow**

Before jumping into YAML, let’s understand what we want our pipeline to do.

We’re going to define a CI/CD pipeline with three stages:

1. **Build Stage:** Install dependencies and build the React app
2. **Test Stage:** Run automated unit tests
3. **Deploy Stage:** Copy the build to the VM and configure NGINX

This structure mimics a real development lifecycle: **build, validate, and deliver**.

---

## **Step 3: Define the Azure Pipeline**

Now we’ll define the actual pipeline. In your Azure Repo, create a new file at the root of the project called `azure-pipelines.yml`.

Paste the following code into it:

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  sshService: 'ssh-to-nginx-vm'
  artifactName: 'react-build'
  webRoot: '/var/www/html'

stages:
  # Stage 1: Build
  - stage: BuildReactApp
    displayName: 'Build React App'
    jobs:
      - job: Build
        steps:
          - checkout: self

          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
            displayName: 'Install Node.js'

          - script: npm install
            displayName: 'Install dependencies'

          - script: npm run build
            displayName: 'Build the app'

          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: 'build'
              artifactName: '$(artifactName)'
            displayName: 'Publish Build Artifact'

  # Stage 2: Test
  - stage: TestReactApp
    displayName: 'Run Tests'
    dependsOn: BuildReactApp
    condition: succeeded()
    jobs:
      - job: Test
        steps:
          - checkout: self

          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
            displayName: 'Install Node.js'

          - script: npm install
            displayName: 'Install dependencies'

          - script: npm run test tests -- --watchAll=false
            displayName: 'Run React Tests'

  # Stage 3: Deploy
  - stage: DeployToVM
    displayName: 'Deploy to Ubuntu VM with NGINX'
    dependsOn: TestReactApp
    condition: succeeded()
    jobs:
      - job: Deploy
        steps:
          - task: DownloadBuildArtifacts@0
            inputs:
              buildType: 'current'
              artifactName: '$(artifactName)'
              downloadPath: '$(Pipeline.Workspace)'
            displayName: 'Download Build Artifact'

          - task: CopyFilesOverSSH@0
            inputs:
              sshEndpoint: '$(sshService)'
              sourceFolder: '$(Pipeline.Workspace)/$(artifactName)'
              contents: '**'
              targetFolder: '$(webRoot)'
            displayName: 'Copy Files to NGINX Web Root'

          - task: SSH@0
            inputs:
              sshEndpoint: '$(sshService)'
              runOptions: 'inline'
              inline: |
                echo 'Configuring NGINX for React app...'
                echo 'server {
                    listen 80;
                    server_name _;
                    root /var/www/html;
                    index index.html;

                    location / {
                        try_files $uri /index.html;
                    }

                    error_page 404 /index.html;
                }' | sudo tee /etc/nginx/sites-available/default > /dev/null

                sudo chown -R www-data:www-data /var/www/html
                sudo chmod -R 755 /var/www/html
                sudo systemctl restart nginx
            displayName: 'Configure and Restart NGINX'
```

Let’s walk through this:

* The first stage builds the app and publishes the build folder as an artifact
* The second stage runs all your test cases
* Only if the tests pass, the third stage deploys the app to your VM

This mimics the kind of checks and balances real engineering teams implement in production.

---

## **Step 4: Run the Pipeline**

1. Commit the YAML file and push it to your `main` branch.
2. Go to **Pipelines → Create Pipeline** in Azure DevOps.
3. Select your repo and point to `azure-pipelines.yml`.
4. Click **Run**.

The pipeline will execute each stage in sequence:

* Build the React app
* Run tests
* Deploy to your Ubuntu VM if everything passes

---

## **Step 5: Verify Your Deployment**

Once the pipeline finishes, you should be able to access your deployed React app by visiting your VM's public IP in the browser:

```
http://<your-vm-ip>
```

If everything worked, you’ll see your React app running and served by NGINX.
