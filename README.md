# AWS Load Balancer Controller Installation and Ingress Deployment

This document consolidates and streamlines the AWS Load Balancer Controller installation process and the Ingress deployment steps for an EKS cluster. The eksctl-related steps have been omitted as per the updated workflow.

---

## Prerequisites

- **AWS CLI** installed and configured.
- **kubectl** installed and configured to interact with your EKS cluster.
- **Helm** installed (v3.0 or later).

---

## Steps

### Step 1: Create an IAM OIDC Provider

```bash
aws eks update-kubeconfig --region <aws-region> --name <your-cluster-name>
aws eks describe-cluster --name <your-cluster-name> --query "cluster.identity.oidc.issuer" --output text
```

Use the OIDC URL obtained to set up the IAM OIDC provider.

### Step 2: Create an IAM Policy

```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam-policy.json
```

### Step 3: Create a Service Account

Using Helm, we configure and set up the service account directly:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations.eks\.amazonaws\.com/role-arn="arn:aws:iam::<your-aws-account-id>:role/<your-role-name>" \
  --set region=<aws-region> \
  --set vpcId=<vpc-id> \
  --set enableWaf=false \
  --set enableWafv2=false
```

### Step 4: Verify the Installation

Check the deployment status:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Ingress Deployment

### Prerequisites for Ingress

Ensure the following roles and policies are set up in AWS:

- Cluster Role with `eks-cluster-policy`.
- Node Group Role with the following policies:
  - `AmazonEC2ContainerRegistryReadOnly`
  - `AmazonEC2WorkerNodePolicy`
  - `AmazonEKS_CNI_Policy`
- Ingress Role with appropriate JSON policy (as described in IAM setup).

### Steps

1. Update **ingress.yaml** with the appropriate Security Group from the EKS networking section.
2. Deploy the services:

```bash
kubectl apply -f ./order-service
kubectl apply -f ./product-catalog
kubectl apply -f ./shopping-cart
kubectl apply -f ingress.yaml
```

3. Verify the ingress:

```bash
kubectl get ing
kubectl describe ing <ingress-name>
```

4. Validate the Load Balancer:

   - Open the Load Balancer created by the Ingress Controller.
   - Check the rules and health checks.
   - Use the DNS name of the Load Balancer to access the services. Append paths like `/products` or `/orders` as needed.

---

## Troubleshooting

1. Check pods in the kube-system namespace:

```bash
kubectl get pods -n kube-system
kubectl logs <pod-name> -n kube-system
```

2. Reapply configurations if needed:

```bash
kubectl delete -f ingress.yaml
kubectl apply -f ingress.yaml
```

3. Ensure the subnet tags are correct for the Load Balancer.

---

This guide ensures a smooth workflow for deploying and managing AWS Load Balancer Controller and ingress resources without eksctl.

