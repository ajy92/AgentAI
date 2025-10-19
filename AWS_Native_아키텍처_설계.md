# LGM Project - AWS Native 아키텍처 설계

## 1. 전체 아키텍처 개요

AWS Native 서비스들을 활용하여 LGM Project를 완전히 클라우드 기반으로 구현하는 아키텍처입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Cloud                                │
├─────────────────────────────────────────────────────────────────┤
│  Frontend Layer                                                 │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │   CloudFront    │    │   S3 Static     │                    │
│  │   (CDN)         │◄──►│   Website       │                    │
│  └─────────────────┘    └─────────────────┘                    │
├─────────────────────────────────────────────────────────────────┤
│  API Gateway Layer                                              │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │   API Gateway   │    │   WebSocket     │                    │
│  │   (REST API)    │    │   API Gateway   │                    │
│  └─────────────────┘    └─────────────────┘                    │
├─────────────────────────────────────────────────────────────────┤
│  Compute Layer                                                  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐ │
│  │   Lambda        │    │   ECS Fargate   │    │   SageMaker  │ │
│  │   (API Handler) │    │   (MCP Servers) │    │   (ML Jobs)  │ │
│  └─────────────────┘    └─────────────────┘    └──────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  AI/ML Services                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐ │
│  │   Bedrock       │    │   Knowledge     │    │   OpenSearch │ │
│  │   (LLM)         │    │   Base          │    │   (Vector)   │ │
│  └─────────────────┘    └─────────────────┘    └──────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  Data Layer                                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐ │
│  │   S3            │    │   DynamoDB      │    │   RDS        │ │
│  │   (Documents)   │    │   (Sessions)    │    │   (Metadata) │ │
│  └─────────────────┘    └─────────────────┘    └──────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## 2. 핵심 AWS 서비스 구성

### 2.1 Frontend Layer

#### 2.1.1 CloudFront + S3 Static Website
```yaml
# CloudFront Distribution
CloudFront:
  Type: AWS::CloudFront::Distribution
  Properties:
    DistributionConfig:
      Origins:
        - DomainName: !GetAtt S3Bucket.DomainName
          Id: S3Origin
          S3OriginConfig:
            OriginAccessIdentity: !Ref OriginAccessIdentity
      DefaultCacheBehavior:
        TargetOriginId: S3Origin
        ViewerProtocolPolicy: redirect-to-https
        CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
      Enabled: true
      DefaultRootObject: index.html
```

#### 2.1.2 React/Vue.js 기반 SPA
- **S3 Static Website Hosting**: 정적 웹사이트 호스팅
- **CloudFront**: 글로벌 CDN 및 캐싱
- **Route 53**: 도메인 관리 및 라우팅

### 2.2 API Layer

#### 2.2.1 API Gateway
```yaml
# REST API Gateway
RestApi:
  Type: AWS::ApiGateway::RestApi
  Properties:
    Name: lgm-project-api
    Description: LGM Project REST API
    EndpointConfiguration:
      Types:
        - REGIONAL

# WebSocket API Gateway
WebSocketApi:
  Type: AWS::ApiGatewayV2::Api
  Properties:
    Name: lgm-project-websocket
    ProtocolType: WEBSOCKET
    RouteSelectionExpression: $request.body.action
```

#### 2.2.2 Lambda Functions
```yaml
# Main API Handler
ApiHandler:
  Type: AWS::Serverless::Function
  Properties:
    CodeUri: src/api/
    Handler: app.handler
    Runtime: python3.11
    Timeout: 30
    MemorySize: 1024
    Environment:
      Variables:
        KNOWLEDGE_BASE_ID: !Ref KnowledgeBase
        BEDROCK_REGION: !Ref AWS::Region
    Policies:
      - BedrockInvokeModelPolicy: {}
      - S3ReadPolicy:
          BucketName: !Ref DocumentsBucket
      - DynamoDBCrudPolicy:
          TableName: !Ref SessionsTable
```

### 2.3 Compute Layer

