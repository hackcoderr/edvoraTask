# EdvoraTask
1. Create a Kubernetes cluster with 2 node pools; 2 nodes each.
2. Install cert-manager, ingress-nginx and configure them.
3. Setup a Terraform script to deploy the container in a separate namespace.
4. Configure the deployment to autoscale at 60% memory usage and 70% CPU usage.
5. Create a PVC for the deployment with 10GB space and attach.

**Note: I am setting up this lab on Azure cloud.**
## 1. Create a Kubernetes cluster with 2 node pools; 2 nodes each.

**Note: I am unable to add one more node pool in this cluster due to resourse limit on my azure account so I have generated my request for increase resourse limit on my account but I added the command ``az aks nodepool add --resource-group GitHubActionsRunners --cluster-name GitHubActionsRunnersK8sCluster --name mynodepool --node-count 2`` in my installation script to add node pool. so as the resourse limit of my account will be increse, I will run this command to add one more node pool.**

**Rightnow My cluster is having 1 node pool with 2 nodes.**

:warning: *All the below assumes you are running `Bash`.*

### Install az cli

```bash
# Refresh packages
apt-get update
apt-get upgrade

# From:
# https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt
sudo apt-get update
sudo apt-get install ca-certificates curl apt-transport-https lsb-release gnupg

# Download the microsoft signing keys
curl -sL https://packages.microsoft.com/keys/microsoft.asc |
    gpg --dearmor |
    sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null

# Add the Azure CLI software repository:
AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" |
    sudo tee /etc/apt/sources.list.d/azure-cli.list

# Update repository information and install the azure-cli package:
sudo apt-get update
sudo apt-get install azure-cli
```

### Install kubectl (latest stable version)

```bash
# Download the latest release 
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Download the kubectl checksum file:
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# Validate the kubectl binary against the checksum file:
echo "$(<kubectl.sha256)  kubectl" | sha256sum --check

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client

# Install auto-completion
# OPTIONAL
# -
sudo apt-get install bash-completion
source /usr/share/bash-completion/bash_completion
```

### Install Helm

```bash

# Add signing keys
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -

# Install dependencies
sudo apt-get install apt-transport-https --yes

# Add repository
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update

# Install helm
sudo apt-get install helm

# Verify
helm version

```

### Setup AKS via Azure CLI

```bash
# Authenticate with Azure CLI
az login

# list regions with az
az account list-locations

# Create a resource group for our AKS cluster
az group create --name GitHubActionsRunners --location westeurope

# Get list of resources in the resource group
az group show --resource-group GitHubActionsRunners

# Verify Microsoft.OperationsManagement and Microsoft.OperationalInsights 
# are registered on your subscription.
az provider show -n Microsoft.OperationsManagement -o table
az provider show -n Microsoft.OperationalInsights -o table

# Create AKS cluster in resource group
# --name cannot exceed 63 characters and can only contain letters, 
# numbers, or dashes (-).
az aks create \
  --resource-group GitHubActionsRunners \
  --name GitHubActionsRunnersK8sCluster \
  --enable-addons monitoring \
  --node-count 1 \
  --generate-ssh-keys

###############################################################################
# Access K8s cluster
###############################################################################

# Configure kubectl to connect to your Kubernetes cluster
# Downloads credentials and configures the Kubernetes CLI to use them.
  # Uses ~/.kube/config, the default location for the Kubernetes configuration 
  # file. Specify a different location for your Kubernetes configuration file 
  # using --file.
az aks get-credentials \
  --resource-group GitHubActionsRunners \
  --name GitHubActionsRunnersK8sCluster

# Verify
kubectl config get-contexts
# AND
kubectl get nodes

###############################################################################
# Manually scaling nodes
###############################################################################
# add node pool
az aks nodepool add \
    --resource-group GitHubActionsRunners \
    --cluster-name GitHubActionsRunnersK8sCluster \
    --name mynodepool \
    --node-count 2

# varify 
az aks nodepool list --resource-group GitHubActionsRunners --cluster-name GitHubActionsRunnersK8sCluster

# Scale down
# (OPTIONAL)
az aks scale \
  --resource-group GitHubActionsRunners \
  --name GitHubActionsRunnersK8sCluster \
  --node-count 1

# Check progress
watch -n 2 kubectl get nodes

```

