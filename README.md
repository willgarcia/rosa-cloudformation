# ROSA PrivateLink

This repository provides AWS CloudFormation templates that the AWS networking infrastructure resources required to support a [private Red Hat OpenShift on AWS (ROSA) cluster with AWS PrivateLink](https://aws.amazon.com/blogs/containers/red-hat-openshift-service-on-aws-private-clusters-with-aws-privatelink/).

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
export AWS_REGION=us-west-2 AWS_STACK_NAME=rosa-networking
aws cloudformation create-stack --stack-name $AWS_STACK_NAME --template-body file://rosa-privatelink-egress-vpc.single-az.yml
```

### Step 2: ROSA - initialisation

Once step 1 is completed, the ROSA cluster can be created with the following commands:

```bash
rosa create account-roles \
    --mode auto \
    --yes
```

### Step 3: ROSA - cluster installation

```bash
export ROSA_CLUSTER_NAME=rosa-cluster
export ROSA_PRIVATE_SUBNET=`aws cloudformation describe-stacks --stack-name $AWS_STACK_NAME --query "Stacks[0].Outputs[?OutputKey=='oRosaVpcSubnet'].OutputValue" --output text`
echo "ROSA private subnet: $ROSA_PRIVATE_SUBNET"

rosa create cluster \
    -y \
    --cluster-name $ROSA_CLUSTER_NAME \
    --private-link \
    --machine-cidr=10.1.0.0/16 \
    --sts \
    --version=4.9.15 \
    --subnet-ids=$ROSA_PRIVATE_SUBNET \
    --region $AWS_REGION

rosa create operator-roles --cluster $ROSA_CLUSTER_NAME
rosa create oidc-provider --cluster $ROSA_CLUSTER_NAME
```

Please, proceed with the following steps **during** cluster creation:

1. Find the current status of the cluster with the `rosa list cluser`.
2. If the cluster status is *installing*, run the following commands:

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

1. Tests

You will to install [Docker](https://docs.docker.com/get-docker/) and [cfn-lint](https://github.com/aws-cloudformation/cfn-lint) and verify that the linting and security scanning tests are successful by using the following command: `bash test/run.sh`.

2. Open a Github Pull Request

---

Copyright 2022 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License Version 2.0 (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at <http://www.apache.org/licenses/> or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied. See the License for the specific language governing permissions and limitations under the License.
