# MWAA Python Dependencies Management - Private Web Server Access

## Executive Summary

For MWAA environments with `PRIVATE_ONLY` web server access, Python dependency management requires specific approaches due to network isolation constraints. This document outlines best practices and architectural solutions for enterprise migration from on-premises Airflow to MWAA.

## Current Architecture Analysis

```mermaid
graph TB
    A[🔧 Airflow Workers<br/>🏢 On-Premises Current State]
    B[⏰ Airflow Schedulers<br/>🏢 On-Premises Current State]
    C[🌐 Airflow Webserver<br/>🏢 On-Premises Current State]
    N[📦 Nexus Repository<br/>🏢 On-Premises Current State]
    P[🐍 Python Packages<br/>🏢 On-Premises Current State]
    
    MW[🔧 MWAA Workers<br/>☁️ MWAA Target State]
    MS[⏰ MWAA Schedulers<br/>☁️ MWAA Target State]
    MWS[🌐 MWAA Webserver<br/>🔒 PRIVATE_ONLY<br/>☁️ MWAA Target State]
    VE[🔗 VPC Endpoints<br/>☁️ MWAA Target State]
    S3[📦 S3 Bucket<br/>☁️ MWAA Target State]
    NR[📦 Nexus Repository<br/>🏢 On-Premises<br/>☁️ MWAA Target State]
    PP[🐍 Python Packages<br/>📄 requirements.txt<br/>☁️ MWAA Target State]
    PZ[🔌 plugins.zip<br/>☁️ MWAA Target State]
    
    A -->|📦 Package Request| N
    B -->|📦 Package Request| N
    C -->|📦 Package Request| N
    N -->|🐍 Provide Packages| P
    
    MW -->|📦 Package Request| S3
    MS -->|📦 Package Request| S3
    MWS -.->|🔗 Optional Connection| VE
    VE -.->|🏢 Hybrid Access| NR
    S3 -->|📄 Dependencies| PP
    S3 -->|🔌 Custom Code| PZ
    
    %% Styling for colorful appearance
    style A fill:#ff6b6b,stroke:#ff4757,stroke-width:3px,color:#fff
    style B fill:#ffa502,stroke:#ff6348,stroke-width:3px,color:#fff
    style C fill:#3742fa,stroke:#2f3542,stroke-width:3px,color:#fff
    style N fill:#2ed573,stroke:#20bf6b,stroke-width:4px,color:#fff
    style P fill:#a4b0be,stroke:#747d8c,stroke-width:2px,color:#fff
    
    style MW fill:#ff6b6b,stroke:#ff4757,stroke-width:3px,color:#fff
    style MS fill:#ffa502,stroke:#ff6348,stroke-width:3px,color:#fff
    style MWS fill:#3742fa,stroke:#2f3542,stroke-width:3px,color:#fff
    style S3 fill:#00d2d3,stroke:#01a3a4,stroke-width:4px,color:#fff
    style VE fill:#5f27cd,stroke:#341f97,stroke-width:3px,color:#fff
    style NR fill:#2ed573,stroke:#20bf6b,stroke-width:3px,color:#fff
    style PP fill:#a4b0be,stroke:#747d8c,stroke-width:2px,color:#fff
    style PZ fill:#fd79a8,stroke:#e84393,stroke-width:2px,color:#fff
```

## Python Dependency Installation Options for Private Web Server Access

### ✅ **Option 1: S3-Based Dependencies (Recommended)**

**🏗️ S3 Bucket Location & Configuration:**
- **Same AWS Account**: S3 bucket MUST be in same account as MWAA
- **Same Region**: S3 bucket MUST be in same region as MWAA
- **VPC Endpoint**: S3 VPC Endpoint required for PRIVATE_ONLY access
- **Bucket Policy**: Specific IAM permissions for MWAA service role

**Complete Architecture Flow:**
```mermaid
graph TB
    DEV[👨💻 Developer Laptop/EC2]
    CICD[🚀 CI/CD Pipeline<br/>🔧 CodeBuild/Jenkins]
    
    MW[👷 MWAA Workers]
    MS[⏰ MWAA Schedulers]
    MWS[🌐 MWAA Web Server<br/>🔒 PRIVATE_ONLY]
    S3VPE[🔗 S3 VPC Endpoint<br/>🌐 com.amazonaws.s3]
    
    S3[📦 S3 Bucket<br/>mwaa-dependencies-company]
    REQ[📄 requirements.txt]
    PLUG[🔌 plugins.zip]
    CONST[📝 constraints.txt]
    
    IAM[🔑 MWAA Execution Role<br/>🛡️ AmazonMWAAServiceRolePolicy]
    
    DEV -->|📤 1. Upload via AWS CLI/SDK| S3
    CICD -->|🤖 2. Automated deployment| S3
    S3 --> REQ
    S3 --> PLUG
    S3 --> CONST
    
    MW -->|🔍 3. Read via VPC Endpoint| S3VPE
    MS -->|🔍 4. Read via VPC Endpoint| S3VPE
    MWS -->|🔍 5. Read via VPC Endpoint| S3VPE
    S3VPE -->|🔒 6. Secure access| S3
    
    IAM -.->|🔑 7. Permissions| S3
    
    %% Colorful styling
    style DEV fill:#ff6b6b,stroke:#ff4757,stroke-width:3px,color:#fff
    style CICD fill:#ffa502,stroke:#ff6348,stroke-width:3px,color:#fff
    
    style MW fill:#3742fa,stroke:#2f3542,stroke-width:3px,color:#fff
    style MS fill:#5f27cd,stroke:#341f97,stroke-width:3px,color:#fff
    style MWS fill:#00d2d3,stroke:#01a3a4,stroke-width:3px,color:#fff
    style S3VPE fill:#2ed573,stroke:#20bf6b,stroke-width:3px,color:#fff
    
    style S3 fill:#fd79a8,stroke:#e84393,stroke-width:4px,color:#fff
    style REQ fill:#a4b0be,stroke:#747d8c,stroke-width:2px,color:#fff
    style PLUG fill:#ff9ff3,stroke:#f368e0,stroke-width:2px,color:#fff
    style CONST fill:#70a1ff,stroke:#5352ed,stroke-width:2px,color:#fff
    
    style IAM fill:#ff7675,stroke:#d63031,stroke-width:3px,color:#fff
```

