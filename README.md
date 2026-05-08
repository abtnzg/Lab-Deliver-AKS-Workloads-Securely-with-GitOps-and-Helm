Introduction
In this Hands-on lab, you will learn how to deliver AKS workloads securely using GitOps and Helm. You will set up a GitHub repository, configure GitHub Actions for CI/CD, and deploy applications to an AKS cluster using ArgoCD and Helm charts.

Log in to the Azure portal
Log in to the Azure portal using the credentials provided on the lab page. Be sure to use an incognito or private browser window to ensure you're using the lab account, rather than your own.

Fork and Configure Github Repository
Fork the repository for the Hands-on lab
Sign into GitHub using your personal GitHub account.

Go to the repository for this Hands-on lab: https://github.com/pluralsight-cloud/Lab-Deliver-AKS-Workloads-Securely-with-GitOps-and-Helm

Click Fork.

Note: If you have completed this Hands-on lab more than once, you need to delete the existing fork before you can continue.

Note your GitHub username in the Owner field and the Repository name for later.

Click Create Fork.

Enable Issues in the Forked Repository
Under your repository name, click Settings.

Under Features, select the checkbox next to Issues.

Create Staging Environment
Still on the Settings tab.
In the left menu, under the Code and Automation heading, select Environments.
Click New environment.
Provide the name staging and click Configure environment.
Create Production Environment
Select Environments in the breadcrumbs to return to the list of environments.
Click New environment.
Provide the name production and click Configure environment.
Click the checkbox next to Required reviewers.
In the text box under Add up to 6 more reviewers, type your GitHub username and select it from the dropdown list.
Click Save protection rules.
Install and Configure ArgoCD
Install ArgoCD
Go to the Azure Portal.

Open Cloud Shell by selecting the icon in the top menu.

Select Bash.

Select No storage account required and select the existing subscription.

Click Apply.

Note: You can maximize the Cloud Shell pane by clicking the maximize icon in the top menu of the Cloud Shell pane for easier viewing of the output from the following commands.

Run the following commands to save the resource group name and AKS cluster name to variables, and retrieve the AKS credentials:

RG=$(az group list --query [].name --output tsv)
AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
az aks get-credentials --resource-group $RG --name $AKS
Deploy ArgoCD by running the following commands:

kubectl create namespace argocd
kubectl create -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
Wait for the resources to be created.

Create an ArgoCD Application
Set a variable for your forked repository URL:

Important Note: Before applying the applications, make sure your GitHub repository URL is correct and replace the placeholder values. This is the source ArgoCD uses to sync your manifests. If this value is incorrect, ArgoCD will not be able to retrieve your application manifests, and your workloads may not deploy as expected.

If needed, you can update the value and re-run the kubectl apply command. The command is idempotent, so it is safe to reapply.

export REPO_URL="https://github.com/<GitHub Username>/<GitHub Repository Name>"
Set a variable for the Azure Container Registry login server.

export IMAGE_REGISTRY=$(az acr list --query [].loginServer -o tsv)
Create an ArgoCD Application for the staging release by running the following command:

kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: simple-grocery-store-staging
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  source:
    repoURL: ${REPO_URL}
    targetRevision: HEAD
    path: charts/simple-grocery-store
    helm:
      valueFiles:
        - values/staging.yaml
      values: |
        global:
          imageRegistry: ${IMAGE_REGISTRY}/
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
Create a second ArgoCD Application for the production release by running the following command:

kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: simple-grocery-store-production
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  source:
    repoURL: ${REPO_URL}
    targetRevision: HEAD
    path: charts/simple-grocery-store
    helm:
      valueFiles:
        - values/production.yaml
      values: |
        global:
          imageRegistry: ${IMAGE_REGISTRY}/
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
Confirm with the following command:

kubectl get application --namespace argocd
Note: The applications will remain out of sync for now, as GitHub hasn't pushed the images to the registry yet.

Configure CI/CD pipeline using Workload Identity Federation and GitHub Actions
Create Federated Credentials
Minimize Cloud Shell

In the Azure portal, select the Managed Identity from the list of resources.

In the left menu under Settings select Federated credentials.

Click on Add Credential.

For the Federated credential scenario, select GitHub Actions deploying Azure resources.

In the Connect your Github account section, provide the following information to create a credential for the staging environment:

