# cloudformation-ba
Cloudformation Files for Bachelor Thesis

## ECS Stack
#### Deploy
```
aws cloudformation create-stack \
	--stack-name lakeside-test \
	--template-body file://./main.yml \
	--capabilities CAPABILITY_NAMED_IAM
````

#### Delete
```
aws cloudformation delete-stack \
  --stack-name lakeside-test
````

## EKS Stack
#### Deploy
```
aws cloudformation create-stack \
	--stack-name lakeside-eks-test \
	--template-body file://./main.yml \
	--capabilities CAPABILITY_NAMED_IAM
````

#### Delete
```
aws cloudformation delete-stack \
	--stack-name lakeside-eks-test
````