**Detailed Sequence Flow:**
```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor': '#ff6b6b', 'primaryTextColor': '#000000', 'actorTextColor': '#000000', 'labelTextColor': '#000000', 'loopTextColor': '#000000', 'noteTextColor': '#000000', 'activationTextColor': '#000000', 'primaryBorderColor': '#ff4757', 'lineColor': '#5f27cd', 'secondaryColor': '#00d2d3', 'tertiaryColor': '#fff', 'background': '#f8f9fa', 'mainBkg': '#ffffff', 'secondBkg': '#e3f2fd', 'tertiaryBkg': '#fff3e0', 'actorTextColor': '#000000', 'labelTextColor': '#000000'}}}%%
sequenceDiagram
    participant DEV as 👨‍💻 Developer<br/>(Laptop/EC2)
    participant CICD as 🚀 CI/CD Pipeline
    participant S3 as 📦 S3 Bucket<br/>(Same Account)
    participant VPE as 🔗 S3 VPC Endpoint
    participant MWAA as ☁️ MWAA Environment
    participant W as 👷 Workers/Schedulers
    participant WS as 🌐 Web Server<br/>(PRIVATE_ONLY)
    
    Note over DEV,WS: 📋 Dependency Management Workflow
    
    DEV->>+S3: 1. 📤 Upload requirements.txt<br/>aws s3 cp requirements.txt s3://bucket/
    Note right of S3: 💾 File stored securely
    DEV->>+S3: 2. 📦 Upload plugins.zip<br/>aws s3 cp plugins.zip s3://bucket/
    Note right of S3: 💾 Custom packages ready
    CICD->>+S3: 3. 🤖 Automated upload via CodeBuild<br/>Triggered by Git commits
    S3-->>-CICD: ✅ Upload complete
    
    Note over MWAA: 🔄 MWAA Environment Startup/Update
    
    MWAA->>+VPE: 4. 🔍 Request S3 access<br/>via VPC Endpoint (private)
    Note right of VPE: 🔒 Private network only
    VPE->>+S3: 5. 🔒 Secure S3 connection<br/>No internet routing
    S3-->>-VPE: 6. 📥 Return requirements.txt
    VPE-->>-MWAA: 7. 📝 Dependency files
    
    Note over W,WS: 🔧 Installation Phase
    
    MWAA->>+W: 8. 🔧 Install on Workers/Schedulers<br/>pip install -r requirements.txt
    Note right of W: 🔄 Installing packages...
    W-->>-MWAA: ✅ Installation complete
    
    MWAA->>+WS: 9. 🔧 Install on Web Server<br/>pip install -r requirements.txt
    Note right of WS: 🔄 Installing packages...
    WS-->>-MWAA: ✅ Installation complete
    
    Note over W,WS: 🔐 All components use same S3 source<br/>via VPC Endpoint for security
```

**🔧 Engineer Notes - S3 Configuration:**

**S3 Bucket Requirements:**
```yaml
# S3 Bucket Configuration
BucketName: mwaa-dependencies-<company>-prod
Region: us-east-1  # Same as MWAA
Encryption: AES-256
Versioning: Enabled
PublicAccess: Blocked

# Required Objects:
# ├── requirements.txt      (Python dependencies)
# ├── plugins.zip          (Custom packages/operators)
# ├── constraints.txt      (Version constraints)
# └── dags/               (DAG files)
```

**S3 Bucket Policy (Required):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/service-role/AmazonMWAA-MyEnvironment-XXXXX"
      },
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::mwaa-dependencies-<company>-prod/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/service-role/AmazonMWAA-MyEnvironment-XXXXX"
      },
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::mwaa-dependencies-<company>-prod"
    }
  ]
}
```

**MWAA Configuration (CloudFormation):**
```yaml
MWAAEnvironment:
  Type: AWS::MWAA::Environment
  Properties:
    Name: <company>-mwaa-prod
    SourceBucketArn: !GetAtt MWAADependenciesBucket.Arn
    RequirementsS3Path: requirements.txt
    PluginsS3Path: plugins.zip
    WebserverAccessMode: PRIVATE_ONLY
    NetworkConfiguration:
      SubnetIds:
        - !Ref MWAAPrivateSubnet1
        - !Ref MWAAPrivateSubnet2
      SecurityGroupIds:
        - !Ref MWAASecurityGroup
```

**S3 VPC Endpoint (Required for PRIVATE_ONLY):**
```yaml
S3VPCEndpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    VpcId: !Ref MWAAVPC
    ServiceName: !Sub 'com.amazonaws.${AWS::Region}.s3'
    VpcEndpointType: Gateway
    RouteTableIds:
      - !Ref MWAAPrivateRouteTable1
      - !Ref MWAAPrivateRouteTable2
```

**🤖 Automation Options:**

**Option A: Developer Laptop (Manual)**
```bash
# Developer workflow
aws s3 cp requirements.txt s3://mwaa-dependencies-<company>-prod/
aws s3 cp plugins.zip s3://mwaa-dependencies-<company>-prod/

# Update MWAA environment
aws mwaa update-environment --name <company>-mwaa-prod
```

**Option B: CI/CD Pipeline (Recommended)**
```yaml
# CodeBuild buildspec.yml
version: 0.2
phases:
  build:
    commands:
      - echo "Building Python dependencies"
      - pip wheel -r requirements.txt -w wheels/
      - zip -r plugins.zip plugins/
      - aws s3 cp requirements.txt s3://mwaa-dependencies-<company>-prod/
      - aws s3 cp plugins.zip s3://mwaa-dependencies-<company>-prod/
      - aws mwaa update-environment --name <company>-mwaa-prod
```

**Option C: Lambda-based Automation**
```python
# Lambda function triggered by Git webhook
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    mwaa = boto3.client('mwaa')
    
    # Download from Git, build dependencies
    # Upload to S3
    s3.upload_file('requirements.txt', 'mwaa-dependencies-<company>-prod', 'requirements.txt')
    
    # Update MWAA
    mwaa.update_environment(Name='<company>-mwaa-prod')
