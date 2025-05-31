# 🚀 Bits-Bank AWS EKS Cluster Setup Guide

This guide helps you recreate and clean up your **Bits-Bank Project EKS infrastructure** using `eksctl`, `kubectl`, and `Helm`. It includes steps for:
- Provisioning EKS
- Installing ALB Ingress Controller
- Deploying Kubernetes Dashboard
- Configuring IAM
- Managing Route53 and ACM

---

## 🛠️ Prerequisites
- AWS CLI configured (`aws configure`)
- IAM permissions to create roles, policies, and EKS resources
- A domain registered (e.g., from GoDaddy)

---

## 📦 1. Install `eksctl`
```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
```

---

## ☁️ 2. Create EKS Cluster
```bash
eksctl create cluster --name bits-bank-cluster --region ap-south-1 --nodes 3 --node-type t3.micro --managed
```

Verify:
```bash
eksctl get cluster --name bits-bank-cluster
```

---

## ❌ 3. Delete EKS Cluster (if needed)
```bash
eksctl delete cluster --name bits-bank-cluster
```

---

## 🔐 4. Set Up IAM OIDC Provider
```bash
eksctl utils associate-iam-oidc-provider --region=ap-south-1 --cluster=bits-bank-cluster --approve
```

---

## 🛡️ 5. Create IAM Service Account for ALB Ingress
```bash
eksctl create iamserviceaccount \
  --cluster bits-bank-cluster \
  --region ap-south-1 \
  --namespace kube-system \
  --name alb-ingress-controller \
  --attach-policy-arn arn:aws:iam::481665107730:policy/bits-bank-eks-policy \
  --attach-policy-arn arn:aws:iam::481665107730:policy/bits-bank-cluster-policy-1 \
  --override-existing-serviceaccounts \
  --approve
```

---

## 🌐 6. Install AWS ALB Ingress Controller via Helm
```bash
helm repo add eks https://aws.github.io/eks-charts

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=bits-bank-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=alb-ingress-controller \
  --set region=ap-south-1 \
  --set vpcId=vpc-0a9d05516b2de00de \
  --set ingressClass=alb
```

---

## 📊 7. Install Kubernetes Dashboard
```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace \
  --namespace kubernetes-dashboard

kubectl create serviceaccount dashboard-admin-sa -n kubernetes-dashboard

kubectl create clusterrolebinding dashboard-admin-sa \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:dashboard-admin-sa

kubectl -n kubernetes-dashboard create token dashboard-admin-sa

kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard-kong-proxy 8443:443
```

---

## 📈 8. Scale Nodegroup (Optional)
```bash
eksctl scale nodegroup \
  --cluster bits-bank-cluster \
  --name <node-group> \
  --nodes 10 \
  --nodes-min 8 \
  --nodes-max 12
```
```bash
eksctl get nodegroup --cluster bits-bank-cluster --region ap-south-1 --name ng-0de9252c
```

List nodes:
```bash
kubectl get nodes -o custom-columns="NAME:.metadata.name,PODS:.status.allocatable.pods"
```

---

## 🌍 9. Route53 Setup
If your domain (e.g., `bitsbank-project.site`) is registered elsewhere (e.g., GoDaddy), update **name servers** in GoDaddy to point to AWS Route53.

Then, create these records in Route53:

| Type | Name                     | Value (Target)                                |
|------|--------------------------|-----------------------------------------------|
| A    | api.bitsbank-project.site | ALB DNS name (from Ingress/ELB)               |
| A    | www.bitsbank-project.site | Amplify domain target (or CNAME)              |


---

## 🔒 10. ACM Certificate
1. Go to **AWS ACM** → Request Public Certificate
2. Use domain: `api.bitsbank-project.site`
3. Add CNAME record in Route53 for DNS validation
4. Use issued certificate ARN in your Ingress annotations

```yaml
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:XXXXXXX
```

---

## 🔁 Next Steps
- Apply all manifests:
  ```bash
  kubectl apply -f k8s-manifests/
  ```
- Push updates via CI/CD
- Clean up using `eksctl delete cluster` when done

---

> 💡 Tip: Back up this guide and related IAM policies, Helm values, manifests, and credentials securely to reproduce infra anytime.

