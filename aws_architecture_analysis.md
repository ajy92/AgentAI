# AWS 아키텍처 다이어그램 분석 (Flow별 Markdown)

## 1. 고수준 개요 (High-Level Overview)

이 다이어그램은 크게 **Seoul Region (서울 리전)**을 중심으로 **죽전 Data Center (Jukjeon Data Center)**, **Virginia Region (버지니아 리전)**, 그리고 **MS Azure** 클라우드 환경이 상호 연결되어 있는 하이브리드 및 멀티 클라우드 아키텍처를 보여줍니다. 핵심 애플리케이션 및 데이터 처리는 서울 리전의 AWS 환경에서 이루어지며, 특정 AI/ML 모델은 버지니아 리전과 Azure에서 활용됩니다.

## 2. Seoul Region (서울 리전) 상세 분석

서울 리전은 여러 논리적 영역(LDZ)과 BDP-PRO 환경으로 구성되어 있습니다.

### 2.1. LDZ (Landing Zone) 영역

#### LDZ-IAM (Identity and Access Management)
- **MFA (Multi-Factor Authentication)**: 보안 강화를 위한 다단계 인증
- **Single Sign On**: 통합 인증 시스템
- **LDZ-Billing**: AWS 비용 관리 및 청구
- **Step Function, Lambda**: 서버리스 워크플로우 및 컴퓨팅 서비스

#### LDZ-SEC-PRO (Security Production)
- **LDZ-SEC-PRO-VPC**: 보안 프로덕션 VPC
- **Managed NodeGroup (SecuXper PRISMA)**: 보안 솔루션(SecuXper, PRISMA)이 배포된 관리형 노드 그룹

#### LDZ-Network (네트워크)
- **Transit Gateway (Central)**: 중앙 집중식 네트워크 연결 허브
- **Direct Connect Gateway (LDZ-NTW-DGW-GEN-PRD)**: 온프레미스 데이터 센터와 AWS 간 전용 네트워크 연결
- **다양한 라우팅 테이블**: Ops-RT, Sec-RT, OV-RT, Service-RT, DNS-RT, Proxy-RT (트래픽 라우팅 관리)

#### LDZ-Ops (운영)
- **개발/CI/CD 도구**: Gitlab, Nexus, Terraform, Packer, ArgoCD, Vault
- **LDZ-Observability (관측성)**:
  - **AMP (Amazon Managed Service for Prometheus)**: 컨테이너 모니터링
  - **CloudTrail**: AWS API 호출 로깅
  - **CloudWatch**: 리소스 및 애플리케이션 모니터링
  - **A-HA (Application Health Awareness)**: 애플리케이션 상태 모니터링
  - **Config**: AWS 리소스 구성 관리
  - **Elastic Search (App Log)**: 애플리케이션 로그 분석
  - **Grafana**: 시각화 대시보드
  - **Prometheus**: 시계열 데이터베이스 및 모니터링

### 2.2. BDP-PRO (Business Data Platform - Production) 영역

이 영역은 핵심 애플리케이션 및 데이터 처리 워크로드를 호스팅합니다.

#### GEN-PRD-VPC (General Production VPC)
- **Private Hosting (Route53)**: 내부 서비스 도메인 관리
  - `bedrock.region.amazonaws.com`
  - `gen.ai.shinhancard.com`
- **DAP (Data Analytics Platform)**:
  - **GEN-PRD-EKS (Elastic Kubernetes Service)**: 컨테이너 오케스트레이션
  - **Managed NodeGroups**:
    - Platform (기반 플랫폼)
    - Infra (인프라 서비스)
    - Training/Inference (AI/ML 학습 및 추론)
    - Service (일반 서비스)
- **Inbound Resolver**: VPC 내부 DNS 쿼리 해결
- **VPC Endpoint (Interface)**: AWS 서비스에 대한 프라이빗 연결
- **NAT Gateway (GEN-PRD-SBN-NAT-PR-A/C)**: 프라이빗 서브넷의 인스턴스가 인터넷에 액세스할 수 있도록 함
- **OpenSearch Service (GEN-PRD-OSS-GENP-PRI)**: 검색 및 분석 서비스

#### 애플리케이션 계층 (Ingress를 통한 접근)
- **`dap.gen.ai.shinhancard.com` Ingress**:
  - **Secondary IPs Pods (Platform)**: API, RAG (Retrieval Augmented Generation), Prompt, etc.
  - **Secondary IPs Pods (Infra)**: Chunking, Parsing, Inference, Library, MPI, Monitoring
  - **Secondary IPs Pods (Training/Inference)**: File, Batch, Training Env, Inference Env, ETC
- **`svc.gen.ai.shinhancard.com` Ingress**:
  - **Secondary IPs Pods (Platform)**: Auth, Billing, Managing, Subject, etc.

#### 데이터베이스 계층
- **Vector Store**: 벡터 데이터 저장소 (OpenSearch Service 활용)
- **Platform DB**: MariaDB (Master-Slave 구조, Write API/Read API 분리)
- **Portal/GPTS DB**: PostgreSQL (Master-Slave 구조, Write API/Read API 분리)

