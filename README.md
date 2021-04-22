Attempting to get this to work with:

```terraform
/*==== Variables used across all modules ======*/

variable "environment" {
}

variable "name" {
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-2"
}
/*==== Variables used across all modules ======*/
locals {
  vpc_cidr             = "10.0.0.0/16"
  environment          = "qa"
  public_subnet_cidrs  = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
  private_subnet_cidrs = ["10.0.60.0/24", "10.0.61.0/24", "10.0.62.0/24"]
  availability_zones   = ["us-east-2a", "us-east-2b", "us-east-2c"]
  cluster_name         = "${var.environment}-${var.name}-eks-cluster"
  region               = "us-east-2" // TODO: sync with provider or use same root var
  domain               = "example.com"
  # max_size = 1
  # min_size = 1
  # desired_capacity = 1
  # instance_type = "t2.micro"
  # ecs_aws_ami = "ami-0254e5972ebcd132c"

}

/* Setup vpc pub/private network model */
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = var.name
  cidr = local.vpc_cidr

  azs             = local.availability_zones
  private_subnets = local.private_subnet_cidrs
  public_subnets  = local.public_subnet_cidrs

  enable_nat_gateway = true
  enable_vpn_gateway = true

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}

/* Create security groups */
// TODO: these are not correct
resource "aws_security_group" "main-node" {
  name        = "terraform-eks-main-node"
  description = "Security group for all nodes in the cluster"
  vpc_id      = module.vpc.vpc_id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group_rule" "main-node-ingress-self" {
  type              = "ingress"
  description       = "Allow node to communicate with each other"
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.main-node.id
  to_port           = 65535
  cidr_blocks       = local.private_subnet_cidrs
}

// TODO: maybe redundant from eks module
resource "aws_security_group_rule" "main-node-ingress-cluster" {
  type                     = "ingress"
  description              = "Allow worker Kubelets and pods to receive communication from the cluster control plane"
  from_port                = 1025
  protocol                 = "tcp"
  security_group_id        = aws_security_group.main-node.id
  source_security_group_id = aws_security_group.main-node.id # TODO: aws_security_group.eks.id
  to_port                  = 65535
}

/*Setup intial empty EKS cluster*/
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = local.cluster_name
  cluster_version = "1.19"
  subnets         = module.vpc.private_subnets

  tags = {
    Environment = var.environment
  }

  vpc_id = module.vpc.vpc_id

  workers_group_defaults = {
    root_volume_type = "gp2"
  }

  worker_groups = [
    {
      name                          = "worker-group-1"
      instance_type                 = "t3.small"
      additional_userdata           = "echo foo bar"
      asg_desired_capacity          = 1
      additional_security_group_ids = [aws_security_group.main-node.id]
    },
    {
      name                          = "worker-group-2"
      instance_type                 = "t3.medium"
      additional_userdata           = "echo foo bar"
      additional_security_group_ids = [aws_security_group.main-node.id]
      asg_desired_capacity          = 1
    },
  ]

  # worker_additional_security_group_ids = [aws_security_group.all_worker_mgmt.id]
  # map_roles                            = var.map_roles
  # map_users                            = var.map_users
  # map_accounts                         = var.map_accounts
}

/*
Setup Local dev info -  This info provides a kubeconfig and related information for connecting to the eks cluster locally.

aws configure
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)
*/
output "cluster_id" {
  description = "EKS cluster ID."
  value       = module.eks.cluster_id
}

output "cluster_endpoint" {
  description = "Endpoint for EKS control plane."
  value       = module.eks.cluster_endpoint
}

output "cluster_security_group_id" {
  description = "Security group ids attached to the cluster control plane."
  value       = module.eks.cluster_security_group_id
}

output "kubectl_config" {
  description = "kubectl config as generated by the module."
  value       = module.eks.kubeconfig
}

output "config_map_aws_auth" {
  description = "A kubernetes configuration to authenticate to this EKS cluster."
  value       = module.eks.config_map_aws_auth
}

output "cluster_name" {
  description = "Kubernetes Cluster Name"
  value       = local.cluster_name
}

output "region" {
  description = "AWS region"
  value       = local.region
}

output "kubeconfig-certificate-authority-data" {
  value = data.aws_eks_cluster.cluster.certificate_authority[0].data
}

/* Setup kuberenetes provider for use in tf */
data "aws_eks_cluster" "cluster" {
  name = module.eks.cluster_id
}

data "aws_eks_cluster_auth" "cluster" {
  name = module.eks.cluster_id
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
  load_config_file       = false
}

provider "helm" {
  kubernetes {
    host                   = data.aws_eks_cluster.cluster.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
    token                  = data.aws_eks_cluster_auth.cluster.token
    load_config_file       = false
  }
}


module "alb-ingress-controller" {
  // source = "github.com/GSA/terraform-kubernetes-aws-load-balancer-controller"
  source = "../../../../Projects/terraform-kubernetes-aws-load-balancer-controller"
  providers = {
    kubernetes = kubernetes,
    helm       = helm
  }

  k8s_cluster_type = "eks"
  k8s_namespace    = "kube-system"

  aws_region_name           = local.region
  k8s_cluster_name          = data.aws_eks_cluster.cluster.name
  alb_controller_depends_on = [module.alb.target_group_arns]
  target_groups             = local.target_groups2 //module.alb.target_group_arns
}



data "aws_route53_zone" "selected" {
  name = local.domain
}

resource "aws_route53_record" "k8dash" {
  zone_id = data.aws_route53_zone.selected.zone_id
  name    = "k8dash.${data.aws_route53_zone.selected.name}"
  type    = "CNAME"
  ttl     = "300"
  records = [module.alb.this_lb_dns_name]
}

locals { // TODO: merge
  target_groups = [
    {
      # name_prefix      = "pref-"
      name             = module.kubernetes_dashboard.kubernetes_dashboard_service_name
      backend_protocol = "HTTP"
      backend_port     = 80
      target_type      = "ip"

    }
  ]
  target_groups2 = [
    {
      # name_prefix      = "pref-"
      name             = module.kubernetes_dashboard.kubernetes_dashboard_service_name
      backend_protocol = "HTTP"
      backend_port     = 80
      target_type      = "ip"
      target_group_arn = module.alb.target_group_arns[0]
    }
  ]
}

module "alb" {
  source = "terraform-aws-modules/alb/aws"
  # version = "~> 5.0"

  name = "my-alb"

  load_balancer_type = "application"

  vpc_id          = module.vpc.vpc_id
  subnets         = module.vpc.public_subnets
  security_groups = [aws_security_group.main-node.id]

  # access_logs = {
  #   bucket = "my-alb-logs"
  # }

  target_groups = local.target_groups

  http_tcp_listeners = [
    {
      port               = 80
      protocol           = "HTTP"
      target_group_index = 0
    }
  ]

  tags = {
    Environment = "Test"
  }
}

// TODO: helm repo add eks https://aws.github.io/eks-charts && kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"


module "kubernetes_dashboard" {
  source  = "cookielab/dashboard/kubernetes"
  version = "0.9.0"

  kubernetes_namespace_create = true
  kubernetes_dashboard_csrf   = "babblblb" // TODO::
  # kubernetes_resources_labels =
}

```