## 2. Install cert-manager, ingress-nginx and configure them

### Install Cert-Manager on Kubernetes

- Add Helm repo
```bash
helm repo add jetstack https://charts.jetstack.io
```

- Update Helm
```bash
helm repo update
```

- Search for latest verion
```bash
helm search repo cert-manager
```

- Install Helm chart
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.6.0 \
  --set prometheus.enabled=false \
  --set installCRDs=true
```

### Ingress-nginx on Kubenetes cluster

- Clone the Ingress Controller repo:
```
git clone https://github.com/nginxinc/kubernetes-ingress.git --branch v2.2.0
```
- Change your working directory to /deployments/helm-chart:
```
cd kubernetes-ingress/deployments/helm-chart
```
- Adding the Helm Repository
```
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
```

- Installing via Helm Repository
```
helm install my-release nginx-stable/nginx-ingress
```
- Installing Using Chart Sources
```
helm install my-release .
```
- Upgrading the Chart
```
kubectl apply -f crds/
```
**Note:** The following warning is expected and can be ignored: Warning: ``kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply``.


- Upgrade Using Chart Sources:
```
helm upgrade my-release .
```
- Upgrade via Helm Repository:
```
helm upgrade my-release nginx-stable/nginx-ingress
```
## 3. Setup a Terraform script to deploy the container in a separate namespace

- Install [terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli) on your K8S cluster.

- Configure Our Kubernetes Provider.
```
provider "kubernetes" {
  config_path    = "~/.kube/config"
}
```
- Now run ``terraform init`` for download the required plugins.

- Create namespace on k8s cluster.
```
resource "kubernetes_namespace" "ns" {
  metadata {
    name = "terrafrom-namespace"
  }
}
```

- Create deployment
```

resource "kubernetes_deployment" "terradeploy" {
  metadata {
    name = "terraform-deployment"
    namespace = kubernetes_namespace.ns.metadata.0.name
    labels = {
      test = "terraApp"
    }
  }

  spec {
    replicas = 1
    selector {
      match_labels = {
        test = "terraApp"
      }
    }

    template {
      metadata {
        labels = {
          test = "terraApp"
        }
      }

      spec {
        container {
          image = "vad1mo/hello-world-rest"
          name  = "terra-hello"
        resources {
            limits = {
              cpu    = "0.5"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "50Mi"
            }
          }

        }
      }
    }
  }
}

```
- Run ``terraform apply --auto-approve`` to deployment.

## 4. Configure the deployment to autoscale at 70% CPU usage.
- Create autoscaling
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: hello-scale
  namespace: terraform-namespace
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: terraform-deployment
  targetCPUUtilizationPercentage: 70
```
- Now run ``kubectl apply -f edvora/autoscale/autoscale.yaml``.

## 5. Create a PVC for the deployment with 10GB space and attach

- Create an index.html file on your Node.
```
sudo mkdir /mnt/data
sudo sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
```
- Create a PersistentVolume
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```
run the below command.
```
kubectl apply -f edvora/pvc/pv-volume.yaml
```
View information about the PersistentVolume:
```
kubectl get pv task-pv-volume
```
- Create a PersistentVolumeClaim
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
Run the below command to create PVC.
```
kubectl apply -f edvora/pvc/pv-claim.yaml
```
Look at the PersistentVolumeClaim:
```
kubectl get pvc task-pv-claim
```
- Create a Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```
Run the below commad to create pod.
```
kubectl apply -f edvora/pvc/pv-pod.yaml
```
Verify that the container in the Pod is running;
```
kubectl get pod task-pv-pod
```
Get a shell to the container running in your Pod:
```
kubectl exec -it task-pv-pod -- /bin/bash
```
In your shell, verify that nginx is serving the index.html file from the hostPath volume:

- Be sure to run these 3 commands inside the root shell that comes from
- running "kubectl exec" in the previous step
```
apt update
apt install curl
curl http://localhost/
```