```

**Implementation:**
- **requirements.txt**: Standard pip requirements file uploaded to S3
- **plugins.zip**: Custom Python packages and wheels
- **constraints.txt**: Pin specific versions to avoid conflicts
- **S3 VPC Endpoint**: Mandatory for PRIVATE_ONLY web server access

### ✅ **Option 2: VPC Endpoint Service to Nexus (Advanced)**

**Detailed Architecture Flow:**
```mermaid
graph TB
    MW[🔧 MWAA Workers]
    MS[⏰ MWAA Schedulers] 
    MWS[🌐 MWAA Web Server<br/>🔒 PRIVATE_ONLY]
    VPE[🔗 VPC Endpoint<br/>🌐 nexus.company.com]
    
    VPES[🔌 VPC Endpoint Service]
    NLB[⚖️ Network Load Balancer<br/>🎯 Target: On-premises Nexus]
    DX[🌉 Direct Connect Gateway<br/>🔒 or VPN Connection]
    
    FW[🔥 Corporate Firewall]
    NR[📦 Nexus Repository<br/>🌐 nexus.company.local:8081]
    DNS[🌐 Internal DNS<br/>📋 nexus.company.com]
    
    MW -->|🔍 Package Request| VPE
    MS -->|🔍 Package Request| VPE
    MWS -->|🔍 Package Request| VPE
    VPE -.->|🔗 AWS PrivateLink| VPES
    VPES -->|⚖️ Load Balance| NLB
    NLB -.->|🌉 Direct Connect/VPN| DX
    DX -->|🔥 Firewall Rules| FW
    FW -->|📦 Repository Access| NR
    DNS -.->|🌐 DNS Resolution| NR
    
    %% Colorful styling
    style MW fill:#ff6b6b,stroke:#ff4757,stroke-width:3px,color:#fff
    style MS fill:#ffa502,stroke:#ff6348,stroke-width:3px,color:#fff
    style MWS fill:#3742fa,stroke:#2f3542,stroke-width:3px,color:#fff
    style VPE fill:#5f27cd,stroke:#341f97,stroke-width:3px,color:#fff
    
    style VPES fill:#00d2d3,stroke:#01a3a4,stroke-width:3px,color:#fff
    style NLB fill:#2ed573,stroke:#20bf6b,stroke-width:3px,color:#fff
    style DX fill:#fd79a8,stroke:#e84393,stroke-width:3px,color:#fff
    
    style FW fill:#ff7675,stroke:#d63031,stroke-width:3px,color:#fff
    style NR fill:#a4b0be,stroke:#747d8c,stroke-width:4px,color:#fff
    style DNS fill:#70a1ff,stroke:#5352ed,stroke-width:2px,color:#fff
```

**Requirements:**
- **On-premises Nexus**: Already installed in customer data center
- **Direct Connect/VPN**: Existing connection to AWS
- **Network Load Balancer**: Deploy in AWS, target on-premises Nexus
- **VPC Endpoint Service**: Create in AWS account
- **VPC Endpoint**: Create in MWAA VPC
- **DNS Configuration**: Route nexus.customer.com to VPC Endpoint
- **Security Groups**: Allow HTTPS/HTTP traffic
- **Firewall Rules**: Allow AWS NLB to reach on-premises Nexus

### ❌ **Option 3: Direct Internet Access (Not Applicable)**
Not possible with `PRIVATE_ONLY` web server access mode.

## Default MWAA Libraries

### Pre-installed Packages
MWAA comes with a base set of Python packages. To determine what's installed:

```python
# Create a DAG to list installed packages
import subprocess
import logging

def list_installed_packages():
    result = subprocess.run(['pip', 'list'], capture_output=True, text=True)
    logging.info(f"Installed packages:\n{result.stdout}")
    return result.stdout

# Use aws-mwaa-local-runner for local testing
```

### Version Compatibility Matrix

| MWAA Version | Python Version | Apache Airflow | Key Libraries |
|--------------|----------------|----------------|---------------|
| 2.8.1 | 3.11 | 2.8.1 | boto3, pandas, numpy |
| 2.7.2 | 3.10 | 2.7.2 | boto3, pandas, numpy |
| 2.6.3 | 3.10 | 2.6.3 | boto3, pandas, numpy |

## Recommended Implementation Strategy

### Phase 1: S3-Based Approach (Immediate)

```mermaid
flowchart TD
    A[🔍 Audit Current Dependencies<br/>📋 Inventory existing packages<br/>🔢 Document versions] --> B[📝 Create requirements.txt<br/>📄 List all dependencies<br/>🔒 Pin specific versions]
    B --> C[🧪 Test with aws-mwaa-local-runner<br/>🐳 Docker environment<br/>✅ Validate compatibility]
    C --> D[📤 Upload to S3<br/>☁️ S3 bucket deployment<br/>🔐 Secure storage]
    D --> E[🚀 Deploy MWAA Environment<br/>⚙️ Infrastructure setup<br/>🌐 Private web server]
    E --> F[🔬 Validate Dependencies<br/>🧪 Test package imports<br/>📊 Monitor logs]
    
    F --> G{🤔 All Dependencies Work?<br/>✨ No import errors<br/>🎯 Full functionality}
    G -->|✅ Yes| H[🎉 Production Deployment<br/>🚀 Go live<br/>📈 Monitor performance]
    G -->|❌ No| I[🛠️ Create plugins.zip<br/>📦 Custom packages<br/>🔧 Build wheels]
    I --> C
    
    %% Vibrant color styling
    style A fill:#ff6b6b,stroke:#ff4757,stroke-width:4px,color:#fff
    style B fill:#4ecdc4,stroke:#26d0ce,stroke-width:3px,color:#fff
    style C fill:#45b7d1,stroke:#3742fa,stroke-width:3px,color:#fff
    style D fill:#96ceb4,stroke:#6c5ce7,stroke-width:3px,color:#fff
    style E fill:#feca57,stroke:#ff9f43,stroke-width:3px,color:#fff
    style F fill:#ff9ff3,stroke:#f368e0,stroke-width:3px,color:#fff
    style G fill:#fd79a8,stroke:#e84393,stroke-width:4px,color:#fff
    style H fill:#2ed573,stroke:#20bf6b,stroke-width:4px,color:#fff
    style I fill:#70a1ff,stroke:#5352ed,stroke-width:3px,color:#fff
