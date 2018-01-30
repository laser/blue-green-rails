## What's This Presentation About?

  1. Creating a Docker image containing your Rails application
  1. Using AWS CloudFormation to create an ECR repository to store your images
  1. Using AWS CloudFormation to provision an ECS cluster to which you'll deploy your app
  1. Tweaking ECS settings to affect zero-downtime deploys to cluster
  1. Executing cluster-safe database migrations

## Dockerizing Your Application

### What is Docker?

> Docker is a software technology providing operating-system-level virtualization also known as containers, promoted by the company Docker, Inc.[6] Docker provides an additional layer of abstraction and automation of operating-system-level virtualization on Windows and Linux.[7] Docker uses the resource isolation features of the Linux kernel such as cgroups and kernel namespaces, and a union-capable file system such as OverlayFS and others[8] to allow independent "containers" to run within a single Linux instance, avoiding the overhead of starting and maintaining virtual machines (VMs).

### Why use Docker?

### Generate a Rails App

```
rails new websvc -d postgresql --skip-puma --skip-spring \
  && cd websvc \
  && bundle \
  && rails generate scaffold Post title:string content:text
```
      
### Create a Dockerfile

```
cat <<EOF > ./Dockerfile

FROM ruby:2.5.0-alpine

RUN apk add --update postgresql-dev alpine-sdk nodejs tzdata

COPY Gemfile* /opt/bundle/
WORKDIR /opt/bundle

RUN bundle update && bundle install

COPY . /opt/app

WORKDIR /opt/app
ENTRYPOINT ["/bin/ash", "-c"]

EOF
```

### Build the Docker Image

An image is an ordered collection of root filesystem changes and the corresponding execution parameters for use within a container runtime. An image typically contains a union of layered filesystems stacked on top of each other. An image does not have state and it never changes.

```
docker build . -t demo
```

### Run Tests

```
docker run --rm demo "rails test"
```

### Replace Database Configuration YAML

```
cat <<EOF > ./config/database.yml

default: &default
  adapter: postgresql
  encoding: unicode

development:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>

test:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>_test

production:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>

EOF
```

### Add a Database (Docker Compose)

```
cat <<EOF > docker-compose.yml

version: "2"
services:
  websvc:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3333:3000"
    environment:
      - DATABASE_URL=postgresql://postgres@websvcdb:5432/postgres
      - PORT=3000
    depends_on:
      - dockerize
    command:
        - "rails db:migrate && rails server -b 0.0.0.0"
    volumes:
      - .:/opt/app
  websvcdb:
    image: postgres:9.6.5-alpine
  dockerize:
    image: jwilder/dockerize
    command: ["dockerize", "-wait", "tcp://websvcdb:5432", "-timeout", "50s"]
    depends_on:
      - websvcdb
      
EOF
```

### Service Dependencies

> It is common when using tools like Docker Compose to depend on services in other linked containers, however oftentimes relying on links is not enough - whilst the container itself may have started, the service(s) within it may not yet be ready - resulting in shell script hacks to work around race conditions.

> Dockerize gives you the ability to wait for services on a specified protocol (file, tcp, tcp4, tcp6, http, https and unix) before starting your application:

```
dockerize -wait tcp://databasehost:5432 echo "database ready"
```

### Run the Tests (again)

First, generate `schema.rb`:

```
docker-compose run --rm websvc "rails db:migrate"
```

Then, run tests:

```
docker-compose run --rm websvc "rails db:test:prepare && rails test"
```

### Launch the App (locally)

```
docker-compose up
```

```
open 'http://localhost:3333/posts'
```

## Pushing Docker Image to ECR

### Create ECR (via UI)

(click through UI to demonstrate how to create an ECR repository)

### Why CloudFormation?