Organization: Provide your GitHub username
Repository: Provide the repository name of your forked repository, for example: Lab-Deliver-AKS-Workloads-Securely-with-GitOps-and-Helm
Entity: Environment
Environment: staging
In the Credential details section, provide a Name for the federated credential.

Note: It's a good idea to use a descriptive name, for example: GitHubUsername-RepositoryName-Environment

Click on Add to create the federated credential.

Repeat the steps to create a second federated credential for the production environment.

Organization: Provide your GitHub username
Repository: Provide the repository name of your forked repository, for example: Lab-Deliver-AKS-Workloads-Securely-with-GitOps-and-Helm
Entity: Environment
Environment: production
Repeat the steps to create a third federated credential for the main branch.

Organization: Provide your GitHub username
Repository: Provide the repository name of your forked repository, for example: Lab-Deliver-AKS-Workloads-Securely-with-GitOps-and-Helm
Entity: Branch
Branch: main
Note: For simplicity, a single managed identity with a Contributor role assignment is being used in this Hands-on lab to deploy to both staging and production, and from the main branch. In a production environment, you could use separate managed identities with permissions scoped to the cluster or namespace or container registry where the credential is permitted to deploy and apply the principle of least privilege for each environment. The main branch federated credential is used to update the container images in the container registry.

In the left menu, under the Settings menu heading, select Properties. Copy the Tenant Id to use later.

In the left menu, go to Overview page of the User-assigned managed identity.

Copy the values for Client ID and Subscription ID to use later.

Create GitHub Secrets
In the GitHub Repository, go to Settings.

In the left menu, under the Security and quality heading, select Secrets and variables > Actions.

Click New repository secret.

Create the following secrets:

AZURE_CLIENT_ID: The Client ID of the User-assigned managed identity
AZURE_SUBSCRIPTION_ID: The Subscription ID of the Azure subscription
AZURE_TENANT_ID: The Directory (tenant) ID of the Azure AD tenant
Note: These secrets will be used by the Azure Login Action in the GitHub actions workflow.

Set up GitHub Actions Workflows
Click the Actions tab in the GitHub Repository.

Select the link to set up a workflow yourself.

Replace the contents of the workflow file with the following code:

name: Build and Release
on:
  push:
    branches: [ main ]
    paths-ignore:
      - 'charts/simple-grocery-store/values/**'

