# Install Crossplane

## Configure authentication
Upbound supports authentication AWS via access key, service account or AssumeRole.

Authentication method with `ProviderConfig`, applied to the `Provider`.

In this case, we use `Service Accounts`.

#### Authenticate using IAM Roles for Service Accounts
AWS EKS can use IAM Roles for Service Accounts (IRSA) for authenticate the AWS Provider.

First, create IAM Role and Policy for use authen in the AWS Provide.

```yaml
module "eks" {
    source = "terraform-aws-modules/eks/aws"
    ...
}
data "aws_caller_identity" "current" {}
data "aws_iam_policy_document" "assume" {
  statement {
    actions = [ "sts:AssumeRoleWithWebIdentity" ]
    effect = "Allow"
    condition {
      test = "StringLike"
      values = [ "system:serviceaccount:upbound-system:provider-aws-*" ]
      variable = "${module.eks.oidc_provider}:sub"
    }
    condition {
      test     = "StringLike"
      variable = "${module.eks.oidc_provider}:aud"
      values = ["sts.amazonaws.com"]
    }
    principals {
      identifiers = [module.eks.oidc_provider_arn]
      type        = "Federated"
    }
  }
}

resource "aws_iam_role" "crossplane" {
  name_prefix = "${var.tags["Service"]}-${var.tags["System"]}-${var.environment}-crossplane-"
  path        = "/"
  description = "IAM Role for Crossplane"
  assume_role_policy    = data.aws_iam_policy_document.assume[0].json
  tags = merge(var.tags, {
    Name = "${var.tags["Service"]}-${var.tags["System"]}-${var.environment}-crossplane"
  })
}

resource "aws_iam_policy" "CrossplaneControllerPolicy" {
  name        = "AWSGSCrossplanePolicy-${random_string.default.result}"
  path        = "/"
  description = "Policy for clossplane resource provider"
  
  ## Recommend Setting policy according to the desired use
  policy      = templatefile("${path.module}/policys/crossplane_policy.json", {})
}

resource "aws_iam_role_policy_attachment" "CrossplaneControllerPolicy" {
  role       = aws_iam_role.crossplane.name
  policy_arn = aws_iam_policy.CrossplaneControllerPolicy.arn
}
```

## Install the provider-aws
### Install Crossplane in EKS
#### Upbound Universal Crossplane
First, install Upbound Up commad-line. Up CLI is simplifies configuration and management of Upbound Universal Crossplane (UXP).

Install `up` with the command:
```sh
curl -sL "https://cli.upbound.io" | sh
```

More information the Upbound command-line is in the [Upbound Up Document](https://docs.upbound.io/reference/cli/).

UXP is the Upbound official of Crossplane for self-hostes control planes.

Next to, Install UXP into Kubernetes cluster:
```sh
up uxp install
```
More infomation use Upbound command for uxp in the [Upbound UXP documentation](https://docs.upbound.io/uxp/).
#### Helm - Upbound Universal Crossplane

### Create ControllerConfig

First, create controller. `ControllerConfig` create setting used by `Provider` deployment. For IRSA, the `ControllerConfig` provides an `annotation` of the ARN of role use with Kubernetes Service Account.

```yaml
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: aws-config
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::000000000000:role/eks-test-role
spec:
  podSecurityContext:
    fsGroup: 2000
```
After that, apply the `CrotrollerConfig` and verify the installation with `kubectl get crontrollerconfig`
```
$ kubectl get controllerconfig
NAME                    AGE
aws-config              6s
```

### Create a ProviderConfig
Next, create the `ProviderConfig` for explicitly configures the official AWS Provider-Family to use `IRSA` authentication.

This creates `Provider : provider-family-aws` version of the image `provider-family-aws` according to version Controller of the Upbound Universal Crossplane.

Ex. helm version or UXP version.
```
helm 
```