> Why use AWS CloudFormation with Amazon ECS?
> 
> Using CloudFormation to deploy and manage services with ECS has a number of nice benefits over more traditional methods (AWS CLI, scripting, etc.).
> 
> Infrastructure-as-Code
> 
> A template can be used repeatedly to create identical copies of the same stack (or to use as a foundation to start a new stack). Templates are simple YAML- or JSON-formatted text files that can be placed under your normal source control mechanisms, stored in private or public locations such as Amazon S3, and exchanged via email. With CloudFormation, you can see exactly which AWS resources make up a stack. You retain full control and have the ability to modify any of the AWS resources created as part of a stack.
Self-documenting
> 
> Fed up with outdated documentation on your infrastructure or environments? Still keep manual documentation of IP ranges, security group rules, etc.?
> 
> With CloudFormation, your template becomes your documentation. Want to see exactly what you have deployed? Just look at your template. If you keep it in source control, then you can also look back at exactly which changes were made and by whom.
Intelligent updating & rollback
> 
> CloudFormation not only handles the initial deployment of your infrastructure and environments, but it can also manage the whole lifecycle, including future updates. During updates, you have fine-grained control and visibility over how changes are applied, using functionality such as change sets, rolling update policies and stack policies.

### Configure AWS Client

```
aws configure set default.region us-east-1
```

### Create ECR (via CloudFormation)

Using CloudFormation, we'll create an ECR repository to which we'll push our Docker images:

```
cat <<EOF > ./infrastructure/cloudformation/stacks/image-repository.yml

Resources:
  Repository:
    Type: AWS::ECR::Repository
Outputs:
  RepositoryArn:
    Value: !Sub arn:aws:ecr:${AWS::Region}:${AWS::AccountId}:repository/${Repository}
  RepositoryUri:
    Value: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${Repository}
    
EOF
```

```
aws cloudformation deploy \
  --stack-name demorepo \
  --template-file ./infrastructure/cloudformation/stacks/image-repository.yml
```

Once the CloudFormation stack containing our ECR repository has been created, we obtain its URI:

```
export REPOSITORY_URI=$(aws cloudformation describe-stacks --stack-name demorepo | jq -r '(.Stacks[0].Outputs[] | select(.OutputKey == "RepositoryUri")).OutputValue')
```

### Log in to an Amazon ECR Registry

This command retrieves a token that is valid for a specified registry for 12 hours, and then it prints a docker login command with that authorization token. You can execute the printed command to log in to your registry with Docker. After you have logged in to an Amazon ECR registry with this command, you can use the Docker CLI to push and pull images from that registry until the token expires.

```
eval $(aws ecr get-login --no-include-email)
```

### Push to ECR

Tag the previously-built image:

```
docker tag demo:latest ${REPOSITORY_URI}:latest
```

Push the image to the ECR:

```
docker push ${REPOSITORY_URI}:latest
```

## Provisioning an ECS Cluster (default VPC)

Create a database (15m15.288s):

```
cat <<EOF > ./infrastructure/cloudformation/stacks/database.yml

Description: >
  This stack provisions the RDS instance which will be used by our app.
Resources:
  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      AllocatedStorage: 5
      AutoMinorVersionUpgrade: true
      BackupRetentionPeriod: 0
      DBInstanceClass: db.t2.micro
      DBName: !Ref AWS::StackName
      Engine: postgres
      EngineVersion: 9.6.5
      MasterUsername: rinkydink
      MasterUserPassword: smurfmagnet
      MultiAZ: true
      PubliclyAccessible: true
Outputs:
  DatabaseUrl:
    Description: A database connection string
    Value:
      Fn::Join:
      - ""
      - - "postgresql://rinkydink:smurfmagnet"
        - "@"
        - !GetAtt Database.Endpoint.Address
        - ":"
        - !GetAtt Database.Endpoint.Port
        - "/"
        - !Ref AWS::StackName

EOF
```

```
aws cloudformation deploy \
  --stack-name demodb \
  --template-file ./infrastructure/cloudformation/stacks/database.yml
```

After that's created, get the database URL from the stack output:

```
DATABASE_URL=$(aws cloudformation describe-stacks \
  --stack-name demodb 
  | jq -r '(.Stacks[0].Outputs[] | select(.OutputKey == "DatabaseUrl")).OutputValue')
```

