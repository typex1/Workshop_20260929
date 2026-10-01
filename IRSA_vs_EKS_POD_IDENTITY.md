## IAM Roles for Service Accounts (IRSA) vs. EKS Pod Identity

**Amazon EKS Pod Identity** is the recommended alternative to IRSA (IAM Roles for Service Accounts) for managing IAM permissions on EKS clusters.

While IRSA relies on OpenID Connect (OIDC) federation, EKS Pod Identity operates natively through an EKS API and a local agent running on your nodes.

---

### Key Architectural Differences

| Feature | IRSA (Older Approach) | EKS Pod Identity (Newer Approach) |
| --- | --- | --- |
| **Authentication Flow** | OIDC Provider Federation via STS (`AssumeRoleWithWebIdentity`) | Direct EKS API via `eks-pod-identity-agent` DaemonSet (`AssumeRole`) |
| **IAM OIDC Setup** | Required per cluster (subject to IAM quota limits) | **None needed** |
| **Trust Policy** | Custom per role, referencing cluster-specific OIDC provider ARN and ServiceAccount names | **Reusable across clusters** using the generic service principal `pods.eks.amazonaws.com` |
| **Association Mapping** | Configured via ServiceAccount annotations (`[eks.amazonaws.com/role-arn](https://eks.amazonaws.com/role-arn)`) | Defined at the **EKS API level** (`aws_eks_pod_identity_association` in IaC) |
| **IAM Session Tags** | Limited / Manual configuration | **Automatic** (includes `eks-cluster-name`, `kubernetes-namespace`, `kubernetes-service-account`) |

---

### Why EKS Pod Identity was Introduced

1. **Eliminates OIDC Infrastructure Management:** With IRSA, every new cluster required an IAM OIDC Provider. Pod Identity manages credential delivery natively.
2. **Role Reusability Across Clusters:** IRSA trust policies contain hardcoded cluster OIDC IDs, making role reuse across staging/prod environments painful. With Pod Identity, you define a single IAM role with a fixed trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}

```

3. **Cleaner Separation of Concerns:** Identity mappings are attached directly to the cluster resource rather than declared solely as annotations inside the Kubernetes cluster.

---

### How to Implement EKS Pod Identity

1. **Install the EKS Add-on:** Deploy the `eks-pod-identity-agent` add-on to your cluster.
2. **Create the IAM Role:** Create your IAM role trusting `pods.eks.amazonaws.com`.
3. **Map the Association:** Bind the IAM role to your Kubernetes Namespace and ServiceAccount via AWS Console, AWS CLI, or IaC (Terraform, CloudFormation):

```bash
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace my-app-ns \
  --service-account my-app-sa \
  --role-arn arn:aws:iam::123456789012:role/MyApplicationRole

```

---

### When Should You Still Use IRSA?

Although EKS Pod Identity is the recommended default for new workloads, IRSA remains fully supported and is still necessary for certain use cases:

* **AWS Fargate:** Because Pod Identity relies on a DaemonSet running on compute nodes, it is not supported on AWS Fargate.
* **Legacy AWS SDKs:** Pod Identity requires relatively modern versions of the AWS SDKs inside your container applications.
* **Non-EKS Kubernetes:** If running self-managed Kubernetes or hybrid setups (like EKS Anywhere) where the EKS Pod Identity control plane API is unavailable.
