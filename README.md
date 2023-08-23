# EKS Custom Networking with Karpenter

This example demonstrates how to provision an EKS cluster with Karpenter and default addons. Once deployed, we will enable the EKS Custom Networking feature of VPC CNI and migrate the existing workloads.

This example solution provides:

- Amazon EKS Cluster (control plane)
- Amazon EKS Managed Node group 
- Amazon EKS managed addons `coredns`, `vpc-cni` and `kube-proxy`
- Karpenter
- sample deployment is provided to demonstrates scaling a deployment to view how Karpenter responds to provision, and de-provision, resources on-demand

## Prerequisites:

Ensure that you have the following tools installed locally:

1. [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
2. [kubectl](https://Kubernetes.io/docs/tasks/tools/)
3. [terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)

## Deploy

To provision this example:

```sh
terraform init
terraform apply -target module.vpc
terraform apply -target module.eks
terraform apply
```

Enter `yes` at command prompt to apply

## Enable the custom networking

Provision the secondary cidr & subnets by uncommenting `secondary_cidr_blocks` and `private_subnets` lines and comment the existing `private_subnets` under `module vpc`

```yaml
  secondary_cidr_blocks = [local.secondary_vpc_cidr]
  ....
  # private_subnets = [for k, v in local.azs : cidrsubnet(local.vpc_cidr, 4, k)]
  private_subnets = concat(
    [for k, v in local.azs : cidrsubnet(local.vpc_cidr, 4, k)],
    [for k, v in local.azs : cidrsubnet(local.secondary_vpc_cidr, 2, k)]
  )
```

Run the terraform vpc module

```sh
terraform apply -target module.vpc -auto-approve
```

Configure the `VPC CNI Addon` to enable the `custom networking` and install `ENIConfig` resources:


Add configuration values to vpc_cni in main.tf:

```yaml
  vpc-cni    = {
    # Specify the VPC CNI addon should be deployed before compute to ensure
    # the addon is configured before data plane compute resources are created
    # See README for further details
    before_compute = true
    most_recent    = true # To ensure access to the latest settings provided
    configuration_values = jsonencode({
      env = {
        # Reference https://aws.github.io/aws-eks-best-practices/reliability/docs/networkmanagement/#cni-custom-networking
        AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG = "true"
        ENI_CONFIG_LABEL_DEF               = "topology.kubernetes.io/zone"
      }
    })
  }
```

Uncomment below lines in main.tf to create `ENIConfig` resources
```yaml
  resource "kubectl_manifest" "eni_config" {
    for_each = zipmap(local.azs, slice(module.vpc.private_subnets, 3, 6))
  
    yaml_body = yamlencode({
      apiVersion = "crd.k8s.amazonaws.com/v1alpha1"
      kind       = "ENIConfig"
      metadata = {
        name = each.key
      }
      spec = {
        securityGroups = [
          module.eks.node_security_group_id,
        ]
        subnet = each.value
      }
    })
  }
```

Deploy the terraform code

```sh
terraform apply -auto-approve
```

At this point, all required VPC infrastructure and CNI configuration is applied to the cluster. We will move the pods to new nodes to take this into effect.

Add new Managed node group:

```yaml
  eks_managed_node_groups = {
    initial = {
      instance_types = ["m5.large", "m6a.large", "m6i.large"]

      min_size     = 2
      max_size     = 10
      desired_size = 2
    },
    mng1 = {
      instance_types = ["m5.large", "m6a.large", "m6i.large"]

      min_size     = 2
      max_size     = 10
      desired_size = 2
    }
  }
```

```sh
terraform apply -auto-approve
```

Delete existing Managed node group:

```yaml
  eks_managed_node_groups = {
    mng1 = {
      instance_types = ["m5.large", "m6a.large", "m6i.large"]

      min_size     = 2
      max_size     = 10
      desired_size = 2
    }
  }
```

```sh
terraform apply -auto-approve
```

Verify the pods:

```sh
kubectl get pods -A -o wide
```

```text
  
```

You have successfully enabled the Custom Networking in your EKS cluster.


## Destroy

To teardown and remove the resources created in this example:

```sh
kubectl delete deployment inflate
terraform destroy -target="module.eks_blueprints_addons" -auto-approve
terraform destroy -target="module.eks" -auto-approve
terraform destroy -auto-approve
```
