Apigee Hybrid CI/CD with Argo Workflows
This repository contains the source code for Apigee Hybrid API proxies and a CI/CD pipeline definition using Argo Workflows. It is designed to deploy proxies to Apigee Hybrid on Google Cloud using Workload Identity (keyless authentication) for enhanced security.

📂 Project Structure
Plaintext

.
├── apigee-deploy.yaml          # The Argo Workflow definition (CI/CD Pipeline)
└── src
    └── gateway
        ├── hello-world-mock    # The API Proxy source code
        │   ├── apiproxy        # Standard Apigee proxy structure
        │   └── pom.xml         # Child POM (inherits config from parent)
        └── parent-pom          # Central Maven configuration
            └── pom.xml         # Defines plugins, auth settings, and profiles
🚀 Prerequisites
Before running the pipeline, ensure you have the following installed on your GKE cluster:

Apigee Hybrid: Installed and running in your Kubernetes cluster.

Argo Workflows: Installed in the argo namespace.

Bash

kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.5/install.yaml
Argo CLI: (Optional, for easier management)

Bash

curl -sLO https://github.com/argoproj/argo-workflows/releases/latest/download/argo-linux-amd64.gz
gunzip argo-linux-amd64.gz && chmod +x argo-linux-amd64 && sudo mv argo-linux-amd64 /usr/local/bin/argo
🔐 Setup: Workload Identity (Keyless Auth)
This project uses GKE Workload Identity so the pipeline can deploy without managing JSON key files.

1. Create Google Service Account (GSA)
Create a GSA that has the Apigee Environment Admin role.

Bash

gcloud iam service-accounts create apigee-deployer
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member "serviceAccount:apigee-deployer@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
    --role "roles/apigee.environmentAdmin"
2. Create Kubernetes Service Account (KSA)
Create the Service Account in the argo namespace that the workflow will use.

Bash

kubectl create serviceaccount argo-workflow -n argo
3. Bind GSA to KSA
This step allows the Kubernetes Service Account to impersonate the Google Service Account.

Bash

# 1. Allow the KSA to impersonate the GSA
gcloud iam service-accounts add-iam-policy-binding apigee-deployer@YOUR_PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:YOUR_PROJECT_ID.svc.id.goog[argo/argo-workflow]"

# 2. Annotate the KSA
kubectl annotate serviceaccount argo-workflow -n argo \
    iam.gke.io/gcp-service-account=apigee-deployer@YOUR_PROJECT_ID.iam.gserviceaccount.com
⚙️ Configuration (Maven)
The deployment logic is handled by the Parent POM (src/gateway/parent-pom/pom.xml).

API Version: v1

Auth Type: oauth (Bearer Token)

Target: https://apigee.googleapis.com

If you need to change the organization or environment globally, update the <properties> section in src/gateway/parent-pom/pom.xml.

🏃‍♂️ How to Run the Pipeline
Option 1: Using Argo CLI (Recommended)
This method allows you to watch the build logs in real-time.

Bash

# Submit the workflow and watch the logs
argo submit -n argo apigee-deploy.yaml --watch

Option 2: Using kubectl
If you do not have the Argo CLI installed:

Bash
# Submit the workflow and watch the logs
argo submit -n argo apigee-deploy.yaml --watch

# Create the workflow
kubectl create -f apigee-deploy.yaml -n argo

# Find the pod name
kubectl get pods -n argo --watch

# View logs
kubectl logs -n argo -f <pod-name> -c main
🛠 Local Development
You can also deploy directly from your local machine using Maven, provided you have gcloud installed and authenticated.

Bash

cd src/gateway/hello-world-mock

mvn clean install \
  -Dbearer="$(gcloud auth print-access-token)" \
  -Dapigee.options=override
📄 Argo Workflow Details (apigee-deploy.yaml)
The workflow performs the following steps:

Clone: Pulls the latest code from the main branch.

Environment Prep: Installs the Google Cloud SDK inside the build container.

Authentication: Generates an OAuth token using the Workload Identity binding.

Deploy: Executes the mvn install command to package and upload the proxy.