#### 2.3.1 ECS Fargate (MCP Servers)
```yaml
# ECS Cluster
EcsCluster:
  Type: AWS::ECS::Cluster
  Properties:
    ClusterName: lgm-mcp-cluster
    CapacityProviders:
      - FARGATE
      - FARGATE_SPOT
    DefaultCapacityProviderStrategy:
      - CapacityProvider: FARGATE
        Weight: 1

# MCP Server Task Definition
McpServerTask:
  Type: AWS::ECS::TaskDefinition
  Properties:
    Family: lgm-mcp-server
    NetworkMode: awsvpc
    RequiresCompatibilities:
      - FARGATE
    Cpu: 512
    Memory: 1024
    ContainerDefinitions:
      - Name: mcp-server
        Image: !Sub "${ECRRepository}:latest"
        PortMappings:
          - ContainerPort: 8000
        Environment:
          - Name: AWS_REGION
            Value: !Ref AWS::Region
          - Name: KNOWLEDGE_BASE_ID
            Value: !Ref KnowledgeBase
```

#### 2.3.2 AWS Batch (Code Execution)
```yaml
# Batch Compute Environment
BatchComputeEnvironment:
  Type: AWS::Batch::ComputeEnvironment
  Properties:
    Type: MANAGED
    ComputeResources:
      Type: FARGATE
      MaxvCpus: 100
      Subnets:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      SecurityGroupIds:
        - !Ref BatchSecurityGroup
    ServiceRole: !GetAtt BatchServiceRole.Arn

# Job Queue
BatchJobQueue:
  Type: AWS::Batch::JobQueue
  Properties:
    JobQueueName: lgm-code-execution-queue
    Priority: 1
    ComputeEnvironmentOrder:
      - Order: 1
        ComputeEnvironment: !Ref BatchComputeEnvironment
```

### 2.4 AI/ML Services

#### 2.4.1 Amazon Bedrock
```yaml
# Bedrock Foundation Model Access
BedrockAccess:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: bedrock.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: BedrockInvokeModel
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - bedrock:InvokeModel
                - bedrock:InvokeModelWithResponseStream
              Resource: "*"
```

#### 2.4.2 Knowledge Base
```yaml
# Knowledge Base
KnowledgeBase:
  Type: AWS::Bedrock::KnowledgeBase
  Properties:
    Name: lgm-knowledge-base
    Description: LGM Project Knowledge Base
    RoleArn: !GetAtt KnowledgeBaseRole.Arn
    KnowledgeBaseConfiguration:
      Type: VECTOR
      VectorKnowledgeBaseConfiguration:
        EmbeddingModelArn: !Sub "arn:aws:bedrock:${AWS::Region}::foundation-model/amazon.titan-embed-text-v1"
    StorageConfiguration:
      Type: OPENSEARCH_SERVERLESS
      OpensearchServerlessConfiguration:
        CollectionArn: !GetAtt OpenSearchCollection.Arn
        VectorIndexName: lgm-vector-index
        FieldMapping:
          VectorField: "bedrock-knowledge-default-vector"
          TextField: "AMAZON_BEDROCK_TEXT_CHUNK"
          MetadataField: "AMAZON_BEDROCK_METADATA"
```

#### 2.4.3 OpenSearch Serverless
```yaml
# OpenSearch Serverless Collection
OpenSearchCollection:
  Type: AWS::OpenSearchServerless::Collection
  Properties:
    Name: lgm-vector-collection
    Type: VECTORSEARCH
    VectorSearchConfiguration:
      VectorSearchConfiguration:
        - Name: lgm-vector-index
          Dimensions: 1536
          VectorSearchConfiguration:
            Algorithm: HNSW
```

### 2.5 Data Layer