#### 기타 서비스
- **S3 (Level 0, Level X Embedding)**: 객체 스토리지 (데이터 저장 및 임베딩)
- **Compliance Log (VPC, LB, S3)**: 규정 준수 로깅
- **ECR (Elastic Container Registry)**: 컨테이너 이미지 저장소
- **PrivateLink**: AWS 서비스에 대한 프라이빗 연결

## 3. 죽전 Data Center (Jukjeon Data Center)

온프레미스 데이터 센터로, 서울 리전과 Direct Connect Gateway를 통해 연결됩니다.

- **사상담지원 시스템, 준법지원 시스템, 스마일**: 내부 업무 시스템
- **Shinhan Worker**: 신한 내부 워커
- **카드 클라우드 스위치**: 클라우드 연결 스위치

## 4. Virginia Region (버지니아 리전)

주로 AI/ML Foundation Model을 활용하는 영역입니다.

- **Foundation Model (ANTHROPIC, Embedding)**: 앤트로픽(Anthropic)의 파운데이션 모델 및 임베딩 서비스
- **Oregon Region**:
  - **Foundation Model (ANTHROPIC Claude X.X., Embedding)**: 앤트로픽 클로드(Claude) 모델 및 임베딩 서비스
- **GEN-PRD-VPC-US-WEST-2**:
  - **Guardrails**: AI 모델의 안전성 및 정책 준수
  - **VPC Endpoint (bedrock, bedrock-runtime)**: Amazon Bedrock 서비스에 대한 프라이빗 연결
- **Peering**: 다른 VPC와의 연결

## 5. MS Azure 클라우드

AWS와 VPN을 통해 연결되어 특정 AI 서비스를 활용합니다.

- **Eastus Region, Australiasoutheast Region**: Azure의 두 가지 리전
- **Virtual Network Gateway**: Azure 가상 네트워크와 온프레미스 또는 다른 클라우드 간 연결
- **Private Link Endpoint, Private Link Service**: Azure 서비스에 대한 프라이빗 연결
- **Azure OpenAI Service (ChatGPT/GPT-X)**: Azure에서 제공하는 OpenAI 서비스 (ChatGPT, GPT-X 모델)

## 6. 주요 연결 흐름 (Connectivity Flow)

### 6.1. 네트워크 연결
- **서울 리전 ↔ 죽전 Data Center**: Direct Connect Gateway를 통한 전용 회선 연결
- **서울 리전 ↔ Virginia Region**: VPC Peering 또는 VPN을 통한 연결
- **Virginia Region ↔ MS Azure**: VPN을 통한 연결
- **서울 리전 내부**: Transit Gateway를 통해 다양한 LDZ 및 BDP-PRO VPC들이 상호 연결

### 6.2. 애플리케이션 흐름
- **BDP-PRO 내부**: Ingress를 통해 애플리케이션 Pods로 트래픽 라우팅
- **데이터 연동**: Pods는 Vector Store 및 Platform/Portal DB와 연동
- **AI/ML 워크플로우**: 서울 리전의 DAP에서 처리된 데이터가 Virginia Region의 Foundation Model이나 MS Azure의 OpenAI Service로 전송되어 AI/ML 추론 및 처리에 활용

## 7. 아키텍처 특징

### 7.1. 보안
- **다층 보안**: LDZ-SEC-PRO를 통한 보안 관리
- **네트워크 격리**: VPC 및 서브넷을 통한 리소스 격리
- **프라이빗 연결**: VPC Endpoint, PrivateLink를 통한 안전한 서비스 접근

### 7.2. 확장성
- **EKS 기반**: 컨테이너 오케스트레이션을 통한 자동 확장
- **관리형 서비스**: AWS의 관리형 서비스 활용으로 운영 부담 감소
- **멀티 리전**: 여러 리전을 활용한 글로벌 서비스 제공

### 7.3. 고가용성
- **Master-Slave DB**: 데이터베이스 이중화
- **Multi-AZ**: 가용 영역 분산 배치
- **로드 밸런싱**: Ingress를 통한 트래픽 분산

### 7.4. AI/ML 최적화
- **전용 노드 그룹**: Training/Inference 전용 EKS 노드 그룹
- **벡터 스토어**: OpenSearch를 활용한 벡터 데이터 관리
- **멀티 클라우드 AI**: AWS Bedrock과 Azure OpenAI 서비스 동시 활용

## 8. 결론

이 아키텍처는 보안, 확장성, 고가용성을 고려하여 설계되었으며, 온프레미스, AWS, Azure를 아우르는 하이브리드 및 멀티 클라우드 전략을 보여줍니다. 특히 AI/ML 워크로드 처리를 위해 여러 클라우드 및 리전의 전문화된 서비스를 활용하는 것이 특징입니다.

### 주요 장점
- **유연성**: 다양한 클라우드 서비스 활용
- **비용 최적화**: 각 서비스의 장점을 최대한 활용
- **보안성**: 엔터프라이즈급 보안 요구사항 충족
- **확장성**: 비즈니스 성장에 따른 자동 확장 가능