# Terraform module: AWS Load Balancer Controller installation

This Terraform module can be used to install the [AWS Load Balancer
Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller)
into a Kubernetes cluster.

## Improved integration with Amazon Elastic Kubernetes Service (EKS)

This module can be used to install the AWS Load Balancer controller into a "vanilla" Kubernetes cluster (which is the default)
or it can be used to integrate tightly with AWS-managed [EKS](https://aws.amazon.com/eks/) clusters which allows the deployed pods to
[use IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html).

It is required that an OpenID connect provider [has already been created](https://www.terraform.io/docs/providers/aws/r/eks_cluster.html#example-iam-role-for-eks-cluster) for your EKS cluster for this feature to work.

Just make sure that you set the variable `k8s_cluster_type` to `eks` type if running on EKS.

## Examples

### EKS deployment

To deploy the AWS Load Balancer Controller into an EKS cluster, use the following
snippet as an example.

```hcl
locals {
   # Your AWS EKS Cluster ID goes here.
  "k8s_cluster_name" = "my-k8s-cluster"
}

data "aws_region" "current" {}

data "aws_eks_cluster" "target" {
  name = local.k8s_cluster_name
}

data "aws_eks_cluster_auth" "aws_iam_authenticator" {
  name = data.aws_eks_cluster.target.name
}

provider "kubernetes" {
  alias = "eks"
  host                   = data.aws_eks_cluster.target.endpoint
  token                  = data.aws_eks_cluster_auth.aws_iam_authenticator.token
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.target.certificate_authority[0].data)
  load_config_file       = false
}

provider "helm" {
  alias = "eks"
  kubernetes {
    host                   = data.aws_eks_cluster.target.endpoint
    token                  = data.aws_eks_cluster_auth.aws_iam_authenticator.token
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.target.certificate_authority[0].data)
  }
}

module "alb_controller" {
  source  = "iplabs/alb-controller/kubernetes"
  version = "3.4.0"

  providers = {
    kubernetes = "kubernetes.eks",
    helm       = "helm.eks"
  }

  k8s_cluster_type = "eks"
  k8s_namespace    = "kube-system"

  aws_region_name  = data.aws_region.current.name
  k8s_cluster_name = data.aws_eks_cluster.target.name
}
```
