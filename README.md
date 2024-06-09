# AWS-EKS-Container-Deploy
This project focuses on building an EKS cluster using EC2 or Fargate and deploying a containerized gaming application into it.
The application is a Java gaming application packaged as a docker container and pushed to Amazon ECR repo. In this project, i have provisioned the kubernetes environment in EKS using AWS cloudformation.

## Prerequisites 
  1. create and AWS account and set up and IAM user with neccesary priviledge to create, administer and describe kubernetes and related infrastructure.
  2. kubectl – A command line tool for working with Kubernetes clusters. For more information, see Installing or updating kubectl [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
  3. eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see Installing or updating. 
  4. AWS CLI – A command line tool for working with AWS services, including Amazon EKS. Install this on your local machine [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html), and configure it to access your AWS user profile with,
     ```
     aws configure
     ```
  5. If the kubernetes will run on EC2 and not with EKS, then you need to set up the networking environment. to achieve this, you will need to,
     * create a VPC and subnet
     * security group and define the inbound and outbound rules.
     * open access to port 80 from anywhere (internet)
     * attach the security gropu to the eks worker nodes ( or EC2 instance group or auto scaling group).
     * create and attach and internet gateway to your VPC, ensuring that the route table is updated with destination of all traffic to internet (0.0.0.0/0).
     * create an appropriate policy which is attached to a role to be assumed by the worker nodes of the EKS.
     * When launching your EKS worker nodes, specify the IAM role ARN (Amazon Resource Name) of the IAM role that includes the necessary IAM policy. The IAM role allows the worker nodes to authenticate with the EKS cluster and access AWS resources based on the permissions defined in the attached IAM policy.
    
  6. for EKS cluster, the worker node can be ec2 instance (node group) with appropriate configuration define or fargate with fargate profile.

## Install EKS

Please follow the prerequisites doc before this.

### Install using Fargate

```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

### Install using EC2

```
eksctl create cluster \
--name demo-cluster \       
--version 1.29 \
--region us-east-1 \
--nodegroup-name linux-nodes \
--node-type t2.micro \
--nodes 3
```

### Delete the cluster

```
eksctl delete cluster --name demo-cluster --region us-east-1
```
### Create Fargate profile
the namespace specified here will be created in the deployement file. 

```
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### Deploy the deployment, service and Ingress

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
### use the kubectl and the AWS CLI to update the kubeconfig file:
     ```
     aws eks update-kubeconfig --name your-cluster-name
     ```
   - Verify the configuration by running a kubectl command against your EKS cluster:
     ```
     kubectl get nodes
     ```

## Check if there is an IAM OIDC provider configured already

- aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4\n 

If not, run the below command

```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```



### How to setup alb add on
this part is necessary to congigure the ingress. the ingress is necessary to route traffic from the internet to the application running in the pods. the ingress parameters are included in the deploymenmtb yaml which is only made manifest with the ALB controller. thus, the ALB controller needs to be set up and attached with IAM policy following the steps below.

Download IAM policy

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

Create IAM Policy

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Create IAM Role

```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## Deploy ALB controller
this is done using helm. you might need to install helm, if not alreasy installed. follow the steps below to configure the rest of the helm to read the eks cluster. 
Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
```

Update the repo

```
helm repo update eks
```

Install
this creates the ALB and attaches it to the eks with the ingress rule.
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

Verify that the deployments are running.

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```


the application can then be accessed from the web browser using the ALB address. see screen shots below. 


![Screenshot 2023-08-03 at 7 57 15 PM](https://github.com/iam-veeramalla/aws-devops-zero-to-hero/assets/43399466/93b06a9f-67f9-404f-b0ad-18e3095b7353)



     
