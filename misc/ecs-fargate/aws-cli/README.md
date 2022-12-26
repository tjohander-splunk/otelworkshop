>https://stackoverflow.com/questions/54174290/fargate-error-cannot-pull-container-hosted-in-ecr-from-a-private-subnet?rq=1 ) *DNSSupport*

This will guide you through setting up an instrumented application, sending its telemetry to a sidecar instance of the Splunk OpenTelemetry Collector, using the traditional AWS CLI toolset.

Instructions for this guide are borrowed heavily from [this](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_AWSCLI_Fargate.html) AWS tutorial.

## Create the Task Execution Role
This role will be required in order to execute the ECS Task. This role will need IAM permissions for the `logs:CreateLogGroup` action.  These two steps provision the role, assume role capabilities and an IAM Managed Policy for Full Logs Access.  For the purposes of this tutorial, these permissions are sufficient.  Your specific scenario may requite tighter controls or the ability to use an existing IAM role and policy with the required permissions.  

Two steps are captured here to create the Role and attach the managed policy.  Note the ARN of the Task Execution Role. It's needed in the next step.
> Region is optional, the default is whatever is set in your AWS CLI profile
```bash
aws iam create-role \
--role-name ecsTaskExecutionRole \
--assume-role-policy-document file://task-execution-assume-role.json
```
```bash
aws iam attach-role-policy \
--role-name ecsTaskExecutionRole \
--policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

## Prepare `tracegen-java-otel-fargate-otelcol.json` File
There are user-defined values that need to be replaced with values for your environment.  Follow this table:

| Value             | Definition                                                                                         | Notes                                                                                             |
|-------------------|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| `YOURREGIONHERE`  | The AWS Region in which this sample will be deployed                                               | `us-west-1`, etc...                                                                               |
| `YOURTOKENHERE`   | Splunk Observability Cloud Suite Access (Ingest) Token                                             | [See Docs](https://docs.splunk.com/Observability/admin/authentication-tokens/org-tokens.html)     |
| `YOURINITIALHERE` | Arbitrary Value to Differentiate your environment from others in a shared workshop (If applicable) | Disregard if not running this example in a workshop                                               |
| `YOURREALMHERE`   | Splunk Observability Cloud Suite Region                                                            | [See Docs](https://docs.splunk.com/Observability/references/organizations.html#nav-Organizations) |
| `YOURROLEHERE`    | The Task Execution Role ARN that will execute the ECS Task                                         | Acquired in the previous step                                                                     |

## Create a Cluster: 
```bash
aws ecs create-cluster --cluster-name fargate-cluster

cluster:
  activeServicesCount: 0
  capacityProviders: []
  clusterArn: arn:aws:ecs:us-east-1:455790677231:cluster/fargate-cluster
  clusterName: fargate-cluster
  defaultCapacityProviderStrategy: []
  pendingTasksCount: 0
  registeredContainerInstancesCount: 0
  runningTasksCount: 0
  settings:
  - name: containerInsights
    value: disabled
  statistics: []
  status: ACTIVE
  tags: []
