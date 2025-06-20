


```sh
this_custer=flask-app
AWS_ACC_ID=$(aws sts get-caller-identity --query Account --output text) # 717546795560
# get OICD provider

aws eks describe-cluster --name $this_cluster \
  --query "cluster.identity.oidc.issuer" --output text
## https://oidc.eks.us-east-1.amazonaws.com/id/1DB6BA2BBB0696F76C1DF1F5563524D7


# save as `trust.json`
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::717546795560:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:default:argocd-image-access"
        }
      }
    }
  ]
}

# e.g., 
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::717546795560:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/1DB6BA2BBB0696F76C1DF1F5563524D7"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/1DB6BA2BBB0696F76C1DF1F5563524D7:sub": "system:serviceaccount:default:argocd-image-access"
        }
      }
    }
  ]
}

# create IAM role & attach policy
aws iam create-role \
  --role-name ArgoCDECRRole \
  --assume-role-policy-document file://trust.json

aws iam create-policy \
  --policy-name ECRAccessPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ],
        "Resource": "*"
      }
    ]
  }'


aws iam attach-role-policy \
  --role-name ArgoCDECRRole \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/ECRAccessPolicy


# Create Kubernetes ServiceAccount with IRSA annotation

kubectl apply -f argocd-irsa-sa.yaml


## add this to app deployment, in spec.

serviceAccountName: argocd-image-access

```
