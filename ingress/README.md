# Ingress Controller — AWS Load Balancer Controller

Sets up the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
so that Kubernetes `Ingress` objects provision real AWS Application Load
Balancers (ALBs). This directory does **not** create any AWS resources by
itself — the commands below are what you run once the EKS cluster (see the
top-level `deployment.yml`) already exists.

## Files

| File | Purpose |
|------|---------|
| `iam-policy.json` | Official AWS IAM policy required by the controller (fetched from the upstream project). |
| `ingress-class.yaml` | `IngressClass` named `alb`, set as the cluster default. |
| `sample-app.yaml` | Minimal demo `Deployment` + `Service` to route traffic to. |
| `sample-ingress.yaml` | Example `Ingress` that provisions a public ALB routing `/` to the demo app. |

## Prerequisites

- The EKS cluster from the repo root must already exist (`eksctl create
  cluster -f ../deployment.yml`), with `iam.withOIDC: true` (already set).
- [`helm`](https://helm.sh/docs/intro/install/) installed locally.

  ```bash
  brew install helm
  ```

## Step-by-step

1. **Create the IAM policy** the controller needs (one-time, per AWS account):

   ```bash
   aws iam create-policy \
     --policy-name AWSLoadBalancerControllerIAMPolicy \
     --policy-document file://ingress/iam-policy.json
   ```

   Note the returned `Arn` (or construct it as
   `arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy`).

2. **Create the IRSA service account** that binds that policy to a Kubernetes
   `ServiceAccount` via OIDC:

   ```bash
   eksctl create iamserviceaccount \
     --cluster my-eks-cluster \
     --region us-east-1 \
     --namespace kube-system \
     --name aws-load-balancer-controller \
     --role-name AmazonEKSLoadBalancerControllerRole \
     --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
     --approve
   ```

   > Alternative: uncomment the `iamServiceAccounts` block in `../deployment.yml`
   > (after step 1) and this gets created automatically the first time you
   > run `eksctl create cluster -f deployment.yml`, instead of running this
   > as a separate step.

3. **Install the controller via Helm:**

   ```bash
   helm repo add eks https://aws.github.io/eks-charts
   helm repo update eks

   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
     -n kube-system \
     --set clusterName=my-eks-cluster \
     --set region=us-east-1 \
     --set vpcId=<VPC_ID> \
     --set serviceAccount.create=false \
     --set serviceAccount.name=aws-load-balancer-controller
   ```

   Get `<VPC_ID>` with:

   ```bash
   aws eks describe-cluster --name my-eks-cluster --query "cluster.resourcesVpcConfig.vpcId" --output text
   ```

4. **Verify the controller is running:**

   ```bash
   kubectl get deployment -n kube-system aws-load-balancer-controller
   kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
   ```

5. **Apply the `IngressClass`:**

   ```bash
   kubectl apply -f ingress/ingress-class.yaml
   ```

6. **Deploy the demo app and its Ingress:**

   ```bash
   kubectl apply -f ingress/sample-app.yaml
   kubectl apply -f ingress/sample-ingress.yaml
   ```

7. **Get the ALB address** (takes a minute or two to provision):

   ```bash
   kubectl get ingress hello-app-ingress -n default
   ```

   Once `ADDRESS` is populated, `curl` it (or open in a browser) to see
   `Hello from the EKS cluster!`.

## Tearing it down

```bash
kubectl delete -f ingress/sample-ingress.yaml
kubectl delete -f ingress/sample-app.yaml
helm uninstall aws-load-balancer-controller -n kube-system
eksctl delete iamserviceaccount --cluster my-eks-cluster --namespace kube-system --name aws-load-balancer-controller
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

(Do this before `eksctl delete cluster`, and only after removing any
Ingress-provisioned ALBs — deleting the cluster first can orphan the ALB.)
