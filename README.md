# NiFiKop AWS Deployment Guide

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Testing & Verification](#testing--verification)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Security Considerations](#security-considerations)

## Prerequisites

### Required Tools
```bash
# Check tool versions
terraform version    # Should be >= 1.0
kubectl version     # Should be >= 1.21
aws --version       # Should be >= 2.0
helm version        # Should be >= 3.0
```

### AWS Configuration
1. Configure AWS CLI with appropriate credentials:
```bash
aws configure
# Enter your:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region (e.g., us-west-2)
# - Default output format (json)
```

2. Ensure your AWS account has necessary permissions:
- AmazonEKSClusterPolicy
- AmazonEKSServicePolicy
- AmazonEKSVPCResourceController
- AmazonEKSWorkerNodePolicy
- AmazonEKS_CNI_Policy
- AmazonEC2ContainerRegistryReadOnly

## Installation

1. Clone the repository:
```bash
git clone <repository-url>
cd nifi-kop-aws
```

2. Make the deployment script executable:
```bash
chmod +x scripts/deploy.sh
```

## Project Structure

```
nifi-kop-aws/
├── terraform/
│   ├── main.tf                 # Main Terraform configuration
│   ├── variables.tf            # Variable definitions
│   ├── outputs.tf             # Output definitions
│   ├── vpc.tf                 # VPC configuration
│   ├── eks.tf                 # EKS cluster configuration
│   ├── security-groups.tf     # Security group configurations
│   └── providers.tf           # Provider configurations
├── kubernetes/
│   ├── nifi-kop-operator.yaml # NiFiKop operator deployment
│   └── nifi-cluster.yaml      # NiFi cluster configuration
├── scripts/
│   └── deploy.sh              # Deployment automation script
└── README.md                  # This file
```

## Configuration

1. Update Terraform variables (terraform/variables.tf):
```bash
# Create a terraform.tfvars file
cat << EOF > terraform/terraform.tfvars
aws_region = "us-west-2"
cluster_name = "nifikop-cluster"
vpc_cidr = "10.0.0.0/16"
private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
public_subnets = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
eks_node_instance_types = ["t3.xlarge"]
eks_node_desired_capacity = 3
EOF
```

2. Review and adjust Kubernetes configurations:
- kubernetes/nifi-kop-operator.yaml
- kubernetes/nifi-cluster.yaml

## Deployment

1. Initialize and deploy the infrastructure:
```bash
./scripts/deploy.sh
```

2. Monitor deployment progress:
```bash
# Check EKS cluster status
aws eks describe-cluster --name nifikop-cluster --region us-west-2

# Check node status
kubectl get nodes

# Check NiFiKop operator status
kubectl get pods -n nifi
```

## Testing & Verification

1. Verify EKS Cluster:
```bash
# Check cluster info
kubectl cluster-info

# Check node status and readiness
kubectl get nodes
kubectl describe nodes

# Check system pods
kubectl get pods -n kube-system
```

2. Verify NiFiKop Operator:
```bash
# Check operator deployment
kubectl get deployments -n nifi
kubectl get pods -n nifi
kubectl logs -n nifi deployment/nifikop

# Check operator CRDs
kubectl get crds | grep nifi
```

3. Verify NiFi Cluster:
```bash
# Check NiFi cluster status
kubectl get nificlusters -n nifi
kubectl describe nificluster nifi -n nifi

# Check NiFi pods
kubectl get pods -n nifi -l app=nifi
```

4. Access NiFi UI:
```bash
# Get LoadBalancer URL
kubectl get svc -n nifi

# The URL will be in the EXTERNAL-IP column of the LoadBalancer service
```

## Monitoring

1. View pod logs:
```bash
# NiFiKop operator logs
kubectl logs -f deployment/nifikop -n nifi

# NiFi node logs (replace pod-name)
kubectl logs -f pod/nifi-0 -n nifi
```

2. Monitor resources:
```bash
# Check pod resource usage
kubectl top pods -n nifi

# Check node resource usage
kubectl top nodes
```

3. Check pod events:
```bash
kubectl get events -n nifi --sort-by='.lastTimestamp'
```

## Troubleshooting

1. Pod issues:
```bash
# Check pod status
kubectl get pods -n nifi
kubectl describe pod <pod-name> -n nifi

# Check pod logs
kubectl logs <pod-name> -n nifi
```

2. Operator issues:
```bash
# Check operator status
kubectl describe deployment nifikop -n nifi

# Check operator logs
kubectl logs deployment/nifikop -n nifi
```

3. Node issues:
```bash
# Check node status
kubectl describe node <node-name>

# Check node logs (requires AWS CloudWatch)
aws logs get-log-events --log-group-name /aws/eks/nifikop-cluster/cluster \
    --log-stream-name <node-name>
```

4. Common Issues:
- Insufficient resources:
  ```bash
  kubectl describe pods -n nifi | grep -i "insufficient"
  ```
- Image pull errors:
  ```bash
  kubectl describe pods -n nifi | grep -i "imagepull"
  ```
- Volume mount issues:
  ```bash
  kubectl describe pods -n nifi | grep -i "volume"
  ```

## Cleanup

1. Delete NiFi resources:
```bash
# Delete NiFi cluster
kubectl delete nificluster nifi -n nifi

# Delete NiFiKop operator
kubectl delete -f kubernetes/nifi-kop-operator.yaml
```

2. Delete Kubernetes resources:
```bash
# Delete namespace
kubectl delete namespace nifi

# Verify resources are deleted
kubectl get all -n nifi
```

3. Destroy AWS infrastructure:
```bash
# Navigate to Terraform directory
cd terraform

# Destroy infrastructure
terraform destroy -auto-approve

# Verify AWS resources
aws eks list-clusters --region us-west-2
aws ec2 describe-vpcs --region us-west-2
```

## Security Considerations

1. Network Security:
- All pods run in private subnets
- Control plane access is restricted by security groups
- Worker nodes communicate through internal network only

2. Access Control:
- Use AWS IAM roles for service accounts (IRSA)
- Implement RBAC policies for Kubernetes resources
- Keep kubeconfig files secure

3. Data Security:
- Enable encryption at rest for EBS volumes
- Use AWS KMS for secret encryption
- Implement network policies to control pod communication

4. Monitoring and Auditing:
- Enable AWS CloudWatch logs
- Enable EKS control plane logging
- Monitor security group changes

## Support

For issues related to:
- NiFiKop: [NiFiKop GitHub Issues](https://github.com/Orange-OpenSource/nifikop/issues)
- EKS: [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- Infrastructure: Create an issue in this repository