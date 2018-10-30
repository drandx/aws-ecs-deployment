# AWS ECS Deployment Steps
This guide enumerates the steps needed to deploy either long-running tasks (tasks such as micro-services running inside ECS services) or short-running (tasks that don't require a service configuration) tasks to ECS.

### 1. Create the ECR Image
`aws ecr create-repository --repository-name <REPOSITORY_NAME>`
### 2. Create Cluster (One cluster can handle multiple services)
`aws ecs create-cluster --cluster-name <CLUSTER_NAME>`
### 3. Create Security Group  (Create ONLY one for multiple micro-services)
`aws ec2 create-security-group --group-name my-ecs-sg --description <SECURITY_GROUP_NAME>`
### 4. Register Task Definition  (Create one per micro-service and environment: dev, qa, uat, prod).

#### IAM Roles Creation
In order to register a new task the task definition config requires two IAM roles (__Execution__ and __Task__) with the next policies on each one:

* Execution Role Policies:
	
	`AmazonECSTaskExecutionRolePolicy	`

* Task Role Policies: 
	
	`AmazonECSFullAccess`
	
	```
	ECS-CloudWatchLogs
		{
		  "Version": "2012-10-17",
		  "Statement": [
		    {
		      "Effect": "Allow",
		      "Action": [
		        "logs:CreateLogGroup",
		        "logs:CreateLogStream",
		        "logs:PutLogEvents",
		        "logs:DescribeLogStreams"
		      ],
		      "Resource": [
		        "arn:aws:logs:*:*:*"
		      ]
		    }
		  ]
		}
	```
#### Create Log Group Create one per micro-service and environment)
`aws logs create-log-group --log-group-name <LOG_GROUP_NAME>`
#### Register Task
Config file template: [example-task-definition.json](https://github.com/drandx/aws-ecs-deployment/blob/master/fargate/example-task-definition.json)
`aws ecs register-task-definition --cli-input-json file://deploy/example-task-definition.json`

### 5. Create ECS Service (Create one per micro-service environment)
Config file template: [example-ecs-service.json](https://github.com/drandx/aws-ecs-deployment/blob/master/fargate/example-ecs-service.json)

   1. Create Load Balancer (Only for micro-services)(Create ONLY one for multiple micro-services)
        
        aws elbv2 create-load-balancer --name <LOAD_BALANCER_NAME> --subnets <SUBNET_ID_1> <SUBNET_ID_2> <SUBNET_ID_3>
        
   2. Create Target Groups (Only for web micro-service)(Create one per micro-service and environment)
        
        aws elbv2 create-target-group --name <TARGET_GROUP_NAME> --protocol HTTP --port 80 --vpc-id <VPC_ID> --target-type ip`
        
   3. Add Target Group As Listener to the Load Balancer (ONLY for microservices)
        
        aws elbv2 create-listener --load-balancer-arn <LOAD_BALANCER_ARN>  --protocol HTTP --port <PROTOCOL> --default-actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN>
        
   4. Register Service
        
        aws ecs create-service --cli-input-json file://deploy/example-ecs-service.json
        
### 6. Push Image to ECR Repository
```
$(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
docker build --tag "${ECR_REPOSITORY_URI}" .
docker push "${ECR_REPOSITORY_URI}"
```