# CICD-with-GitHub-Actions
GitHub Actions is a continuous integration and continuous delivery (CI/CD) platform that allows you to automate your build, test, and deployment pipeline. You can create workflows that build and test every pull request to your repository, or deploy merged pull requests to production. Simply put, GitHub Actions allows automation of software delivery directly from GitHub (codebase).

## Core Components
## Workflows
A GitHub Actions Workflow is a configurable, automated process that executes one or more actions. Workflows are defined by YAML configuration files. It is set up in your GitHub repository to build, test, package, release, or deploy any project on GitHub.

## Jobs
A job is a unit of work that you can define to run sequentially or in parallel on a runner. Jobs in a workflow can be used to organize and configure the steps that are executed when a workflow is triggered.

## Steps
Steps refer to individual tasks that makes up a job. They define what actions GitHub should perform when a workflow is triggered. Steps are executed sequentially by default, meaning each step runs one after another unless specified otherwise.

In this project, we demonstrate the automation of software delivery process of an application to a kubernetes cluster using GitHub Actions

## Architecture

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/5d12e652-67b1-4939-9473-12912d002a7b)

## Creating the Workflow
## Step 1: Define trigger condition
Here, we define the condition to trigger our workflow and the branch where the trigger is set. Here, the workflow will be triggered for every push to the main branch.

```
name: Reactapp Workflow
on:
  push:
    branches:
      - main
```

## Step 2: Define jobs
In our jobs defined, we also make use of github marketplace actions. Actions are pre-built reusable automation components designed for specific tasks.

## Job 1: Unit testing
Firstly, we check out of the repository. Next, we make use of github marketpalce action to cache dependencies. This helps to speed up our workflow runs and reduce load on package manager registry. Then, we integrate unit testing. Unit tests ensure that each unit of code behaves as expected. They validate that the code functions correctly under various input conditions and edge cases. This helps to maintain code quality by ensuring that each function or method performs its intended functionality reliably. Finally, we archive the test results.

```
 unit-testing:
    name: Unit testing
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
      - name: Setup Nodejs version-10
        uses: actions/setup-node@v3
        with:
          node-version: 10
      - name: Cache NPM dependencies
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.os }}-node-modules${{ hashFiles('package-lock.json') }}
      - name: Install dependencies
        run: npm install
        working-directory: ./api
      - name: Unit testing
        run: npm test
        working-directory: ./api
      - name: Archive test result
        uses: actions/upload-artifact@v3
        with:
          name: test-result
          path: ./api/test-result.json
```

## Job 2: Containerization
Here, we go through the process of dockerizing our application. With GitHub marketplace actions, we authenticate to dockerhub registry, build docker image using the docker file created, test the image and then push the image to DockerHub Registry and GitHub Container Registry. Additionally, to successfully carry out this task, we define some variables and secrets in our repository and then reference them properly.

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/1021c116-b6b2-4dcf-bf2c-64a14ce65fe3)

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/e90f3492-a5bc-46c3-9b8b-2c1f55fa7118)

```
docker:
    name: Containerization
    permissions: 
      packages: write
    runs-on: ubuntu-latest
    needs: [unit-testing]
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_TOKEN }}
      - name: Dockerhub login
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Docker build for testing
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: ${{ vars.DOCKERHUB_USERNAME }}/reactapp:${{ github.sha }}
      - name: Docker image testing
        run: |
          docker images
          docker run --name reactapp -d \
            -p 3080:3080 \
            ${{ vars.DOCKERHUB_USERNAME }}/reactapp:${{ github.sha }}
          export IP=$( docker inspect -f '{{ range.NetworkSettings.Networks }}{{ .IPAddress }}{{ end }}' reactapp)
          echo $IP
      - name: Docker push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ vars.DOCKERHUB_USERNAME }}/reactapp:${{ github.sha }}
      - name: GHCR login
        uses: docker/login-action@v2.2.0
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GH_TOKEN }}
      - name: Container Registry push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/reactapp:${{ github.sha }}
```

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/fb10ff55-baec-412d-adec-86a56d6190db)

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/3eca0298-575c-4699-9adb-02c2da0ab18d)

## Job 3: Dev-Deploy
Here, before proceeding to define the dev-deploy job, we first of all setup a cluster where we will deploy our application. Next, we define the job. In this job, we configure AWS credentials, install AWS CLI, connect to our cluster by adding the kubeconfig file to the repository secret, update kubeconfig for EKS and deploy the application to the cluster using the kubernetes manifest files.

```
dev-deploy:
    needs: docker
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_TOKEN }}
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ vars.AWS_REGION }}
      - name: Install kubectl
        run: sudo snap install kubectl --classic
      - name: Install AWS CLI
        run: |
          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
          unzip awscliv2.zip
          sudo ./aws/install --update
      - name: Update kubeconfig for EKS
        run: |
          aws eks update-kubeconfig --name my-eks-cluster --region ${{ vars.AWS_REGION }}    
      - name: Fetch kubernetes cluster details
        run: |
          kubectl version
          echo ---------------------------------
          kubectl get nodes
      - name: Replace token in manifest files
        uses: cschleiden/replace-tokens@v1
        with:
          tokenPrefix: '_{_'
          tokenSuffix: '_}_'
          files: '["kubernetes/development/*.yaml"]'
        env:
          NAMESPACE: ${{ vars.NAMESPACE }}
          REPLICAS: ${{ vars.REPLICAS }}
          IMAGE: ${{ vars.DOCKERHUB_USERNAME }}/reactapp:${{ github.sha }}
      - name: Check files
        run: |
          cat kubernetes/development/*.yaml
      - name: Create namespace if not exists
        run: |
          kubectl create namespace development || true
      - name: Deploy to EKS cluster
        run: |
          kubectl apply -f kubernetes/development
```

Finally, we commit the workflow changes and the workflow executes automatically

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/810a7b4c-f5ae-43e8-a48f-fa6cd52a3b14)

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/670c9ca8-92d2-437d-b392-0b15888e97d5)

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/60e7ed51-6fdf-40ff-b51b-098cbc619a14)

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/fa958a52-1216-44d3-875f-a09410f4123a)

We then check the cluster to view the resources deployed

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/06b8673c-3447-4502-b587-a314c5773a64)

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/90a11cb8-0e9d-4cc7-856e-1231e2f0e379)

Thereafter, we access the application using the Load Balancer DNS name as shown in the Kubernetes service

![image](https://github.com/kenchuks44/CICD-with-GitHub-Actions/assets/88329191/1a3e7a14-f542-4a79-968d-d3143b30727e)

Congratulations!!!








