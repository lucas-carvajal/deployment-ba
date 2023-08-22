# deployments-ba
Deployment Files for Bachelor Thesis

## ECS Stack
cd into the `/ecs` directory, then:
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


## EKS
TODO add deployment steps