Get the default VPC id:

```
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" | jq -r '.Vpcs[].VpcId')
```

Get the subnets for this VPC:

```
export PUBLIC_SUBNETS=$(aws ec2 describe-subnets   --filters Name=vpc-id,Values=${VPC_ID}   | jq -r '[([.Subnets[].SubnetId])[0, 2]] | join(",")')
```

Create the cluster (5m26.842s):

```
cat <<EOF > ./infrastructure/cloudformation/stacks/cluster.yml

Description: >
  This template deploys a network load balancer that exposes our ECS service to
  the internet
Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: Choose which VPC the Application Load Balancer should be deployed to
  PublicSubnets:
    Description: Choose which subnets the Application Load Balancer should be deployed to
    Type: List<AWS::EC2::Subnet::Id>
  DatabaseUrl:
    Type: String
  DockerImage:
    Type: String
Resources:
  WideOpenSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VpcId
      GroupDescription: Accepting traffic from any place
      SecurityGroupIngress:
      - CidrIp: 0.0.0.0/0
        IpProtocol: -1
      Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName}-wide-open-sg
  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Type: network
      Scheme: internet-facing
      Name: !Sub ${AWS::StackName}-nlb
      Subnets: !Ref PublicSubnets
      Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName}-nlb
  LoadBalancerListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref LoadBalancer
      Port: 80
      Protocol: TCP
      DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref TargetGroup
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub ${AWS::StackName}-target-group
      VpcId: !Ref VpcId
      Port: 3000
      Protocol: TCP
      TargetType: ip
      TargetGroupAttributes:
      - Key: deregistration_delay.timeout_seconds
        Value: '10'
  Cluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub ${AWS::StackName}-cluster
  Service:
    Type: AWS::ECS::Service
    DependsOn: LoadBalancerListener
    Properties:
      Cluster: !Ref Cluster
      LaunchType: FARGATE
      DesiredCount: 2
      DeploymentConfiguration:
        MaximumPercent: 150
        MinimumHealthyPercent: 100
      TaskDefinition: !Ref TaskDefinition
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets: !Ref PublicSubnets
          AssignPublicIp: ENABLED
          SecurityGroups:
          - !Ref WideOpenSecurityGroup
      LoadBalancers:
      - ContainerName: !Sub ${AWS::StackName}-container
        ContainerPort: 3000
        TargetGroupArn: !Ref TargetGroup
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      TaskRoleArn: !GetAtt TaskExecutionRole.Arn
      ExecutionRoleArn: !GetAtt TaskExecutionRole.Arn
      Cpu: 256
      Memory: 512
      Family: !Sub ${AWS::StackName}-task-family
      NetworkMode: awsvpc
      RequiresCompatibilities:
      - FARGATE
      ContainerDefinitions:
      - Name: !Sub ${AWS::StackName}-container
        Command:
        - !Sub "rails db:migrate && rails assets:precompile && rails server -b 0.0.0.0"
        Environment:
        - Name: RAILS_SERVE_STATIC_FILES
          Value: true
        - Name: RAILS_LOG_TO_STDOUT
          Value: true
        - Name: RAILS_ENV
          Value: production
        - Name: PORT
          Value: 3000
        - Name: DATABASE_URL
          Value: !Ref DatabaseUrl
        - Name: SECRET_KEY_BASE
          Value: 7db791ebc31366f3fccd67467ed6128f20175ff947c1fe262beb8f28d9466700fda9dbd278cfe7a63d6ace11d282d125a7bfd2d5b813f19d6f23fd46a8d7cdae
        Essential: true
        Image: !Ref DockerImage
        PortMappings:
        - ContainerPort: 3000
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group: !Sub ${AWS::StackName}
            awslogs-region: !Ref AWS::Region
            awslogs-stream-prefix: ecs
  LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub ${AWS::StackName}
      RetentionInDays: 7
  TaskExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - ecs-tasks.amazonaws.com
          Action:
          - sts:AssumeRole
      ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
Outputs:
  LoadBalancerUrl:
    Description: The URL of the NLB
    Value: !GetAtt LoadBalancer.DNSName
    
EOF
```