#### 2.5.1 S3 Buckets
```yaml
# Documents Bucket
DocumentsBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: !Sub "lgm-documents-${AWS::AccountId}-${AWS::Region}"
    VersioningConfiguration:
      Status: Enabled
    LifecycleConfiguration:
      Rules:
        - Id: DeleteOldVersions
          Status: Enabled
          NoncurrentVersionExpirationInDays: 30
    NotificationConfiguration:
      LambdaConfigurations:
        - Event: s3:ObjectCreated:*
          Function: !GetAtt DocumentProcessor.Arn
          Filter:
            S3Key:
              Rules:
                - Name: prefix
                  Value: "documents/"

# Artifacts Bucket (Generated files)
ArtifactsBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: !Sub "lgm-artifacts-${AWS::AccountId}-${AWS::Region}"
    CorsConfiguration:
      CorsRules:
        - AllowedHeaders: ["*"]
          AllowedMethods: [GET, PUT, POST, DELETE]
          AllowedOrigins: ["*"]
          MaxAge: 3600
```

#### 2.5.2 DynamoDB Tables
```yaml
# Sessions Table
SessionsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: lgm-sessions
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: sessionId
        AttributeType: S
      - AttributeName: timestamp
        AttributeType: S
    KeySchema:
      - AttributeName: sessionId
        KeyType: HASH
    GlobalSecondaryIndexes:
      - IndexName: timestamp-index
        KeySchema:
          - AttributeName: timestamp
            KeyType: HASH
        Projection:
          ProjectionType: ALL
    TimeToLiveSpecification:
      AttributeName: ttl
      Enabled: true

# MCP Tools Registry
McpToolsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: lgm-mcp-tools
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: toolId
        AttributeType: S
    KeySchema:
      - AttributeName: toolId
        KeyType: HASH
```

#### 2.5.3 RDS Aurora Serverless
```yaml
# Aurora Serverless v2 Cluster
AuroraCluster:
  Type: AWS::RDS::DBCluster
  Properties:
    DBClusterIdentifier: lgm-aurora-cluster
    Engine: aurora-postgresql
    EngineMode: provisioned
    EngineVersion: "15.4"
    DatabaseName: lgm
    MasterUsername: !Ref DBUsername
    MasterUserPassword: !Ref DBPassword
    ServerlessV2ScalingConfiguration:
      MinCapacity: 0.5
      MaxCapacity: 16
    VpcSecurityGroupIds:
      - !Ref DatabaseSecurityGroup
    DBSubnetGroupName: !Ref DBSubnetGroup
    EnableCloudwatchLogsExports:
      - postgresql
```

## 3. 네트워크 아키텍처

### 3.1 VPC 구성
```yaml
# VPC
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: 10.0.0.0/16
    EnableDnsHostnames: true
    EnableDnsSupport: true
    Tags:
      - Key: Name
        Value: lgm-vpc

# Public Subnets
PublicSubnet1:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.1.0/24
    AvailabilityZone: !Select [0, !GetAZs '']
    MapPublicIpOnLaunch: true

PublicSubnet2:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.2.0/24
    AvailabilityZone: !Select [1, !GetAZs '']
    MapPublicIpOnLaunch: true

# Private Subnets
PrivateSubnet1:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.10.0/24
    AvailabilityZone: !Select [0, !GetAZs '']

PrivateSubnet2:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.20.0/24
    AvailabilityZone: !Select [1, !GetAZs '']
```

### 3.2 NAT Gateway 및 Internet Gateway
```yaml
# Internet Gateway
InternetGateway:
  Type: AWS::EC2::InternetGateway

# NAT Gateway
NATGateway:
  Type: AWS::EC2::NatGateway
  Properties:
    AllocationId: !GetAtt NatGatewayEIP.AllocationId
    SubnetId: !Ref PublicSubnet1

# Elastic IP for NAT Gateway
NatGatewayEIP:
  Type: AWS::EC2::EIP
  Properties:
    Domain: vpc
```

## 4. 보안 구성

### 4.1 IAM 역할 및 정책
```yaml
# Lambda Execution Role
LambdaExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
    Policies:
      - PolicyName: BedrockAccess
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - bedrock:InvokeModel
                - bedrock:InvokeModelWithResponseStream
              Resource: "*"
      - PolicyName: S3Access
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - s3:GetObject
                - s3:PutObject
                - s3:DeleteObject
              Resource:
                - !Sub "${DocumentsBucket}/*"
                - !Sub "${ArtifactsBucket}/*"
```