jobs:
  build-and-push:
    permissions:
      id-token: write # Required write permission to Fetch an OIDC token
      contents: read # Read permission to access the repository contents
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    - name: Azure login
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    - name: Azure CLI script
      uses: azure/cli@v2
      with:
        azcliversion: latest
        inlineScript: |
          # Set required variables
          RG=$(az group list --query [].name --output tsv)
          ACR=$(az acr list --resource-group $RG --query [].name --output tsv)
          cd frontend
          az acr build --registry "$ACR" --image "frontend:${GITHUB_RUN_NUMBER}" .
          cd ../services/cart-service
          az acr build --registry "$ACR" --image "cart-service:${GITHUB_RUN_NUMBER}" .
          cd ../product-service
          az acr build --registry "$ACR" --image "product-service:${GITHUB_RUN_NUMBER}" .
          
  release-and-test-staging:
    needs: build-and-push
    permissions:
      id-token: write # Required write permission to Fetch an OIDC token
      contents: write # Write permission to access the repository contents
      issues: write # Write permission to access the repository issues
    runs-on: ubuntu-latest
    environment: staging
    env:
      POLL_INTERVAL: 60 # Number of times to poll for an updated image in seconds
      MAX_ATTEMPTS: 20 # Number attempts to wait for an image update, should be greater than Flux poll interval
    steps:

    - name: Azure login
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        
    - name: Kubectl tool installer
      uses: Azure/setup-kubectl@v4.0.0

    - name: Configure AKS Credentials
      run: |
          RG=$(az group list --query [].name --output tsv)
          AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
          az aks get-credentials --resource-group $RG --name $AKS --overwrite-existing
    
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Update Helm Release tags
      run: |
        sed -E "/frontend:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
        sed -E "/cartService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
        sed -E "/productService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
        git config --global user.name "GitHub Action"
        git config --global user.email "action@github.com"
        git add charts/simple-grocery-store/values/staging.yaml
        git commit -m "Update HelmRelease image tags to ${GITHUB_RUN_NUMBER}"
        git fetch origin
        git rebase origin/main
        git push

    - name: Wait for Image Tag to Match Run Number
      run: |
          TARGET_TAG="${{ github.run_number }}"
          echo "Polling for Deployment spec tag to match '$TARGET_TAG'..."
          attempt=0
          while [ $attempt -lt ${{ env.MAX_ATTEMPTS }} ]; do
            FULL_IMAGE_SPEC=$(kubectl get deployment frontend-deployment -n staging -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
            CURRENT_TAG="${FULL_IMAGE_SPEC##*:}"
            
            if [ -z "$FULL_IMAGE_SPEC" ]; then
              echo "Attempt $attempt/${{ env.MAX_ATTEMPTS }}: Deployment spec not retrievable. Retrying..."
            elif [ "$CURRENT_TAG" = "$TARGET_TAG" ]; then
              echo "Match found: Image tag is '$CURRENT_TAG'."
              exit 0
            else
              echo "Attempt $attempt/${{ env.MAX_ATTEMPTS }}: Current tag is '$CURRENT_TAG'. Retrying in ${{ env.POLL_INTERVAL }} seconds..."
            fi
            
            attempt=$((attempt + 1))
            sleep ${{ env.POLL_INTERVAL }}
          done
          
          echo "Timeout: Tag did not match '$TARGET_TAG' after $(( ${{ env.POLL_INTERVAL }} * ${{ env.MAX_ATTEMPTS }} )) seconds."
          exit 1

    - name: Retrieve Load Balancer IP
      id: get-ip
      run: |
          IP=$(kubectl get service frontend-service -n staging -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
          if [[ -z "$IP" ]]; then
            echo "Error: IP not available."
            exit 1
          fi
          echo "IP retrieved: $IP"
          echo "ip=$IP" >> $GITHUB_OUTPUT
          TARGET_URL="http://$IP"  # Adjust protocol/port as needed (e.g., http if not HTTP)
          echo "target_url=$TARGET_URL" >> $GITHUB_OUTPUT
          
    - name: ZAP Scan
      if: steps.get-ip.outputs.ip != ''
      uses: zaproxy/action-baseline@v0.14.0
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        target: ${{ steps.get-ip.outputs.target_url }}

  release-production:
    needs: release-and-test-staging
    permissions:
      id-token: write # Required write permission to Fetch an OIDC token
      contents: write # Write permission to access the repository contents
    runs-on: ubuntu-latest
    environment: production
    steps:
    - name: Azure login
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        
    - name: Kubectl tool installer
      uses: Azure/setup-kubectl@v4.0.0
      
    - name: Configure AKS Credentials
      run: |
          RG=$(az group list --query [].name --output tsv)
          AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
          az aks get-credentials --resource-group $RG --name $AKS --overwrite-existing

    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Update Helm Release tags
      run: |
        sed -E "/frontend:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
        sed -E "/cartService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
        sed -E "/productService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
        git config --global user.name "GitHub Action"
        git config --global user.email "action@github.com"
        git add charts/simple-grocery-store/values/production.yaml
        git commit -m "Update HelmRelease image tags to ${GITHUB_RUN_NUMBER}"
        git fetch origin
        git rebase origin/main
        git push
Note: This code is checking out the repository, logging in using the credentials we just created, and running an Azure CLI script to update the container images in the registry. It then releases the application to the staging environment and runs the Zed Attack Proxy (ZAP) scan against that environment. Then, with our approval, it will release to production.

Click Commit changes...

Provide a commit message if required. For example, Add CI/CD workflow for build and release process.

Click Commit changes

Review the Issues and Release to Production
Click the Actions tab in the GitHub Repository.

Select the latest workflow run.

Wait for the workflow to complete the release-and-test-staging stage.

Click the Issues tab in the GitHub Repository.

Review any issues created by the ZAP scan in the staging environment.

Note: In a production environment you could implement any checks that your workload requires. From there you could carefully review and action the findings in the report, for simplicity in the lab, we will skip that step.

Go to the Actions tab in the GitHub Repository.

Select the latest workflow run.

Click Review deployments.

Select the checkbox next to production.

Click Approve and deploy.

Conclusion
Congratulations — you've completed this hands-on lab!# Deliver AKS Workloads Securely with GitOps and Helm