```

### Phase 2: VPC Endpoint Service (Long-term)

**Nexus Location:** On-premises in customer's corporate data center

**Complete Architecture Flow:**
```mermaid
sequenceDiagram
    participant MWAA as MWAA Environment<br/>(Private Web Server)
    participant VPE as VPC Endpoint<br/>(AWS VPC)
    participant VPES as VPC Endpoint Service<br/>(AWS Account)
    participant NLB as Network Load Balancer<br/>(AWS Account)
    participant DX as Direct Connect<br/>or VPN
    participant Nexus as Nexus Repository<br/>(On-Premises)
    
    Note over MWAA,Nexus: MWAA Dependency Installation Process
    
    MWAA->>VPE: 1. Request Python package<br/>pip install package-name
    Note over VPE: VPC Endpoint in MWAA VPC<br/>Private DNS resolution
    
    VPE->>VPES: 2. Route to VPC Endpoint Service<br/>via AWS PrivateLink
    Note over VPES: VPC Endpoint Service<br/>in separate AWS account
    
    VPES->>NLB: 3. Forward to Network Load Balancer<br/>Load balance across targets
    Note over NLB: NLB distributes traffic<br/>Health checks enabled
    
    NLB->>DX: 4. Route to on-premises<br/>via Direct Connect/VPN
    Note over DX: Secure connection to<br/>customer corporate network
    
    DX->>Nexus: 5. Connect to Nexus Repository<br/>HTTP/HTTPS on port 8081
    Note over Nexus: On-premises Nexus<br/>Python package repository
    
    Nexus-->>DX: 6. Return Python package
    DX-->>NLB: 7. Package data
    NLB-->>VPES: 8. Package data
    VPES-->>VPE: 9. Package data via PrivateLink
    VPE-->>MWAA: 10. Install package on MWAA<br/>Workers/Schedulers/Web Server
    
    Note over MWAA: All MWAA components can now<br/>access on-premises Nexus
```

**Implementation Steps:**
```mermaid
flowchart TD
    A[🎨 Design VPC Endpoint Service<br/>📐 Architecture planning<br/>🔧 Service specifications] --> B[⚖️ Deploy Network Load Balancer<br/>🌐 AWS Account setup<br/>🎯 Target configuration]
    B --> C[🔌 Configure VPC Endpoint Service<br/>🔗 NLB as target<br/>🛡️ Security policies]
    C --> D[🔗 Create VPC Endpoint in MWAA VPC<br/>📍 Endpoint Service connection<br/>🏠 Private subnet placement]
    D --> E[🌐 Configure DNS resolution<br/>📋 Nexus hostname mapping<br/>🔍 Route 53 setup]
    E --> F[📝 Update MWAA requirements.txt<br/>🔗 Nexus URL configuration<br/>📦 Package source update]
    F --> G[🧪 Test Nexus Connectivity<br/>🔍 Connection validation<br/>📊 Performance testing]
    G --> H[🚀 Production Migration<br/>✅ Go-live deployment<br/>🎉 Success celebration]
    
    %% Vibrant color styling for visual appeal
    style A fill:#ff6b6b,stroke:#ff4757,stroke-width:4px,color:#fff
    style B fill:#4ecdc4,stroke:#26d0ce,stroke-width:3px,color:#fff
    style C fill:#45b7d1,stroke:#3742fa,stroke-width:3px,color:#fff
    style D fill:#96ceb4,stroke:#6c5ce7,stroke-width:3px,color:#fff
    style E fill:#feca57,stroke:#ff9f43,stroke-width:3px,color:#fff
    style F fill:#ff9ff3,stroke:#f368e0,stroke-width:3px,color:#fff
    style G fill:#fd79a8,stroke:#e84393,stroke-width:3px,color:#fff
    style H fill:#2ed573,stroke:#20bf6b,stroke-width:4px,color:#fff
```

## Implementation Details

### S3-Based Dependencies Configuration

**🔧 Worker/Scheduler Configuration:**
MWAA automatically configures workers and schedulers to use S3 bucket specified in environment configuration. No manual configuration required on workers.

**requirements.txt example:**
```txt
# Core data processing
pandas==1.5.3
numpy==1.24.3
requests==2.31.0

# AWS SDK (usually pre-installed, but pin version)
boto3==1.26.137
botocore==1.29.137

# Custom internal packages (via plugins.zip)
--find-links /usr/local/airflow/plugins
customer-data-utils==1.2.0

# Nexus packages (if using hybrid approach)
--index-url https://nexus.customer.com/repository/pypi-proxy/simple/
--trusted-host nexus.customer.com
customer-proprietary-lib==2.1.0
```

**constraints.txt example:**
```txt
# Pin versions to avoid conflicts with MWAA defaults
apache-airflow==2.8.1
boto3>=1.26.0,<1.27.0
pandas>=1.5.0,<2.0.0
numpy>=1.24.0,<1.25.0
```

**plugins.zip structure:**
```
plugins/
├── customer_data_utils-1.2.0-py3-none-any.whl
├── custom_operators/
│   ├── __init__.py
│   └── customer_operator.py
├── hooks/
│   ├── __init__.py
│   └── nexus_hook.py
├── sensors/
│   ├── __init__.py
│   └── customer_sensor.py
└── utils/
    ├── __init__.py
    └── customer_utils.py
```

**🚀 Automation Scripts:**

**build_dependencies.sh:**
```bash
#!/bin/bash
# Automated dependency building and upload

set -e

BUCKET="mwaa-dependencies-customer-prod"
ENVIRONMENT="customer-mwaa-prod"

echo "Building Python wheels..."
mkdir -p wheels
pip wheel -r requirements.txt -w wheels/

