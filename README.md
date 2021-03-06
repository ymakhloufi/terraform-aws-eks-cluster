## Setup

1. run `terraform apply`
2. Configure k8s context:
    - `aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)`
3. Activate [CNI Prefix Delegation](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html) (
   Not automatable due to: [Issue#1](https://github.com/aws/amazon-vpc-cni-k8s/issues/1571)
   , [Issue #2](https://github.com/aws/containers-roadmap/issues/1333)):
    - `kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true`
4. Replace EKS Node-Groups to propagate Prefix Delegation:
    - `terraform apply -replace="module.eks.module.eks.module.eks_managed_node_group[\"$(terraform output -raw cluster_node_group_name)\"].aws_eks_node_group.this[0]"`
5. (optional) Install Metrics server (required for CPU/Mem based HPAs):
    - `kubectl apply -f k8s/metrics-server.yaml`

### K8S Dashboard (optional)

- install dashboard `kubectl apply -f k8s/dashboard`
- generate auth token for
  dashboard: `kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user-token | awk '{print $1}') | grep token: | sed 's/token:.* ey/ey/'`
- proxy to dashboard: `kubectl proxy`
- Login to
  Dashboard: http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
- and paste auth-token from above.

### Example App with ALB-Ingress

1. Run `kubectl apply -f k8s/sample-app.yaml` to deploy the app
2. Run `kubectl get ingress/ingress-2048 -n game-2048` to ensure the ALB got created and the ingress has an **ADDRESS**
3. Wait about a minute or two for the Load-Balancer to fire up.
4. Copy/paste the **ADDRESS** from the output into your browser (http, not https) - and done.

## Caution
Remember to `kubectl delete` all apps that use an ALB-ingress before destroying the cluster with Terraform. Otherwise
the remainders that were created by the ALB-controller (`./module/eks/alb-controller.tf`) will prevent the VPC from
being destroyed. If you messed it up, you have to manually delete the following resources (check region) and
re-run `terraform destroy`:

- EC2 Target
  Groups: [https://console.aws.amazon.com/ec2/v2/home?#TargetGroups:tag:elbv2.k8s.aws/cluster=*](https://console.aws.amazon.com/ec2/v2/home?#TargetGroups:tag:elbv2.k8s.aws/cluster=*)
- EC2 Load
  Balancers: [https://console.aws.amazon.com/ec2/v2/home?#LoadBalancers:tag:elbv2.k8s.aws/cluster=*](https://console.aws.amazon.com/ec2/v2/home?#LoadBalancers:tag:elbv2.k8s.aws/cluster=*)
- VPC Security
  Groups: [https://console.aws.amazon.com/vpc/home?#securityGroups:tag:ingress.k8s.aws/resource=ManagedLBSecurityGroup](https://console.aws.amazon.com/vpc/home?#securityGroups:tag:ingress.k8s.aws/resource=ManagedLBSecurityGroup)
