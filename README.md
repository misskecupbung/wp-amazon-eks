# WordPress on Amazon EKS

Deploy WordPress on Amazon Elastic Kubernetes Service (EKS) with Amazon EFS for persistent storage and Amazon RDS for the database.

## Architecture

- **Compute**: Amazon EKS (Kubernetes)
- **Storage**: Amazon EFS (Elastic File System) for WordPress files
- **Database**: Amazon RDS (MySQL/Aurora)
- **Load Balancing**: AWS Load Balancer (via Kubernetes Service)

## Prerequisites

- AWS Account with appropriate permissions
- Amazon EKS cluster (running)
- Amazon RDS MySQL/Aurora database (running)
- Amazon EFS file system with access points configured
- EFS CSI Driver installed on EKS

## Install CLI Tools

### Linux

\`\`\`bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
\`\`\`

### macOS

\`\`\`bash
# Install AWS CLI
brew install awscli
aws --version

# Install eksctl
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
eksctl version

# Install kubectl
brew install kubectl
kubectl version --client

# Install Helm
brew install helm
\`\`\`

## Configure AWS Access

\`\`\`bash
# Configure AWS credentials
aws configure

# Update kubeconfig for your EKS cluster
aws eks update-kubeconfig --region <REGION> --name <CLUSTER_NAME>

# Verify cluster access
kubectl get nodes
\`\`\`

## Install EFS CSI Driver

\`\`\`bash
# Add the EFS CSI driver Helm repository
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update

# Install the driver
helm upgrade --install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system
\`\`\`

## Deploy WordPress

### 1. Update Configuration

Before deploying, update the following values in `wordpress-eks.yaml`:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `volumeHandle` | EFS File System ID and Access Point ID | `fs-xxxxxx::fsap-xxxxxx` |
| `WORDPRESS_DB_HOST` | RDS endpoint | `mydb.xxxxx.rds.amazonaws.com` |
| `WORDPRESS_DB_USER` | Database username | `admin` |
| `password` (Secret) | Base64 encoded DB password | `echo -n 'yourpassword' \| base64` |

### 2. Apply Kubernetes Manifests

\`\`\`bash
kubectl apply -f wordpress-eks.yaml
\`\`\`

### 3. Verify Deployment

\`\`\`bash
# Check pod status
kubectl get pods -l app=wordpress

# Check service and get external URL
kubectl get svc wordpress

# View logs
kubectl logs -l app=wordpress -f
\`\`\`

### 4. Access WordPress

Get the LoadBalancer URL:

\`\`\`bash
kubectl get svc wordpress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
\`\`\`

Open the URL in your browser to complete the WordPress installation.

## Cleanup

\`\`\`bash
kubectl delete -f wordpress-eks.yaml
\`\`\`

## Troubleshooting

### Pod stuck in Pending state
- Check EFS mount targets are in the same VPC/subnets as EKS nodes
- Verify EFS security group allows NFS traffic (port 2049) from EKS nodes

### Database connection errors
- Verify RDS security group allows traffic from EKS nodes
- Check database credentials in the Secret

### EFS mount failures
- Ensure EFS CSI driver is installed: `kubectl get pods -n kube-system -l app=efs-csi-controller`
- Check EFS access point permissions