```
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --stack-name democluster \
  --template-file ./infrastructure/cloudformation/stacks/cluster.yml \
  --parameter-overrides \
    DatabaseUrl=${DATABASE_URL} \
    VpcId=${VPC_ID} \
    PublicSubnets=${PUBLIC_SUBNETS} \
    DockerImage=${REPOSITORY_URI}:latest
```

Use stack outputs to get URL of the posts index:

```
NLB_DNS_NAME=$($(aws cloudformation describe-stacks \
  --stack-name wittysubfreshman \
  | jq -r '(.Stacks[0].Outputs[] | select(.OutputKey == "LoadBalancerUrl")).OutputValue'))

POSTS_URL=http://${NLB_DNS_NAME}/posts
```

Then, open the app and create a post:

```
open ${POSTS_URL}
```

## Zero-downtime Deploys

### Visualizing a 

### Fiddling with ECS Settings

Our deployment strategy is determined from `DesiredCount`, `MaximumPercent`, and `MinimumHealthyPercent`:

```
Service:
  Type: AWS::ECS::Service
  Properties:
    DesiredCount: 2
    DeploymentConfiguration:
      MaximumPercent: 200
      MinimumHealthyPercent: 100
```

During a deploy, we will burst up to a maximum number of running tasks `N`, where:

```
N = DesiredCount + FLOOR (MaximumPercent / 100 * DesiredCount)
```

As long as `MinimumHealthyPercent * DesiredCount` is greater than 1, we'll do a zero-downtime deploy.

### A Zero-downtime Deploy In Action

Make requests to the Rails application in a loop:

```
while true; do echo $(date)-$(curl -s ${POSTS_URL} | grep h1); sleep 1; done
```

Change the title of the posts index, stage, and commit it:

```
sed -i -e "s;<h1>.*<\/h1>;<h1>Posts $(word)<\/h1>;g" websvc/app/views/posts/index.html.erb
```

Build a new Docker image, tag it, and push it (0m37.228s):

```
docker build websvc/ -t demo:latest \
  && export REPOSITORY_URI=$(aws cloudformation describe-stacks --stack-name demorepo | jq -r '(.Stacks[0].Outputs[] | select(.OutputKey == "RepositoryUri")).OutputValue') \
  && docker tag demo:latest ${REPOSITORY_URI}:latest \
  && eval $(aws ecr get-login --no-include-email) \
  && docker push ${REPOSITORY_URI}:latest
```

Then, update the service (~3 minutes for new task to run, 15 for full update):

```
aws ecs update-service \
  --service democluster-service \
  --cluster democluster-cluster \
  --force-new-deployment
```

## Database Migrations

### Problem

We need to ensure that migrations are run only once per deploy.

### Approach 1: Slow Rollout

```
Service:
  Type: AWS::ECS::Service
  Properties:
    DesiredCount: 2
    DeploymentConfiguration:
      MaximumPercent: 150
      MinimumHealthyPercent: 100
```

### Approach 2: ECS RunTask API

```
export MIGRATION_TASK_ARN=$(aws ecs run-task \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNETS}],assignPublicIp=ENABLED}" \
  --task-definition democluster-task-family \
  --cluster democluster-cluster \
  --count 1 \
  --overrides '{"containerOverrides": [{"name": "democluster-container", "command": ["rails db:migrate"]}]}' \
  --started-by deploy \
  | jq -r '.["tasks"][0]["taskArn"]')

aws ecs wait tasks-stopped \
    --cluster democluster-cluster \
    --tasks ${MIGRATION_TASK_ARN}

export MIGRATION_EXIT_CODE=$(aws ecs describe-tasks \
    --cluster democluster-cluster \
    --tasks ${MIGRATION_TASK_ARN} | jq -r '.["tasks"][0]["containers"][0]["exitCode"]')

if [ "${MIGRATION_EXIT_CODE}" != "0" ] ; then
    # fail build
    exit 1
fi
```