```

## Register the Task Definition:
  ```bash
  aws ecs register-task-definition --cli-input-json file://tracegen-java-otel-fargate-otelcol.json
  
  taskDefinition:
  compatibilities:
  - EC2
  - FARGATE
  containerDefinitions:
  - cpu: 0
    dockerLabels:
      app: tracegen-java-otel-fargate
    environment:
    - name: OTEL_SERVICE_NAME
      value: tracegen-java-fargate
    - name: SPLUNK_PROFILER_ENABLED
      value: 'true'
    - name: OTEL_RESOURCE_ATTRIBUTES
      value: deployment.environment=tj-devlab
    - name: SPLUNK_ACCESS_TOKEN
      value: 0UPOH-lTdvFDUP5ksYh80Q
    essential: true
    image: docker.io/stevelsplunk/splk-java-autogen
    logConfiguration:
      logDriver: awslogs
      options:
        awslogs-group: ecs
        awslogs-region: us-east-1
        awslogs-stream-prefix: tracegen-java-fargate
    mountPoints: []
    name: tracegen-java-otel-fargate
    portMappings: []
    volumesFrom: []
  - cpu: 0
    environment:
    - name: SPLUNK_CONFIG
      value: /etc/otel/collector/ecs_ec2_config.yaml
    - name: SPLUNK_REALM
      value: us1
    - name: SPLUNK_ACCESS_TOKEN
      value: 0UPOH-lTdvFDUP5ksYh80Q
    - name: ECS_METADATA_EXCLUDED_IMAGES
      value: '["quay.io/signalfx/splunk-otel-collector"]'
    essential: true
    image: quay.io/signalfx/splunk-otel-collector:latest
    logConfiguration:
      logDriver: awslogs
      options:
        awslogs-group: otelcol
        awslogs-region: us-east-1
        awslogs-stream-prefix: otelcol
    mountPoints: []
    name: splunk-otel-collector
    portMappings: []
    volumesFrom: []
  cpu: '256'
  executionRoleArn: arn:aws:iam::455790677231:role/ecsTaskExecutionRole
  family: tracegen-java-otel-fargate
  memory: '512'
  networkMode: awsvpc
  placementConstraints: []
  registeredAt: '2022-12-24T06:56:04.518000-06:00'
  registeredBy: arn:aws:iam::455790677231:user/thomas
  requiresAttributes:
  - name: com.amazonaws.ecs.capability.logging-driver.awslogs
  - name: ecs.capability.execution-role-awslogs
  - name: com.amazonaws.ecs.capability.docker-remote-api.1.19
  - name: com.amazonaws.ecs.capability.docker-remote-api.1.18
  - name: ecs.capability.task-eni
  requiresCompatibilities:
  - FARGATE
  revision: 1
  status: ACTIVE
  taskDefinitionArn: arn:aws:ecs:us-east-1:455790677231:task-definition/tracegen-java-otel-fargate:1
  volumes: []


  ```

### (OPTIONAL) Confirm Task Definition Registraion:
```bash
aws ecs list-task-definitions

taskDefinitionArns:
- arn:aws:ecs:us-east-1:455790677231:task-definition/tracegen-java-otel-fargate:1
```

## Create a Service In a Pre-Existing VPC
> NOTE: In this step, it's assumed you have access to a VPC with a subnet and security group appropriate for your needs.  Creation of such a resource is out of scope for this tutorial.  If you'd prefer to avoid having to meet this pre-requisite, refer to the ECS-CLI approach.  The ECS-CLI will provision a VPC for you as part of the workflow.
### This will create a service running in a private subnet
> The value for `--task-definition` will come from the output in the step "Register Task Definition".  Refer to the property `taskDefinitionArn`.  In the example above, `taskDefinitionArn: arn:aws:ecs:us-east-1:455790677231:task-definition/tracegen-java-otel-fargate:1`, the task definition value would be `tracegen-java-otel-fargate:1`.
```bash
aws ecs create-service \
--cluster fargate-cluster \
--service-name fargate-service \
--task-definition tracegen-java-otel-fargate:1 \
--desired-count 1 \
--launch-type "FARGATE" \
--network-configuration "awsvpcConfiguration={subnets=[subnet-03595eeba4d607bb0],securityGroups=[sg-034aaf8a4c4380c58]}"