echo "Creating plugins.zip..."
zip -r plugins.zip plugins/

echo "Uploading to S3..."
aws s3 cp requirements.txt s3://$BUCKET/
aws s3 cp plugins.zip s3://$BUCKET/
aws s3 cp constraints.txt s3://$BUCKET/

echo "Updating MWAA environment..."
aws mwaa update-environment --name $ENVIRONMENT

echo "Deployment complete!"
```

### VPC Endpoint Service Configuration

**CloudFormation Template:**
```yaml
VPCEndpointService:
  Type: AWS::EC2::VPCEndpointService
  Properties:
    NetworkLoadBalancerArns:
      - !Ref NexusNetworkLoadBalancer
    AcceptanceRequired: false

MWAAVPCEndpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    VpcId: !Ref MWAAVpc
    ServiceName: !Sub 'com.amazonaws.vpce.${AWS::Region}.${VPCEndpointService}'
    VpcEndpointType: Interface
    SubnetIds:
      - !Ref MWAAPrivateSubnet1
      - !Ref MWAAPrivateSubnet2
```

## Dependency Conflict Resolution

### Testing Strategy with aws-mwaa-local-runner

```bash
# 1. Clone the local runner
git clone https://github.com/aws/aws-mwaa-local-runner.git
cd aws-mwaa-local-runner

# 2. Configure your requirements
cp your-requirements.txt docker/config/requirements.txt

# 3. Test locally
./mwaa-local-env build-image
./mwaa-local-env start

# 4. Validate dependencies
docker exec -it mwaa_local_scheduler pip list
```

### Conflict Resolution Process

```mermaid
flowchart TD
    A[🚨 Identify Conflict<br/>📋 Package version mismatch<br/>⚠️ Import errors detected] --> B[🔍 Check MWAA Default Versions<br/>📊 Review pre-installed packages<br/>🐍 Python 3.11 compatibility]
    B --> C[📌 Pin Specific Versions<br/>📄 Update requirements.txt<br/>🔒 Lock dependency versions]
    C --> D[🧪 Test with Local Runner<br/>🐳 Docker environment<br/>✅ Validate functionality]
    D --> E{🤔 Conflicts Resolved?<br/>✨ All packages working<br/>🎯 No import errors}
    E -->|❌ No| F[🛠️ Create Custom Wheel<br/>📦 Build plugins.zip<br/>🔧 Package isolation]
    E -->|✅ Yes| G[🚀 Deploy to MWAA<br/>☁️ Production environment<br/>🎉 Success!]
    F --> D
    
    %% Colorful styling for visual appeal
    style A fill:#ff6b6b,stroke:#ff4757,stroke-width:4px,color:#fff
    style B fill:#4ecdc4,stroke:#26d0ce,stroke-width:3px,color:#fff
    style C fill:#45b7d1,stroke:#3742fa,stroke-width:3px,color:#fff
    style D fill:#96ceb4,stroke:#6c5ce7,stroke-width:3px,color:#fff
    style E fill:#feca57,stroke:#ff9f43,stroke-width:4px,color:#fff
    style F fill:#ff9ff3,stroke:#f368e0,stroke-width:3px,color:#fff
    style G fill:#2ed573,stroke:#20bf6b,stroke-width:4px,color:#fff
