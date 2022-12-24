# How to create an instrumented Fargate-based ECS Task
There are several advantages to using the ECS CLI to manage an ECS container environment.The ECS CLI automatically provisions a VPC, you can use the familiar Docker Compose YAML file format to define your Task and many other ergonomic aids.  For more information, refer to the ECS CLI [documentation.](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI.html)

The steps in this section borrow heavily from the ECS [tutorial](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-cli-tutorial-fargate.html).  Modifications have been made to run an instrumented Python application image with a sidecar container running the Splunk Distribution of the OpenTelemetry Collector.

# Pre Requisites
Make sure you have access to the following:
* An AWS account with Administrator permissions
* Basic CLI knowledge on the platform of your choice. Bash examples are provided here, PowerShell or other terminal frameworks are out of scope for this tutorial.

## Create the Task Execution Role:
> Region is optional, the default is whatever is set in your AWS CLI profile
```bash
aws iam create-role \
--role-name ecsTaskExecutionRole \
--assume-role-policy-document file://task-execution-assume-role.json
```
## Attach the task execution role policy:
```bash
aws iam attach-role-policy \
--role-name ecsTaskExecutionRole \
--policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```
# Configure ECS CLI
In this section we will setup an ECS CLI Profile to allow your instance of the ECS CLI to manage multiple clusters simultaneously.

> **Note:** The `ECS CLI` is a different application from the `AWS CLI` used in the previous steps.  The ECS CLI offers a streamlined means to interact with AWS ECS Resources.  

## Create an ECS Cluster Configuration Object
Create a Fargate-based ECS Cluster
> **Note:** The `cluster` name and `config-name` values are arbitrary. Feel free to supply your own values.
```bash
ecs-cli configure --cluster tutorial --default-launch-type FARGATE --config-name tutorial --region us-east-1
```

## Create an ECS-CLI Profile
Use the values appropriate to your account for the Access Key and Secret Key.  If you're using a Splunk-backed AWS Account, you will also need to add in your short-lived Session Token.
```bash
ecs-cli configure profile \
--access-key <your-aws-access-key> \
--secret-key <your-aws-secret-key> \
--session-token <your-aws-session-token-if-required> \
--profile-name tutorial-profile
```

# Create a Cluster and Configure Security Group
## Create a Cluster
This command will be called using the profile name created in the previous step.  This command may take a few minutes to complete as your resources are created. The output of this command contains the VPC and subnet IDs that are created. **Make a note of these IDs as you will need them in the following steps.**
```bash
ecs-cli up --cluster-config tutorial --ecs-profile tutorial-profile
```
The output result should look similar to this:
```
INFO[0000] Created cluster                               cluster=tutorial region=us-east-1
INFO[0001] Waiting for your cluster resources to be created...
INFO[0001] Cloudformation stack status                   stackStatus=CREATE_IN_PROGRESS
INFO[0062] Cloudformation stack status                   stackStatus=CREATE_IN_PROGRESS
VPC created: vpc-0c6ce5d26b0ad38c8 <--------- Note
Subnet created: subnet-0187e10419b998b46 <--------- Note
Subnet created: subnet-0f5aa032566cc195b <--------- Note
Cluster creation succeeded.
```

## Fetch Security Group ID from the new VPC
The previous step created a minimally configured VPC for you.  You will need the security group ID of this newly-minted VPC.  Execute this step to fetch that ID.  
> Note: This step uses the `aws-cli` not the `ecs-cli`
```bash
aws ec2 describe-security-groups --filters Name=vpc-id,Values=VPC_ID --region us-east-1
```
Result:
```json
{
    "SecurityGroups": [
        {
            "Description": "default VPC security group",
            "GroupName": "default",
            "IpPermissions": [
                {
                    "IpProtocol": "-1",
                    "IpRanges": [],
                    "Ipv6Ranges": [],
                    "PrefixListIds": [],
                    "UserIdGroupPairs": [
                        {
                            "GroupId": "sg-045e111e0b3b8b2b0", <------ Note
                            "UserId": "123456789"
                        }
                    ]
                }
            ],
            "OwnerId": "123456789",
            "GroupId": "sg-045e111e0b3b8b2b0",
            "IpPermissionsEgress": [
                {
                    "IpProtocol": "-1",
                    "IpRanges": [
                        {
                            "CidrIp": "0.0.0.0/0"
                        }
                    ],
                    "Ipv6Ranges": [],
                    "PrefixListIds": [],
                    "UserIdGroupPairs": []
                }
            ],
            "VpcId": "vpc-0c6ce5d26b0ad38c8"
        }
    ]
}
```