### 4.2 보안 그룹
```yaml
# API Security Group
ApiSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for API Gateway
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
    SecurityGroupEgress:
      - IpProtocol: -1
        CidrIp: 0.0.0.0/0

# Lambda Security Group
LambdaSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for Lambda functions
    VpcId: !Ref VPC
    SecurityGroupEgress:
      - IpProtocol: -1
        CidrIp: 0.0.0.0/0
```

## 5. 모니터링 및 로깅

### 5.1 CloudWatch 구성
```yaml
# CloudWatch Log Groups
ApiLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: /aws/apigateway/lgm-api
    RetentionInDays: 30

LambdaLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: /aws/lambda/lgm-api-handler
    RetentionInDays: 30

# CloudWatch Alarms
HighErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: lgm-high-error-rate
    AlarmDescription: High error rate in API Gateway
    MetricName: 4XXError
    Namespace: AWS/ApiGateway
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 10
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: ApiName
        Value: !Ref RestApi
```

### 5.2 X-Ray 추적
```yaml
# X-Ray Sampling Rule
XRaySamplingRule:
  Type: AWS::XRay::SamplingRule
  Properties:
    RuleName: lgm-sampling-rule
    Priority: 1
    Version: 1
    FixedRate: 0.1
    ReservoirSize: 1
    ServiceName: lgm-api
    ServiceType: AWS::ApiGateway
    Host: "*"
    HTTPMethod: "*"
    URLPath: "*"
```

## 6. 배포 및 CI/CD

### 6.1 CodePipeline 구성
```yaml
# CodePipeline
DeploymentPipeline:
  Type: AWS::CodePipeline::Pipeline
  Properties:
    Name: lgm-deployment-pipeline
    RoleArn: !GetAtt CodePipelineRole.Arn
    Stages:
      - Name: Source
        Actions:
          - Name: SourceAction
            ActionTypeId:
              Category: Source
              Owner: AWS
              Provider: S3
              Version: 1
            Configuration:
              S3Bucket: !Ref SourceBucket
              S3ObjectKey: source.zip
      - Name: Build
        Actions:
          - Name: BuildAction
            ActionTypeId:
              Category: Build
              Owner: AWS
              Provider: CodeBuild
              Version: 1
            Configuration:
              ProjectName: !Ref CodeBuildProject
      - Name: Deploy
        Actions:
          - Name: DeployAction
            ActionTypeId:
              Category: Deploy
              Owner: AWS
              Provider: CloudFormation
              Version: 1
            Configuration:
              StackName: lgm-stack
              TemplatePath: BuildArtifact::template.yaml
```

### 6.2 ECR 리포지토리
```yaml
# ECR Repository
ECRRepository:
  Type: AWS::ECR::Repository
  Properties:
    RepositoryName: lgm-mcp-server
    ImageScanningConfiguration:
      ScanOnPush: true
    LifecyclePolicy:
      LifecyclePolicyText: |
        {
          "rules": [
            {
              "rulePriority": 1,
              "description": "Keep last 10 images",
              "selection": {
                "tagStatus": "any",
                "countType": "imageCountMoreThan",
                "countNumber": 10
              },
              "action": {
                "type": "expire"
              }
            }
          ]
        }
```

## 7. 비용 최적화

### 7.1 Spot Instances 활용
- **ECS Fargate Spot**: MCP 서버용 비용 절약
- **Batch Spot Instances**: 코드 실행용 비용 절약