```

## Security Considerations

### Network Security
- **VPC Endpoints**: Ensure proper security group rules
- **S3 Access**: Use VPC S3 endpoints for secure access (mandatory for PRIVATE_ONLY)
- **IAM Policies**: Least privilege access to S3 and Nexus
- **S3 Bucket Policy**: Restrict access to MWAA service role only
- **Encryption**: S3 bucket encryption with KMS keys
- **Network Isolation**: All traffic stays within AWS backbone via VPC endpoints

### Package Security
- **Vulnerability Scanning**: Scan all packages before deployment
- **Integrity Checks**: Verify package checksums
- **Version Pinning**: Pin all dependency versions

## Monitoring and Troubleshooting

### CloudWatch Metrics
- Monitor MWAA environment health
- Track dependency installation failures
- Set up alerts for package conflicts

### Troubleshooting Common Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| Package not found | Import errors in DAGs | Add to requirements.txt or plugins.zip |
| Version conflicts | Runtime errors | Pin specific versions |
| Network timeouts | Installation failures | Check VPC endpoints |
| Large package size | Slow startup | Optimize plugins.zip |

## Cost Optimization

### S3 Storage Costs
- Use S3 Intelligent Tiering for requirements files
- Compress plugins.zip files
- Regular cleanup of old versions

### VPC Endpoint Costs
- Monitor data transfer costs
- Consider Regional vs AZ endpoints
- Optimize for usage patterns


### 🔍 How to Discover Pre-installed Packages

**🔧 Engineer Notes - Package Discovery Methods:**

**Method 1: Create Discovery DAG** ⭐ **Best for Production Environments**

### 🎨 **DAG Installation & Execution Architecture**

```mermaid
sequenceDiagram
    participant Dev as 👨‍💻 Developer
    participant Git as 📚 Git Repository
    participant S3 as 📦 S3 DAG Bucket
    participant MWAA as 🚁 MWAA Environment
    participant Scheduler as ⏰ Airflow Scheduler
    participant Worker as 🔧 Airflow Worker
    participant WebUI as 🌐 Airflow Web UI
    participant CW as 📊 CloudWatch Logs
    
    Note over Dev,CW: 🚀 Phase 1: DAG Development & Deployment
    
    Dev->>Git: 1. Create discover_packages.py
    Note right of Dev: DAG Definition:<br/>• Python file with DAG object<br/>• PythonOperator for pip list<br/>• Schedule: None (manual trigger)<br/>• Logging configuration
    
    Dev->>Git: 2. Commit & Push DAG
    Note right of Git: DAG Code Structure:<br/>from airflow import DAG<br/>from airflow.operators.python import PythonOperator<br/>def list_packages(): subprocess.run(['pip', 'list'])
    
    Git->>S3: 3. Deploy to S3 DAG Bucket
    Note right of S3: S3 Bucket Structure:<br/>s3://mwaa-dags-customer-prod/<br/>├── discover_packages.py<br/>├── other_dags/<br/>└── __pycache__/
    
    S3->>MWAA: 4. MWAA Auto-detects New DAG
    Note right of MWAA: MWAA Environment:<br/>• Polls S3 every 30 seconds<br/>• Validates DAG syntax<br/>• Imports Python modules<br/>• Updates DAG registry
    
    MWAA->>Scheduler: 5. DAG Registered in Scheduler
    Note right of Scheduler: Scheduler Process:<br/>• DAG appears in UI<br/>• Schedule: None (manual only)<br/>• Status: Available for trigger<br/>• Dependencies: Validated
    
    Note over Dev,CW: ⚡ Phase 2: Manual DAG Execution
    
    Dev->>WebUI: 6. Access Airflow Web UI
    Note right of WebUI: PRIVATE_ONLY Access:<br/>• VPC-only access (no internet)<br/>• IAM authentication required<br/>• HTTPS endpoint in private subnet<br/>• URL: https://mwaa-webserver-xxx.airflow.region.amazonaws.com
    
    WebUI->>Dev: 7. Show Available DAGs
    Note right of Dev: DAG List View:<br/>• discover_mwaa_packages<br/>• Status: Ready<br/>• Last Run: Never<br/>• Schedule: None
    
    Dev->>WebUI: 8. Trigger DAG Manually
    Note right of WebUI: Manual Trigger:<br/>• Click "Trigger DAG" button<br/>• Optional: Add run configuration<br/>• Creates DAG Run instance<br/>• Assigns unique run_id
    
    WebUI->>Scheduler: 9. Create DAG Run
    Note right of Scheduler: DAG Run Creation:<br/>• run_id: manual__2024-01-15T10:30:00<br/>• state: running<br/>• execution_date: 2024-01-15T10:30:00<br/>• Tasks queued for execution
    
    Scheduler->>Worker: 10. Queue Task for Execution
    Note right of Worker: Task Queuing:<br/>• Task: list_packages<br/>• Queue: default<br/>• Priority: normal<br/>• Worker assignment: automatic
    
    Note over Dev,CW: 🔍 Phase 3: Package Discovery Execution
    
    Worker->>Worker: 11. Execute pip list Command
    Note right of Worker: Task Execution Environment:<br/>• Python 3.11 (MWAA 2.8.1)<br/>• Virtual environment: /usr/local/airflow<br/>• Command: subprocess.run(['pip', 'list'])<br/>• Capture stdout and stderr
    
    Worker->>Worker: 12. Process Package List
    Note right of Worker: Package Processing:<br/>• Parse pip list output<br/>• Format for logging<br/>• Include version numbers<br/>• Add timestamp and metadata
    
    Worker->>CW: 13. Log Results to CloudWatch
    Note right of CW: CloudWatch Logging:<br/>• Log Group: /aws/amazonmwaa/customer-mwaa-prod<br/>• Log Stream: dag_id=discover_packages/run_id=manual__xxx<br/>• Log Level: INFO<br/>• Content: Complete package list with versions
    
    Worker->>Scheduler: 14. Report Task Success
    Note right of Scheduler: Task Completion:<br/>• Task state: success<br/>• Duration: ~30 seconds<br/>• Return value: package list string<br/>• Logs: Available in CloudWatch
    
    Scheduler->>WebUI: 15. Update DAG Run Status
    Note right of WebUI: UI Status Update:<br/>• DAG Run: success<br/>• Task: list_packages (green)<br/>• Duration: 00:00:30<br/>• Logs: Click to view
    
    Note over Dev,CW: 📊 Phase 4: Results Access & Analysis
    
    Dev->>WebUI: 16. View Task Logs in UI
    Note right of WebUI: Airflow UI Logs:<br/>• Real-time log streaming<br/>• Syntax highlighting<br/>• Download log files<br/>• Filter by log level
    
    Dev->>CW: 17. Access CloudWatch Logs
    Note right of CW: CloudWatch Features:<br/>• Log retention: 30 days default<br/>• Search and filter capabilities<br/>• Export to S3 for long-term storage<br/>• CloudWatch Insights queries
    
    CW->>Dev: 18. Package Inventory Results
    Note right of Dev: Discovered Packages:<br/>• apache-airflow==2.8.1<br/>• boto3==1.34.x<br/>• pandas==2.0.x<br/>• numpy==1.24.x<br/>• 200+ total packages
    
    Note over Dev,CW: 🔄 Phase 5: Automated Scheduling (Optional)
    
    Dev->>Git: 19. Update DAG for Automation
    Note right of Git: Schedule Configuration:<br/>schedule_interval='@weekly'<br/>or<br/>schedule_interval=cron('0 2 * * 1')<br/>catchup=False
    
    Git->>S3: 20. Deploy Updated DAG
    S3->>MWAA: 21. Auto-update DAG Definition
    MWAA->>Scheduler: 22. Schedule Weekly Runs
    Note right of Scheduler: Automated Execution:<br/>• Runs every Monday at 2 AM<br/>• No manual intervention needed<br/>• Results logged to CloudWatch<br/>• Package inventory stays current
