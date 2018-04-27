# kops-dump

```
aws s3api create-bucket --bucket microdc-kops-preprod --region eu-west-1
aws s3api put-bucket-versioning --bucket microdc-kops-preprod  --versioning-configuration Status=Enabled
export NAME=$(basename $(pwd))
export AWS_SDK_LOAD_CONFIG=1
kops create cluster --authorization RBAC --topology private --networking weave --node-count 3 --zones eu-west-1a,eu-west-1b,eu-west-1c --node-size m4.2xlarge --master-zones eu-west-1a,eu-west-1b,eu-west-1c --master-size m4.large --kubernetes-version 1.7.9 ${NAME}
echo 'add this to the cluster config edit below:
  kubeAPIServer:
    authorizationRbacSuperUser: admin
    runtimeConfig:
      batch/v2alpha1: "true"
  additionalPolicies:
    master: |
      [
        {
          "Effect": "Allow",
          "Action": [
            "route53:GetHostedZone",
            "route53:ChangeResourceRecordSets",
            "route53:ListResourceRecordSets"
          ],
          "Resource": ["arn:aws:route53:::hostedzone/*"]
        }
      ]
    node: |
      [
        {
          "Effect": "Allow",
          "Action": ["sts:AssumeRole"],
          "Resource": "*"
        }
      ]
'
kops edit cluster ${NAME}
kops update cluster ${NAME} --yes
# we need this but it's not working: aws ec2 describe-subnets --query "Subnets[?Tags[?Key=='Name'&&"'!'"contains(Value,'utility')]&&[?Key=='KubernetesCluster'&&Value=='"$NAME"']].SubnetId" --output text
# so run this instead:
aws ec2 describe-subnets --query "Subnets[?Tags[?Key=='Name'&&"'!'"contains(Value,'utility')]].SubnetId" --output text | xargs aws ec2 create-tags --tags "Key=kubernetes.io/role/internal-elb,Value=true" --resources
```
