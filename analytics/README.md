
curl http://checkip.amazonaws.com # get public ip


# build the image 

docker build -t my-flask-app .

docker run --env-file .env -p 5000:5000 my-flask-app



```sh
# run the serviecs
docker compose up -d --build

# sql migration

## This installs the psql CLI tool, without the PostgreSQL server.
sudo apt update
sudo apt install postgresql-client -y
psql --version # e.g.,  psql (PostgreSQL) 16.9


## you may need to run: export PGPASSWORD=mypass
psql -h localhost -U myuser -d mydatabase -f ../db/1_create_tables.sql
psql -h localhost -U myuser -d mydatabase -f ../db/2_seed_users.sql
psql -h localhost -U myuser -d mydatabase -f ../db/3_seed_tokens.sql



## inspect the tables manually:
psql -h localhost -U myuser -d mydatabase

\dt
\q
```


# deployment =================================== #

## ECR

```sh
aws sts get-caller-identity # you can get ACCOUNT_ID from here

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com


# create an ECR repository
aws ecr create-repository --repository-name flask-app-ecr --region us-east-1

# tag the image
```
cd analytics
docker build -t my-flask-app .
docker tag my-flask-app:latest <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/flask-app-ecr:v1
docker push <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/flask-app-ecr:v1

# don't forget to specify the image in the deployment.yaml
```

```sh
# Automate the above commands:
aws_account_id=123456789012
region=us-east-1
repo_name=flask-app

# Authenticate
aws ecr get-login-password --region $region | docker login --username AWS --password-stdin $aws_account_id.dkr.ecr.$region.amazonaws.com

# Create repo
aws ecr create-repository --repository-name $repo_name --region $region

# Tag
docker tag flask-app:latest $aws_account_id.dkr.ecr.$region.amazonaws.com/$repo_name:latest

# Push
docker push $aws_account_id.dkr.ecr.$region.amazonaws.com/$repo_name:latest
```



# push the image to ecr
```

## EKS

```sh
# create the cluster
eksctl create cluster -f ./k8s/cluster-config.yaml

# apply the k8s manifests
kubectl apply -f k8s/flask-app.yaml


# delete pods with a specific labels:
kubectl delete pods -l app=flask-app

```


## Postgresql

### AWS EBS CSI driver 
```sh
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm upgrade --install aws-ebs-csi-driver \
  aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=true \
  --set controller.serviceAccount.name=ebs-csi-controller-sa \
  --set node.serviceAccount.create=true \
  --set node.serviceAccount.name=ebs-csi-node-sa

# Check EBS CSI Pods Are Running
kubectl get pods -n kube-system -l "app.kubernetes.io/name=aws-ebs-csi-driver,app.kubernetes.io/instance=aws-ebs-csi-driver"

aws iam list-policies --scope AWS --query "Policies[?contains(PolicyName, 'EBS')].[PolicyName,Arn]" --output table
aws iam attach-role-policy \
  --role-name eksctl-flask-app-nodegroup-flask-a-NodeInstanceRole-JqJ8tMrJHRKv \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy


```


```sh
kubectl get pvc postgres-pvc

```
