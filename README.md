# ROSA PrivateLink

ROSA private clusters with PrivateLink can be deployed via the ROSA CLI, however, they must be deployed into an existing VPC.

This repository provides AWS CloudFormation templates that create a default VPC for ROSA along with other AWS networking infrastructure resources required to support a [private Red Hat OpenShift on AWS (ROSA) cluster with AWS PrivateLink](https://aws.amazon.com/blogs/containers/red-hat-openshift-service-on-aws-private-clusters-with-aws-privatelink/).

## Deployment

### Pre-requisites

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [ROSA CLI](https://github.com/openshift/rosa/releases)
- [jq](https://stedolan.github.io/jq/download/0)

### Step 1: AWS CloudFormation

Available AWS CloudFormation templates:

| #   | Setup               | Description                                                           | Architecture                                                                                      | Multi-AZ | CloudFormation template                                                                |
| --- | ------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | -------- | -------------------------------------------------------------------------------------- |
| 1   | PrivateLink cluster | Uses a TransitGateay attached to a ROSA Private VPC and an Egress VPC | [rosa-privatelink-egress-vpc.single-subnet](assets/rosa-privatelink-egress-vpc.single-subnet.png) | **No**   | [rosa-privatelink-egress-vpc.single-az.yml](rosa-privatelink-egress-vpc.single-az.yml) |

Update the following command to launch of the CloudFormation template above, using the AWS CLI:

```bash
# AWS region to install OpenShift
export AWS_DEFAULT_REGION=us-west-2 
# AWS CloudFormation Stack name
export AWS_STACK_NAME=rosa-networking
# AWS profile
export AWS_PROFILE=default
# AWS account ID
export AWS_ACCOUNT_ID=<xxx>

$ aws cloudformation create-stack --stack-name $AWS_STACK_NAME --template-body file://rosa-privatelink-egress-vpc.single-az.yml
```

### Step 2: ROSA - initialisation

Once step 1 is completed, the ROSA configuration and account roles can be created with the following commands:

```bash
# OpenShift version to install
export OPENSHIFT_VERSION=4.9.15
# AWS permission boundary. 
export PERMISSION_BOUNDARY_ARN=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AccountBoundary 
# Name of the cluster
export ROSA_CLUSTER_NAME=rosa-cluster
# Prefix assigned to the role names. Use a unique name per cluster
export ROLE_PREFIX=ManagedOpenShift-${ROSA_CLUSTER_NAME}
# Number of compute nodes
export ROSA_NUM_COMPUTE_NODES=2
# Instance type of the computer nodes
export ROSA_COMPUTE_TYPE=m5.xlarge
export ROSA_HOST_PREFIX=23
# The public and private subnets for OpenShift. These subnets belong to an existing VPC that OpenShift 
# will be using. 
export ROSA_SUBNET_IDS=subnet-0f0c1850850b8e4ab,subnet-00898f202241a075c
# An existing VPC CIDR that OpenShift will be using
export ROSA_VPC_CIDR=10.1.0.0/16
export ROSA_PRIVATE_SUBNET=`aws cloudformation describe-stacks --stack-name $AWS_STACK_NAME --query "Stacks[0].Outputs[?OutputKey=='oRosaVpcSubnet'].OutputValue" --output text`

# Create account roles
rosa create account-roles \
    --version ${OPENSHIFT_VERSION} \
    --mode auto \
    --permissions-boundary ${PERMISSION_BOUNDARY_ARN} \
    --prefix ${ROLE_PREFIX} \
    --profile ${AWS_PROFILE} \
    --region ${AWS_DEFAULT_REGION} \
    --yes
```

### Step 3: ROSA - cluster installation

Create the ROSA private cluster using the following commands:

```bash
# Create ROSA cluster
rosa create cluster \
    --private-link \
    --sts \
    --cluster-name ${ROSA_CLUSTER_NAME} \
    --compute-nodes ${ROSA_NUM_COMPUTE_NODES} \
    --compute-machine-type ${ROSA_COMPUTE_TYPE} \
    --version ${OPENSHIFT_VERSION} \
    --machine-cidr ${ROSA_VPC_CIDR} \
    --subnet-ids ${ROSA_PRIVATE_SUBNET} \
    --host-prefix ${ROSA_HOST_PREFIX} \
    --multi-az \ (optional)
    --additional-trust-bundle-file RootCA.pem \ (optional)
    --http-proxy http://proxy.xxxxxx:3128  \    (optional)
    --https-proxy http://proxy.xxxxx:3128       (optional)

# Create OpenShift operators roles
rosa create operator-roles \
    --cluster ${ROSA_CLUSTER_NAME} \
    --permissions-boundary ${PERMISISON_BOUNDARY_ARN} \
    --prefix ${ROLE_PREFIX} \
    --profile ${AWS_PROFILE} \
    --region ${AWS_DEFAULT_REGION} \
    --mode auto \
    --yes

# Create OpenID Connect (OIDC) provider for OpenShift
rosa create oidc-provider \
    --cluster ${ROSA_CLUSTER_NAME} \
    --profile ${AWS_PROFILE} \
    --region ${AWS_DEFAULT_REGION} \
    --mode auto \
    --yes
```

You must then proceed with the following steps **during** cluster installation.

---

IMPORTANT: If DNS resolution is not configured as per the following steps, the ROSA cluster will fail to be created.This is because the boostrap and provisioned nodes must be able to use internal DNS name resolution in order to resolve OpenShift API endpoints **during** cluster installation.

---

1. Find the current status of the cluster with the `rosa list cluser` command.
2. When the cluster status is *installing*, run the following commands:

```bash
VPC_EGRESS=`aws cloudformation describe-stacks --stack-name $AWS_STACK_NAME --query "Stacks[0].Outputs[?OutputKey=='oEgressVpc'].OutputValue" --output text`
echo "Egress VPC Id: $VPC_EGRESS"
DNS_DOMAIN=$(rosa describe cluster --cluster $ROSA_CLUSTER_NAME -ojson | jq -r .dns.base_domain)
echo "ROSA Cluster Domain Name: $DNS_DOMAIN"

# The following step may fail if the cluster installation has not reached the DNS configuration stage. 
# Please repeat the command until the Route 53 Hosted Zone is found
R53HZ_ID=$(aws route53 list-hosted-zones-by-name | jq --arg name "$ROSA_CLUSTER_NAME.$DNS_DOMAIN." -r '.HostedZones | .[] | select(.Name=="\($name)") | .Id')
echo "ROSA Cluster Route 53 Hosted Zone Id: $R53HZ_ID"
aws route53 associate-vpc-with-hosted-zone --hosted-zone-id $R53HZ_ID --vpc VPCRegion=$AWS_REGION,VPCId=$VPC_EGRESS
```

### Decommissioning

Delete the ROSA cluster and CloudFormation stack by running the following commands:

```bash
rosa delete cluster -c $ROSA_CLUSTER_NAME
aws cloudformation delete-stack --stack-name $AWS_STACK_NAME
```

## Contributions

If you are looking to make your first contribution, follow the steps below.

1. Tests: you will need to install [Docker](https://docs.docker.com/get-docker/) and [cfn-lint](https://github.com/aws-cloudformation/cfn-lint) and verify that the linting and security scanning tests are successful by using the following command: `bash test/run.sh`.
2. Open a Github Pull Request

---

Copyright 2022 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License Version 2.0 (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at <http://www.apache.org/licenses/> or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied. See the License for the specific language governing permissions and limitations under the License.