```

### 🏗️ **DAG Architecture Components**

```mermaid
graph TB
    %% Development Environment
        DEV[👨‍💻 Developer Workstation]:::dev
        IDE[💻 VS Code / PyCharm]:::dev
        GIT[📚 Git Repository]:::dev
    
    %% AWS S3 Storage
        S3BUCKET[📦 S3 DAG Bucket<br/>mwaa-dags-customer-prod]:::s3
        DAGFILE[📄 discover_packages.py]:::s3
        PLUGINS[🔌 plugins.zip]:::s3
        REQS[📋 requirements.txt]:::s3
    
    %% MWAA Environment - Private Subnets
        SCHEDULER[⏰ Airflow Scheduler<br/>• DAG parsing<br/>• Task scheduling<br/>• Dependency resolution]:::scheduler
        WEBSERVER[🌐 Airflow Web Server<br/>• PRIVATE_ONLY access<br/>• DAG management UI<br/>• Log viewing]:::webserver
        WORKER[🔧 Airflow Worker<br/>• Task execution<br/>• Package discovery<br/>• pip list command]:::worker
        
        PYTHON[🐍 Python 3.11 Runtime]:::runtime
        PACKAGES[📦 Pre-installed Packages<br/>• Apache Airflow 2.8.1<br/>• boto3, pandas, numpy<br/>• 200+ libraries]:::runtime
        VENV[🏠 Virtual Environment<br/>/usr/local/airflow]:::runtime
    
    %% AWS CloudWatch
        LOGGROUP[📊 Log Group<br/>/aws/amazonmwaa/customer-mwaa-prod]:::logs
        LOGSTREAM[📝 Log Stream<br/>dag_id=discover_packages]:::logs
        METRICS[📈 CloudWatch Metrics<br/>• DAG success rate<br/>• Task duration<br/>• Error counts]:::logs
    
    %% Access & Monitoring
        CONSOLE[🖥️ AWS Console<br/>• MWAA management<br/>• CloudWatch access]:::access
        CLI[⌨️ AWS CLI<br/>• DAG deployment<br/>• Log retrieval]:::access
    
    DEV -->|📝 Code Development| IDE
    IDE -->|📤 Version Control| GIT
    GIT -->|🚀 Deploy DAGs| S3BUCKET
    S3BUCKET -->|📄 DAG Files| DAGFILE
    S3BUCKET -->|🔌 Plugin Files| PLUGINS
    S3BUCKET -->|📋 Dependencies| REQS
    
    DAGFILE -->|📅 Schedule Tasks| SCHEDULER
    SCHEDULER -->|🌐 Web Interface| WEBSERVER
    SCHEDULER -->|🔧 Execute Tasks| WORKER
    
    WORKER -->|🐍 Runtime Environment| PYTHON
    PYTHON -->|📦 Package Access| PACKAGES
    PYTHON -->|🏠 Isolated Environment| VENV
    
    WORKER -->|📝 Task Logs| LOGGROUP
    LOGGROUP -->|📊 Structured Logs| LOGSTREAM
    LOGGROUP -->|📈 Performance Data| METRICS
    
    WEBSERVER -->|🖥️ Management UI| CONSOLE
    LOGGROUP -->|📊 Monitoring| CONSOLE
    CLI -->|🚀 Automated Deploy| S3BUCKET
    CLI -->|📝 Log Access| LOGGROUP
    
    classDef dev fill:#FF6B6B,stroke:#FF4757,stroke-width:4px,color:#fff
    classDef s3 fill:#4ECDC4,stroke:#26D0CE,stroke-width:4px,color:#fff
    classDef scheduler fill:#45B7D1,stroke:#3742FA,stroke-width:4px,color:#fff
    classDef webserver fill:#96CEB4,stroke:#6C5CE7,stroke-width:4px,color:#fff
    classDef worker fill:#FECA57,stroke:#FF9F43,stroke-width:4px,color:#fff
    classDef runtime fill:#FF9FF3,stroke:#F368E0,stroke-width:4px,color:#fff
    classDef logs fill:#54A0FF,stroke:#2F3542,stroke-width:4px,color:#fff
    classDef access fill:#5F27CD,stroke:#341F97,stroke-width:4px,color:#fff
```

### 🔍 **Key Benefits Highlighted in Architecture:**

**🎯 Real Environment Execution:**
- DAG runs in actual MWAA production environment
- Uses same Python runtime and virtual environment as production workloads
- Discovers exact package versions and dependencies

**🔒 PRIVATE_ONLY Compatible:**
- No internet access required for execution
- All communication within AWS private network
- S3 VPC endpoints enable DAG deployment
- CloudWatch logging works in private subnets

**📊 Comprehensive Logging:**
- All output captured in CloudWatch Logs
- Structured logging with timestamps and metadata
- Long-term retention and analysis capabilities
- Integration with CloudWatch Insights for queries

**⚡ Flexible Execution:**
- Manual trigger for on-demand discovery
- Automated scheduling for regular inventory
- No additional infrastructure required
- Scales with MWAA worker capacity

**Use Case:** When you need to discover packages in actual MWAA production environment
**Pros:** 
- Real production environment data
- Works with PRIVATE_ONLY web server access
- No additional infrastructure needed
- Can be scheduled for regular inventory updates

**Cons:**
- Requires DAG deployment to production
- Takes time to execute (DAG run)
- Limited to runtime discovery

**Implementation Notes:**
- Deploy as one-time discovery DAG
- Use PythonOperator for package listing
- Output goes to Airflow logs (CloudWatch)
- Can export results to S3 for analysis

- 
## Conclusion

For MWAA with `PRIVATE_ONLY` web server access, the S3-based approach using requirements.txt and plugins.zip is the most practical immediate solution. The VPC Endpoint Service approach provides long-term integration with existing Nexus infrastructure but requires additional network architecture investment.

The key to success is thorough testing with aws-mwaa-local-runner and careful dependency version management to avoid conflicts with MWAA's pre-installed packages.


### Web Server API Access from On-Premises

## 🏗️ **Network Architecture Overview - Transit Gateway Solution**

```mermaid
graph TB
    ONPREM[🏢 On-Premises Applications<br/>• REST API clients<br/>• Workflow triggers<br/>• Monitoring systems]:::onprem
    FIREWALL[🔥 Corporate Firewall<br/>• Outbound HTTPS allowed<br/>• VPN/DX routing]:::onprem
    
    VPN[🔗 VPN/Direct Connect<br/>• Secure connection<br/>• BGP routing<br/>• High availability]:::vpn
    
    TGW[🌐 Transit Gateway<br/>• Cross-VPC routing<br/>• On-premises connectivity<br/>• Route table management]:::tgw
    
    MWAAVPC[🏠 MWAA VPC<br/>• Private subnets<br/>• Same account<br/>• Connected to TGW]:::vpc
    
    WORKERS[🔧 MWAA Workers<br/>• Task execution<br/>• Private subnet only<br/>• No internet access]:::workers
    
    WEBSERVER[🌐 MWAA Web Server<br/>• Airflow REST API<br/>• PRIVATE_ONLY mode<br/>• IAM authentication]:::webserver
    
    DNS[🌐 Route 53 Private Zone<br/>• Internal DNS resolution<br/>• mwaa.company.internal<br/>• Cross-VPC association]:::dns
    
    ONPREM --> FIREWALL
    FIREWALL --> VPN
    VPN --> TGW
    TGW --> MWAAVPC
    MWAAVPC --> WORKERS
    MWAAVPC --> WEBSERVER
    
    DNS -.->|DNS Resolution| WEBSERVER
    ONPREM -.->|DNS Query| DNS
    
    classDef onprem fill:#FF6B6B,stroke:#FF4757,stroke-width:4px,color:#fff
    classDef vpn fill:#4ECDC4,stroke:#26D0CE,stroke-width:4px,color:#fff
    classDef tgw fill:#45B7D1,stroke:#3742FA,stroke-width:4px,color:#fff
    classDef vpc fill:#96CEB4,stroke:#6C5CE7,stroke-width:4px,color:#fff
    classDef workers fill:#FECA57,stroke:#FF9F43,stroke-width:4px,color:#fff
    classDef webserver fill:#FF9FF3,stroke:#F368E0,stroke-width:4px,color:#fff
    classDef dns fill:#54A0FF,stroke:#2F3542,stroke-width:4px,color:#fff