### 7.2 자동 스케일링
```yaml
# ECS Auto Scaling
McpServiceAutoScaling:
  Type: AWS::ApplicationAutoScaling::ScalableTarget
  Properties:
    MaxCapacity: 10
    MinCapacity: 1
    ResourceId: !Sub "service/${EcsCluster}/${McpService}"
    RoleARN: !GetAtt AutoScalingRole.Arn
    ScalableDimension: ecs:service:DesiredCount
    ServiceNamespace: ecs

# Auto Scaling Policy
McpServiceScalingPolicy:
  Type: AWS::ApplicationAutoScaling::ScalingPolicy
  Properties:
    PolicyName: lgm-mcp-scaling-policy
    PolicyType: TargetTrackingScaling
    ScalingTargetId: !Ref McpServiceAutoScaling
    TargetTrackingScalingPolicyConfiguration:
      TargetValue: 70.0
      PredefinedMetricSpecification:
        PredefinedMetricType: ECSServiceAverageCPUUtilization
```

## 8. 재해 복구

### 8.1 Multi-AZ 구성
- **RDS Aurora**: 자동 Multi-AZ 복제
- **DynamoDB**: Global Tables 활용
- **S3**: Cross-Region Replication

### 8.2 백업 전략
```yaml
# S3 Cross-Region Replication
S3ReplicationRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: s3.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: S3ReplicationPolicy
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - s3:GetObjectVersion
                - s3:GetObjectVersionAcl
              Resource: !Sub "${DocumentsBucket}/*"
            - Effect: Allow
              Action:
                - s3:ReplicateObject
                - s3:ReplicateDelete
              Resource: !Sub "arn:aws:s3:::${BackupBucket}/*"
```

## 9. 구현 단계

### Phase 1: 기본 인프라 (2주)
1. VPC, 서브넷, 보안 그룹 구성
2. RDS Aurora, DynamoDB, S3 버킷 생성
3. 기본 IAM 역할 및 정책 설정

### Phase 2: API 및 Lambda (2주)
1. API Gateway 구성
2. Lambda 함수 개발 및 배포
3. Bedrock 및 Knowledge Base 연동

### Phase 3: MCP 서버 (2주)
1. ECS Fargate 클러스터 구성
2. MCP 서버 컨테이너화 및 배포
3. 서비스 디스커버리 설정

### Phase 4: Frontend (2주)
1. React/Vue.js SPA 개발
2. S3 Static Website 호스팅
3. CloudFront CDN 구성

### Phase 5: 모니터링 및 최적화 (1주)
1. CloudWatch 대시보드 구성
2. X-Ray 추적 설정
3. 성능 최적화 및 비용 분석

## 10. 예상 비용 (월간)

### 10.1 기본 비용
- **API Gateway**: $3.50 per million requests
- **Lambda**: $0.20 per 1M requests + $0.0000166667 per GB-second
- **ECS Fargate**: $0.04048 per vCPU-hour + $0.004445 per GB-hour
- **RDS Aurora Serverless v2**: $0.12 per ACU-hour
- **DynamoDB**: $1.25 per million read requests + $6.25 per million write requests
- **S3**: $0.023 per GB storage + $0.0004 per 1,000 requests

### 10.2 예상 월간 비용 (중간 규모)
- **API Gateway**: $50-100
- **Lambda**: $30-60
- **ECS Fargate**: $100-200
- **RDS Aurora**: $80-150
- **DynamoDB**: $20-40
- **S3**: $10-30
- **기타 서비스**: $50-100

**총 예상 비용: $340-680/월**

## 11. 결론

이 AWS Native 아키텍처는 다음과 같은 장점을 제공합니다:

1. **완전 관리형 서비스**: 서버 관리 부담 최소화
2. **자동 스케일링**: 트래픽에 따른 자동 확장/축소
3. **고가용성**: Multi-AZ 및 자동 백업
4. **보안**: IAM, VPC, 암호화 등 엔터프라이즈급 보안
5. **비용 효율성**: 사용한 만큼만 지불하는 Pay-as-you-go 모델
6. **모니터링**: CloudWatch, X-Ray를 통한 완전한 가시성

이 아키텍처를 통해 LGM Project를 확장 가능하고 안정적인 클라우드 서비스로 구현할 수 있습니다.
