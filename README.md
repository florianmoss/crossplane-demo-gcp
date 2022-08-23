# Crossplane POC

This is just a demo of my Crossplane POC.

## Table of Contents

- [Crossplane POC](#crossplane-poc)
  - [Table of Contents](#table-of-contents)
  - [Requirements](#requirements)
  - [Installation](#installation)
    - [Check Crossplane Status](#check-crossplane-status)
  - [Access](#access)
    - [GCP](#gcp)
  - [GCP Deployment](#gcp-deployment)
    - [Crossplane GCP ProviderConfig](#crossplane-gcp-providerconfig)
  - [Provision GCP resources with Crossplane](#provision-gcp-resources-with-crossplane)
  - [GCP Service Account Key](#gcp-service-account-key)
  - [Wanna run it in Argo?](#wanna-run-it-in-argo)

## Requirements
- A Kubernetes cluster (can also be minikube, kind, k3d etc..)
- Access to GCP
- gcloud cli
- helm cli
- kubectl cli

## Installation

```bash
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane \
        --namespace crossplane-system crossplane-stable/crossplane
```

### Check Crossplane Status
```
helm list -n crossplane-system
kubectl get all -n crossplane-system
```

## Access

### GCP

For this demo, we will authenticate with our user account to GCP and use the temporary Application Default personal token.

:warning: This is not a proper way to configure crossplane in a Production environment (or even Development). Instead, you should use a Service Account Key or run crossplane from a Cluster where the crossplane Pod can use a Service Account with enough privilege to provision the resources you want.

Authenticate to GCP and setup your environment
```
gcloud auth login
gcloud config configurations create demo-dev
gcloud config set project xxx-yyy-zzz 	   # <- CHANGEME
gcloud config set account gcp-user@email.com # <- CHANGEME
gcloud auth application-default login
```

Once you have run the `application-default login` command, a token will be created and saved under your home directory under the following path (MacOS):
```
/Users/{{ USER }}/.config/gcloud/legacy_credentials/{{ GCP_USER_EMAIL))/adc.json
```

You can then create a Kubernetes secret with your token by running
```
kubectl create secret generic gcp-creds -n crossplane-system --from-file=creds=/Users/{{ USER }}/.config/gcloud/legacy_credentials/{{ GCP_USER_EMAIL))/adc.json
```


## GCP Deployment

Install the GCP provider
```
kubectl crossplane install configuration \                                          registry.upbound.io/xp/getting-started-with-gcp:latest
```

### Crossplane GCP ProviderConfig

```
export PROJECT_ID=<Google Cloud Project ID>

echo "apiVersion: gcp.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: ${PROJECT_ID}
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-creds
      key: creds" | kubectl apply -f -

```

Confirm that this worked

```
kubectl get providerconfigs.gcp.crossplane.io
NAME              PROJECT-ID                 AGE
default           inspired-berm-338112       31s
```

## Provision GCP resources with Crossplane

```
kubectl apply -f gcp/
```

The output should look something like this:
```
bucket.storage.gcp.crossplane.io/demo-bucket-338112 created
cluster.container.gcp.crossplane.io/demo-cluster created
nodepool.container.gcp.crossplane.io/demo-nodepool created
network.compute.gcp.crossplane.io/demo-network created
subnetwork.compute.gcp.crossplane.io/demo-subnet created
```

## Wanna run it in Argo?
```
kubectl apply -f https://raw.githubusercontent.com/florianmoss/crossplane-demo-gcp/master/argo-gcp-application.yaml
```

## GCP Service Account Key

```
# replace this with your own gcp project id and the name of the service account
# that will be created.
PROJECT_ID=my-project
NEW_SA_NAME=test-service-account-name

# create service account
SA="${NEW_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud iam service-accounts create $NEW_SA_NAME --project $PROJECT_ID

# enable cloud API
SERVICE="sqladmin.googleapis.com"
gcloud services enable $SERVICE --project $PROJECT_ID

# grant access to cloud API
ROLE="roles/cloudsql.admin"
gcloud projects add-iam-policy-binding --role="$ROLE" $PROJECT_ID --member "serviceAccount:$SA"

# create service account keyfile
gcloud iam service-accounts keys create creds.json --project $PROJECT_ID --iam-account $SA
```