```

## 🔄 **Detailed Connection Sequence**

```mermaid
sequenceDiagram
    participant OnPrem as 🏢 On-Premises App
    participant Firewall as 🔥 Corporate Firewall
    participant VPN as 🔗 VPN/Direct Connect
    participant TGW as 🌐 Transit Gateway
    participant MWAAVPC as 🏠 MWAA VPC
    participant DNS as 🌐 Route 53 DNS
    participant WebServer as 🌐 MWAA Web Server
    participant IAM as 👤 AWS IAM
    
    Note over OnPrem,IAM: 🔐 Phase 1: Authentication & DNS Resolution
    
    OnPrem->>IAM: 1. Assume IAM Role for API Access
    Note right of OnPrem: Authentication Method:<br/>• IAM User with API keys<br/>• IAM Role with STS assume<br/>• Service account credentials
    
    IAM-->>OnPrem: 2. Return Temporary Credentials
    Note right of IAM: STS Response:<br/>• AccessKeyId: ASIA...<br/>• SecretAccessKey: temp-secret<br/>• SessionToken: session-token<br/>• Expiration: 1 hour
    
    OnPrem->>DNS: 3. DNS Query for MWAA Web Server
    Note right of OnPrem: DNS Query:<br/>• Hostname: mwaa.company.internal<br/>• Query Type: A record<br/>• Resolver: Corporate DNS
    
    DNS-->>OnPrem: 4. Return MWAA Web Server IP
    Note right of DNS: DNS Response:<br/>• IP: 10.1.10.100 (MWAA VPC)<br/>• TTL: 300 seconds<br/>• Private IP address
    
    Note over OnPrem,IAM: 🌐 Phase 2: Network Routing & Connection
    
    OnPrem->>Firewall: 5. HTTPS Request to MWAA API
    Note right of OnPrem: API Request:<br/>• URL: https://mwaa.company.internal<br/>• Method: POST /dags/my_dag/dagRuns<br/>• Headers: Authorization, Content-Type<br/>• Body: DAG run configuration
    
    Firewall->>VPN: 6. Route via VPN/Direct Connect
    Note right of Firewall: Routing Rules:<br/>• Destination: 10.1.0.0/16 (MWAA VPC)<br/>• Next hop: VPN Gateway<br/>• Protocol: HTTPS (443)<br/>• Source: On-premises subnet
    
    VPN->>TGW: 7. Forward to Transit Gateway
    Note right of VPN: VPN Connection:<br/>• BGP routing enabled<br/>• Advertised routes: 10.1.0.0/16<br/>• Connection status: UP<br/>• Tunnel redundancy: Active
    
    TGW->>MWAAVPC: 8. Route to MWAA VPC
    Note right of TGW: TGW Route Table:<br/>• Route: 10.1.0.0/16 → MWAA VPC<br/>• Propagation: Enabled<br/>• Association: MWAA VPC attachment<br/>• Status: Active
    
    MWAAVPC->>WebServer: 9. Direct connection to MWAA Web Server
    Note right of MWAAVPC: VPC Routing:<br/>• Security Group: Allow 443 from on-premises<br/>• NACL: Allow HTTPS traffic<br/>• Private subnet routing<br/>• No NAT Gateway required
    
    Note over OnPrem,IAM: 🔍 Phase 3: API Processing & Response
    
    WebServer->>IAM: 10. Validate API Request
    Note right of WebServer: API Validation:<br/>• AWS Signature V4 verification<br/>• IAM policy evaluation<br/>• Resource-based permissions<br/>• Rate limiting check
    
    IAM-->>WebServer: 11. Authorization Result
    Note right of IAM: Authorization Check:<br/>• Action: airflow:CreateDagRun<br/>• Resource: arn:aws:airflow:region:account:environment/env-name<br/>• Principal: assumed role ARN<br/>• Result: Allow/Deny
    
    WebServer->>WebServer: 12. Process DAG Trigger Request
    Note right of WebServer: DAG Processing:<br/>• Validate DAG exists<br/>• Check DAG is not paused<br/>• Create DAG run instance<br/>• Queue tasks for execution
    
    WebServer-->>MWAAVPC: 13. Return API Response
    Note right of WebServer: API Response:<br/>• Status: 200 OK<br/>• Body: DAG run details<br/>• Headers: Content-Type, CORS<br/>• Run ID: manual__2024-01-15T10:30:00
    
    MWAAVPC-->>TGW: 14. Route response back
    TGW-->>VPN: 15. Forward via Transit Gateway
    VPN-->>Firewall: 16. Return via VPN/Direct Connect
    Firewall-->>OnPrem: 17. Deliver API Response
    
    Note right of OnPrem: Response Received:<br/>• DAG triggered successfully<br/>• Run ID for tracking<br/>• Execution timestamp<br/>• Direct MWAA access confirmed
```