## Add a Security Group Rule to Allow HTTP Traffic (Optional)
This step is only needed if you need to allow inbound HTTP traffic (or any other type) to your ECS-based Task. The command below is a **sample** that would allow inbound traffic to TCP port 80 from any source IP address.  Your needs may vary, amend the command to suit your needs.

```bash
aws ec2 authorize-security-group-ingress --group-id <security-group-id-from-previous-step> --protocol tcp --port 80 --cidr 0.0.0.0/0 --region us-east-1
```

Result:
```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0ac287a1c7fa4e7f9",
            "GroupId": "sg-045e111e0b3b8b2b0",
            "GroupOwnerId": "104053265317",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```

# Create/Modify a Compose File and ECS Params File
## Compose File 
A sample application is defined in `docker-compose.yaml` file.  This is a basic, instrumented Java application. As an additional workload to the ECS task definition, an instance of the Splunk Otel Collector is running alongside the application container.    

## Params File
In addition to the Docker compose information, there are some parameters specific to Amazon ECS that you must specify for the service. Using the VPC, subnet, and security group IDs from the previous step, fill in the values in the `ecs-params.yaml` file

# Deploy Compose File to a Cluster
```bash
ecs-cli compose --project-name tutorial service up --create-log-groups --cluster-config tutorial --ecs-profile tutorial-profile
```
Result:
```
INFO[0000] Using ECS task definition                     TaskDefinition="tutorial:1"
INFO[0000] Created Log Group tutorial in us-east-1
INFO[0001] Auto-enabling ECS Managed Tags
INFO[0011] (service tutorial) has started 1 tasks: (task 304c1af9670347d886f85c1405f0fa70).  timestamp="2022-08-18 20:13:35 +0000 UTC"
INFO[0037] Service status                                desiredCount=1 runningCount=1 serviceName=tutorial
INFO[0037] (service tutorial) has reached a steady state.  timestamp="2022-08-18 20:14:03 +0000 UTC"
INFO[0037] (service tutorial) (deployment ecs-svc/3625060107450266417) deployment completed.  timestamp="2022-08-18 20:14:03 +0000 UTC"
INFO[0037] ECS Service has reached a stable state        desiredCount=1 runningCount=1 serviceName=tutorial
INFO[0037] Created an ECS service                        service=tutorial taskDefinition="tutorial:1"
```

#  View the Running Containers on a Cluster
```bash
ecs-cli compose --project-name tutorial service ps --cluster-config tutorial --ecs-profile tutorial-profile
```
Result:
```
Name                                                                  State    Ports  TaskDefinition  Health
tutorial/23642e710fdf4a84affeca6ea7f411aa/splunk-otel-collector       RUNNING         tutorial:3      UNKNOWN
tutorial/23642e710fdf4a84affeca6ea7f411aa/tracegen-java-otel-fargate  RUNNING         tutorial:3      UNKNOWN
```

# View Container Logs
```bash
ecs-cli logs --task-id 23642e710fdf4a84affeca6ea7f411aa --follow --cluster-config tutorial --ecs-profile tutorial-profile
```

# Scale the Cluster
## Scale
```bash
ecs-cli compose --project-name tutorial service scale 2 --cluster-config tutorial --ecs-profile tutorial-profile
```
## Verify
```bash
ecs-cli compose --project-name tutorial service ps --cluster-config tutorial --ecs-profile tutorial-profile
```

# Cleanup
## Delete the Service
```bash
ecs-cli compose --project-name tutorial service down --cluster-config tutorial --ecs-profile tutorial-profile
```

## Take Down the Cluster
```bash
ecs-cli down --force --cluster-config tutorial --ecs-profile tutorial-profile
```








