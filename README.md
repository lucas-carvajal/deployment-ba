# deployments-ba
Deployment Files for Bachelor Thesis 
"Cloud-Native Software Quality: Evaluating A
Practical Example"

### Disclaimer
For the ECS deployment, the health checks for the Customer Self Service Backend and the Risk Management Server still fail
(possibly due to the networking being misconfigured for the ActiveMQ TCP connection) and need some more work.

For the EKS deployment, the connection to the frontend via the ingresses still does not work as expected and needs some more work.



## ECS Stack
cd into the `/ecs` directory, then:
#### Deploy
```
aws cloudformation create-stack \
	--stack-name lakeside \
	--template-body file://./main.yml \
	--capabilities CAPABILITY_NAMED_IAM
````

#### Delete
```
aws cloudformation delete-stack \
  --stack-name lakeside
```

#### Connect to Bastion Host EC2 Instance
In the AWS Console, go to EC2 -> Key Pairs and find the Bastion KeyPair.

Download the private key and by using the Bastion KeyPair's ID:  
`aws ssm get-parameter --name /ec2/keypair/key-0cc8f98192bf6689bREPLACE --with-decryption --query Parameter.Value --output text > new-key-pair.pem`

Then in the AWS Console, got to EC2 -> Instances, find the Bastion Host Instance, 
click on the "Connect" button and follow the instructions.


## EKS
Create EKS cluster
`eksctl create cluster --name lakeside --version 1.27 --without-nodegroup`

Add node group (or create fargate profiles as described below)
```bash
eksctl create nodegroup \
  --cluster lakeside \
  --region eu-central-1 \
  --name graviton-mng \
  --node-type m6g.medium \
  --nodes 4\
  --nodes-min 2\
  --nodes-max 6\
  --managed
```

Create fargate profiles for lakeside and kubeconfig namespaces (if node groups are not added)
```bash
eksctl create fargateprofile \
    --cluster lakeside \
    --name lakeside-profile \
    --namespace lakeside
```

```bash
eksctl create fargateprofile \
    --cluster lakeside \
    --name kube-system-profile \
    --namespace kube-system
```

Update kubeconfig file  
`aws eks update-kubeconfig --region eu-central-1 --name lakeside`

Create lakeside namespace  
`kubectl create namespace lakeside`

#### Set up logging support
reference: https://docs.aws.amazon.com/eks/latest/userguide/fargate-logging.html

Create observability namespace  
`kubectl apply -f eks/other/observability-namespace.yml `

Apply Cloudwatch ConfigMap  
`kubectl apply -f eks/other/aws-logging-cloudwatch-configmap.yml`

Download Cloudwatch IAM policy  
`curl -O https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/cloudwatchlogs/permissions.json`

Create IAM logging policy  
`aws iam create-policy --policy-name eks-fargate-logging-policy --policy-document file://permissions.json`

Attach IAM policy to pod execution policy (replace accountId and execution role if needed)  
```bash
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::063725538836:policy/eks-fargate-logging-policy \
  --role-name eksctl-lakeside-cluster-FargatePodExecutionRole-1XINX5ONUOU0C
```

#### Set up LoadBalancer Controller Add-On
for automatically creating LoadBalancers for services of type LoadBalancer  
reference: https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

Create IAM policy for LoadBalancer Controller  
`curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json`

Create IAM Policy for LoadBalancer Controller
```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Associate OIDC Provider with Cluster  
`eksctl utils associate-iam-oidc-provider --region=eu-central-1 --cluster=lakeside --approve`

Create IAM Role for LoadBalancer Controller (replace accountId and execution role if needed)  
```bash
eksctl create iamserviceaccount \
  --cluster=lakeside \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::063725538836:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

Install LoadBalancer Controller Add-On (replace vpcId with the correct one)  
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=lakeside \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=eu-central-1 \
  --set vpcId=vpc-0ee1381d230a56b8c-TODO
```

#### Apply Kubernetes Manifests to Cluster
```bash
kubectl apply -f eks/
```