service:
  clusterArn: arn:aws:ecs:us-east-1:455790677231:cluster/fargate-cluster
  createdAt: '2022-12-24T07:12:26.227000-06:00'
  createdBy: arn:aws:iam::455790677231:user/thomas
  deploymentConfiguration:
    deploymentCircuitBreaker:
      enable: false
      rollback: false
    maximumPercent: 200
    minimumHealthyPercent: 100
  deploymentController:
    type: ECS
  deployments:
  - createdAt: '2022-12-24T07:12:26.227000-06:00'
    desiredCount: 1
    failedTasks: 0
    id: ecs-svc/7069295983772853028
    launchType: FARGATE
    networkConfiguration:
      awsvpcConfiguration:
        assignPublicIp: DISABLED
        securityGroups:
        - sg-0476905d446ec5084
        subnets:
        - subnet-08a7f9a848c199dc8
    pendingCount: 0
    platformFamily: Linux
    platformVersion: 1.4.0
    rolloutState: IN_PROGRESS
    rolloutStateReason: ECS deployment ecs-svc/7069295983772853028 in progress.
    runningCount: 0
    status: PRIMARY
    taskDefinition: arn:aws:ecs:us-east-1:455790677231:task-definition/tracegen-java-otel-fargate:1
    updatedAt: '2022-12-24T07:12:26.227000-06:00'
  desiredCount: 1
  enableECSManagedTags: false
  enableExecuteCommand: false
  events: []
  launchType: FARGATE
  loadBalancers: []
  networkConfiguration:
    awsvpcConfiguration:
      assignPublicIp: DISABLED
      securityGroups:
      - sg-0476905d446ec5084
      subnets:
      - subnet-08a7f9a848c199dc8
  pendingCount: 0
  placementConstraints: []
  placementStrategy: []
  platformFamily: Linux
  platformVersion: LATEST
  propagateTags: NONE
  roleArn: arn:aws:iam::455790677231:role/aws-service-role/ecs.amazonaws.com/AWSServiceRoleForECS
  runningCount: 0
  schedulingStrategy: REPLICA
  serviceArn: arn:aws:ecs:us-east-1:455790677231:service/fargate-cluster/fargate-service
  serviceName: fargate-service
  serviceRegistries: []
  status: ACTIVE
  taskDefinition: arn:aws:ecs:us-east-1:455790677231:task-definition/tracegen-java-otel-fargate:1
```
### This will create a service running in a public subnet
```bash
aws ecs create-service --cluster fargate-cluster --service-name fargate-service --task-definition sample-fargate:1 --desired-count 1 --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[subnet-abcd1234],securityGroups=[sg-abcd1234],assignPublicIp=ENABLED}"

<output omitted for brevity>
```
### (OPTIONAL) Confirm Service Was Created:
```bash
aws ecs list-services --cluster fargate-cluster

serviceArns:
- arn:aws:ecs:us-east-1:455790677231:service/fargate-cluster/fargate-service
```
### (OPTIONAL) Describe the Running Service
```bash
aws ecs describe-services --cluster fargate-cluster --services fargate-service
```
### Commands Reference
```bash
aws ecs update-service --service fargate-service --cluster fargate-cluster --network-configuration "awsvpcConfiguration={subnets=[subnet-022a62ebe1b8443b0],securityGroups=[sg-0476905d446ec5084]}"
```
```bash
aws ecs scale-service 
aws ecs delete-service --service fargate-service --cluster fargate-cluster
```
```bash
aws ecs create-service \
--cluster fargate-cluster \
--service-name fargate-service \
--task-definition tracegen-java-otel-fargate:1 \
--desired-count 1 \
--launch-type "FARGATE" \
--network-configuration "awsvpcConfiguration={subnets=[subnet-08736e98e80f72d59],assignPublicIp=ENABLED}" \
--region us-east-1

aws ecs delete-service \
--cluster fargate-cluster \
--service fargate-service \
--region us-east-1

aws ecs register-task-definition \
--cli-input-json file://tracegen-java-otel-fargate-otelcol.json

aws ecs update-service \
--cluster fargate-cluster \
--service fargate-service \
--task-definition tracegen-java-otel-fargate:6 \
--network-configuration "awsvpcConfiguration={subnets=[subnet-03595eeba4d607bb0],securityGroups=[sg-034aaf8a4c4380c58]}"
--region us-east-1
```