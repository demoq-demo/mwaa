#### 🔧 **Configuration Solutions by Scenario**

**1. Gitlab Runners (On-Premises) → MWAA REST API**

### 🏗️ **Architecture Overview**

```mermaid
graph TB
    GITLAB[🦊 Gitlab CI/CD Runners<br/>• Pipeline execution<br/>• REST API calls<br/>• Automated workflows]:::gitlab
    FIREWALL[🔥 Corporate Firewall<br/>• Outbound HTTPS rules<br/>• Security policies<br/>• Traffic inspection]:::firewall
    VPN[🔗 VPN/Direct Connect<br/>• Site-to-site tunnel<br/>• BGP routing<br/>• 10Gbps bandwidth]:::vpn
    TGW[🌐 Transit Gateway<br/>• Multi-VPC routing<br/>• On-premises attachment<br/>• Route propagation]:::tgw
    MWAAVPC[🏠 MWAA Service VPC<br/>• Private subnets<br/>• 10.1.0.0/16 CIDR<br/>• No internet gateway]:::mwaavpc
    WEBSERVER[🌐 MWAA Web Server<br/>• REST API endpoints<br/>• PRIVATE_ONLY mode<br/>• IAM authentication]:::webserver
    DNS[🌐 Route 53 Private Zone<br/>• mwaa.company.internal<br/>• Cross-VPC resolution<br/>• A record: 10.1.10.100]:::dns
    IAM[👤 AWS IAM<br/>• SigV4 authentication<br/>• Role-based access<br/>• Temporary credentials]:::iam
    
    GITLAB -->|HTTPS Request| FIREWALL
    FIREWALL -->|Secure Tunnel| VPN
    VPN -->|BGP Routes| TGW
    TGW -->|Private Routing| MWAAVPC
    MWAAVPC -->|Internal Access| WEBSERVER
    
    DNS -.->|DNS Resolution| WEBSERVER
    GITLAB -.->|DNS Query| DNS
    IAM -.->|Authentication| WEBSERVER
    GITLAB -.->|Assume Role| IAM
    
    classDef gitlab fill:#FF6B35,stroke:#FF4500,stroke-width:4px,color:#fff
    classDef firewall fill:#DC143C,stroke:#B22222,stroke-width:4px,color:#fff
    classDef vpn fill:#4ECDC4,stroke:#20B2AA,stroke-width:4px,color:#fff
    classDef tgw fill:#45B7D1,stroke:#1E90FF,stroke-width:4px,color:#fff
    classDef mwaavpc fill:#96CEB4,stroke:#32CD32,stroke-width:4px,color:#fff
    classDef webserver fill:#FF9FF3,stroke:#FF69B4,stroke-width:4px,color:#fff
    classDef dns fill:#54A0FF,stroke:#4169E1,stroke-width:4px,color:#fff
    classDef iam fill:#FFD700,stroke:#FFA500,stroke-width:4px,color:#000
```

### 🔄 **API Call Sequence**

```mermaid
sequenceDiagram
    participant GL as 🦊 Gitlab Runner
    participant FW as 🔥 Firewall
    participant VPN as 🔗 VPN Gateway
    participant TGW as 🌐 Transit Gateway
    participant DNS as 🌐 Route 53 DNS
    participant IAM as 👤 AWS IAM
    participant WS as 🌐 MWAA Web Server
    
    Note over GL,WS: 🔐 Phase 1: Authentication Setup
    
    GL->>IAM: 1. Assume IAM Role
    Note right of GL: Role ARN:<br/>arn:aws:iam::123456789012:role/GitlabMWAARole<br/>External ID: gitlab-ci-token
    
    IAM-->>GL: 2. Return STS Credentials
    Note right of IAM: Temporary Credentials:<br/>• AccessKeyId: ASIA...<br/>• SecretAccessKey: temp-key<br/>• SessionToken: session-token<br/>• Duration: 3600 seconds
    
    Note over GL,WS: 🌐 Phase 2: DNS Resolution
    
    GL->>DNS: 3. Resolve MWAA Endpoint
    Note right of GL: DNS Query:<br/>• Host: mwaa-prod.company.internal<br/>• Type: A record<br/>• Resolver: 192.168.1.10
    
    DNS-->>GL: 4. Return Private IP
    Note right of DNS: DNS Response:<br/>• IP: 10.1.10.100<br/>• TTL: 300 seconds<br/>• Zone: company.internal
    
    Note over GL,WS: 🚀 Phase 3: API Request Flow
    
    GL->>FW: 5. HTTPS POST Request
    Note right of GL: API Request:<br/>• URL: https://10.1.10.100/api/v1/dags/etl_pipeline/dagRuns<br/>• Method: POST<br/>• Headers: Authorization (SigV4), Content-Type<br/>• Body: DAG configuration JSON
    
    FW->>VPN: 6. Route to AWS
    Note right of FW: Firewall Rules:<br/>• Allow HTTPS (443) to 10.1.0.0/16<br/>• Source: Gitlab subnet (192.168.100.0/24)<br/>• Destination: MWAA VPC<br/>• Action: ALLOW
    
    VPN->>TGW: 7. Forward via Tunnel
    Note right of VPN: VPN Routing:<br/>• Tunnel Status: UP<br/>• BGP State: Established<br/>• Advertised Routes: 10.1.0.0/16<br/>• Next Hop: Transit Gateway
    
    TGW->>WS: 8. Direct to MWAA Web Server
    Note right of TGW: TGW Routing:<br/>• Route Table: rtb-mwaa-prod<br/>• Target: 10.1.10.100/32<br/>• Attachment: vpc-mwaa-service<br/>• Status: Active
    
    Note over GL,WS: 🔍 Phase 4: API Processing
    
    WS->>IAM: 9. Validate SigV4 Signature
    Note right of WS: Authentication Check:<br/>• Signature Algorithm: AWS4-HMAC-SHA256<br/>• Credential Scope: 20240115/us-east-1/airflow/aws4_request<br/>• Signed Headers: host and x-amz-date<br/>• Request Hash: SHA256
    
    IAM-->>WS: 10. Authorization Result
    Note right of IAM: Policy Evaluation:<br/>• Action: airflow:CreateDagRun<br/>• Resource: MWAA environment ARN<br/>• Principal: GitlabMWAARole assumed role<br/>• Result: ALLOW
    
    WS->>WS: 11. Process DAG Trigger
    Note right of WS: DAG Execution:<br/>• DAG ID: etl_pipeline<br/>• Run ID: manual__2024-01-15T14:30:00+00:00<br/>• State: running<br/>• Tasks: 5 queued
    
    WS-->>TGW: 12. Return Success Response
    Note right of WS: API Response:<br/>• Status: 201 Created<br/>• Body: DAG run details<br/>• Headers: Content-Type: application/json<br/>• Execution Time: 245ms
    
    TGW-->>VPN: 13. Route Response Back
    VPN-->>FW: 14. Return via Tunnel
    FW-->>GL: 15. Deliver to Gitlab
    
    Note right of GL: Pipeline Success:<br/>• DAG triggered successfully<br/>• Run ID captured for monitoring<br/>• Next stage: Wait for completion<br/>• Logs: Available in MWAA UI
```

**2. EC2 Instance-A (VPC-A, Same Account) → MWAA REST API**

### 🏗️ **Same Account VPC Architecture**

```mermaid
graph TB
    EC2A["💻 EC2 Instance-A<br/>• Application server<br/>• VPC-A 10.2.0.0/16<br/>• Private subnet"]:::ec2
    VPCA["🏠 VPC-A<br/>• Same AWS account<br/>• 10.2.0.0/16 CIDR<br/>• Private subnets only"]:::vpca
    PEERING["🔗 VPC Peering<br/>• pcx-mwaa-vpc-a<br/>• Cross-VPC routing<br/>• Same account"]:::peering
    MWAAVPC["🏠 MWAA Service VPC<br/>• 10.1.0.0/16 CIDR<br/>• PRIVATE_ONLY mode<br/>• Managed by AWS"]:::mwaavpc
    WEBSERVER["🌐 MWAA Web Server<br/>• REST API endpoints<br/>• 10.1.10.100<br/>• IAM authentication"]:::webserver
    ROUTE53["🌐 Route 53 Resolver<br/>• Cross-VPC DNS<br/>• Private hosted zone<br/>• Automatic resolution"]:::dns
    IAM["👤 AWS IAM<br/>• Instance profile<br/>• Cross-service access<br/>• Same account trust"]:::iam
    
    EC2A -->|API Calls| VPCA
    VPCA -->|Peering Connection| PEERING
    PEERING -->|Direct Routing| MWAAVPC
    MWAAVPC -->|Internal Access| WEBSERVER
    
    ROUTE53 -.->|DNS Resolution| WEBSERVER
    EC2A -.->|DNS Query| ROUTE53
    IAM -.->|Instance Profile| EC2A
    IAM -.->|Authorization| WEBSERVER
    
    classDef ec2 fill:#FF6B35,stroke:#FF4500,stroke-width:4px,color:#fff
    classDef vpca fill:#32CD32,stroke:#228B22,stroke-width:4px,color:#fff
    classDef peering fill:#4ECDC4,stroke:#20B2AA,stroke-width:4px,color:#fff
    classDef mwaavpc fill:#96CEB4,stroke:#32CD32,stroke-width:4px,color:#fff
    classDef webserver fill:#FF9FF3,stroke:#FF69B4,stroke-width:4px,color:#fff
    classDef dns fill:#54A0FF,stroke:#4169E1,stroke-width:4px,color:#fff
    classDef iam fill:#FFD700,stroke:#FFA500,stroke-width:4px,color:#000
```

### 🔄 **Same Account API Sequence**

```mermaid
sequenceDiagram
    participant EC2 as 💻 EC2 Instance-A
    participant VPCA as 🏠 VPC-A
    participant PCX as 🔗 VPC Peering
    participant MWAAVPC as 🏠 MWAA VPC
    participant DNS as 🌐 Route 53
    participant IAM as 👤 AWS IAM
    participant WS as 🌐 MWAA Web Server
    
    Note over EC2,WS: 🔐 Phase 1: Instance Profile Authentication
    
    EC2->>IAM: 1. Retrieve Instance Profile Credentials
    Note right of EC2: Instance Metadata:<br/>• Role: EC2-MWAA-Access-Role<br/>• URL: http://169.254.169.254/latest/meta-data/iam/security-credentials/<br/>• Automatic credential rotation
    
    IAM-->>EC2: 2. Return Instance Credentials
    Note right of IAM: Instance Profile Response:<br/>• AccessKeyId: ASIA...<br/>• SecretAccessKey: instance-key<br/>• Token: instance-token<br/>• Expiration: Auto-renewed
    
    Note over EC2,WS: 🌐 Phase 2: VPC Peering Resolution
    
    EC2->>DNS: 3. Resolve MWAA Endpoint
    Note right of EC2: DNS Query:<br/>• Host: mwaa-prod.internal<br/>• Resolver: VPC DNS (10.2.0.2)<br/>• Query Type: A record
    
    DNS-->>EC2: 4. Return MWAA IP Address
    Note right of DNS: DNS Response:<br/>• IP: 10.1.10.100 (MWAA VPC)<br/>• TTL: 300 seconds<br/>• Resolved via peering
    
    Note over EC2,WS: 🚀 Phase 3: Cross-VPC API Call
    
    EC2->>VPCA: 5. Initiate HTTPS Request
    Note right of EC2: API Request:<br/>• URL: https://10.1.10.100/api/v1/dags/data_processing/dagRuns<br/>• Method: POST<br/>• Headers: Authorization (SigV4)<br/>• Source IP: 10.2.1.100
    
    VPCA->>PCX: 6. Route via VPC Peering
    Note right of VPCA: VPC-A Route Table:<br/>• Destination: 10.1.0.0/16<br/>• Target: pcx-mwaa-vpc-a<br/>• Status: Active<br/>• Local preference: 100
    
    PCX->>MWAAVPC: 7. Forward to MWAA VPC
    Note right of PCX: Peering Connection:<br/>• Connection ID: pcx-mwaa-vpc-a<br/>• Status: Active<br/>• Accepter VPC: vpc-mwaa-service<br/>• Requester VPC: vpc-a-12345
    
    MWAAVPC->>WS: 8. Deliver to Web Server
    Note right of MWAAVPC: MWAA VPC Security:<br/>• Security Group: sg-mwaa-webserver<br/>• Inbound Rule: 443 from 10.2.0.0/16<br/>• Source: VPC-A instances<br/>• Action: ALLOW
    
    Note over EC2,WS: 🔍 Phase 4: Same Account Processing
    
    WS->>IAM: 9. Validate Instance Profile
    Note right of WS: Authentication Validation:<br/>• Principal: arn:aws:sts::123456789012:assumed-role/EC2-MWAA-Access-Role/i-0123456789abcdef0<br/>• Signature: AWS4-HMAC-SHA256<br/>• Same account trust: Simplified
    
    IAM-->>WS: 10. Authorization Success
    Note right of IAM: Policy Check:<br/>• Action: airflow:CreateDagRun<br/>• Resource: arn:aws:airflow:us-east-1:123456789012:environment/mwaa-prod<br/>• Same account: Direct access<br/>• Result: ALLOW
    
    WS->>WS: 11. Execute DAG Trigger
    Note right of WS: DAG Processing:<br/>• DAG ID: data_processing<br/>• Trigger Type: API<br/>• Run ID: api__2024-01-15T14:45:00+00:00<br/>• Queue: default
    
    WS-->>MWAAVPC: 12. Return API Response
    Note right of WS: Success Response:<br/>• Status: 201 Created<br/>• Response Time: 180ms<br/>• DAG Run Created<br/>• Same account efficiency
    
    MWAAVPC-->>PCX: 13. Route Back via Peering
    PCX-->>VPCA: 14. Return to VPC-A
    VPCA-->>EC2: 15. Deliver Response
    
    Note right of EC2: Application Success:<br/>• DAG triggered from EC2<br/>• Same account simplicity<br/>• Low latency via peering<br/>• Direct VPC connectivity
```

**3. EC2 Instance-B (VPC-B, Same Account) → MWAA REST API**

### 🏗️ **Transit Gateway Architecture**

```mermaid
graph TB
    EC2B[💻 EC2 Instance-B<br/>• Batch processing<br/>• VPC-B (10.3.0.0/16)<br/>• Private subnet]:::ec2
    VPCB[🏠 VPC-B<br/>• Same AWS account<br/>• 10.3.0.0/16 CIDR<br/>• TGW attachment]:::vpcb
    TGW[🌐 Transit Gateway<br/>• tgw-12345<br/>• Multi-VPC routing<br/>• Route propagation]:::tgw
    MWAAVPC[🏠 MWAA Service VPC<br/>• 10.1.0.0/16 CIDR<br/>• TGW attachment<br/>• Managed service]:::mwaavpc
    WEBSERVER[🌐 MWAA Web Server<br/>• REST API endpoints<br/>• 10.1.10.100<br/>• Cross-VPC access]:::webserver
    ROUTE53[🌐 Route 53 Resolver<br/>• TGW DNS forwarding<br/>• Cross-VPC resolution<br/>• Centralized DNS]:::dns
    IAM[👤 AWS IAM<br/>• Instance profile<br/>• Same account access<br/>• Simplified trust]:::iam
    
    EC2B -->|API Requests| VPCB
    VPCB -->|TGW Attachment| TGW
    TGW -->|Route Propagation| MWAAVPC
    MWAAVPC -->|Internal Access| WEBSERVER
    
    ROUTE53 -.->|DNS Resolution| WEBSERVER
    EC2B -.->|DNS Query| ROUTE53
    IAM -.->|Instance Profile| EC2B
    IAM -.->|Authorization| WEBSERVER
    
    classDef ec2 fill:#FF6B35,stroke:#FF4500,stroke-width:4px,color:#fff
    classDef vpcb fill:#9B59B6,stroke:#8E44AD,stroke-width:4px,color:#fff
    classDef tgw fill:#45B7D1,stroke:#1E90FF,stroke-width:4px,color:#fff
    classDef mwaavpc fill:#96CEB4,stroke:#32CD32,stroke-width:4px,color:#fff
    classDef webserver fill:#FF9FF3,stroke:#FF69B4,stroke-width:4px,color:#fff
    classDef dns fill:#54A0FF,stroke:#4169E1,stroke-width:4px,color:#fff
    classDef iam fill:#FFD700,stroke:#FFA500,stroke-width:4px,color:#000
```

### 🔄 **Transit Gateway API Sequence**

```mermaid
sequenceDiagram
    participant EC2 as 💻 EC2 Instance-B
    participant VPCB as 🏠 VPC-B
    participant TGW as 🌐 Transit Gateway
    participant MWAAVPC as 🏠 MWAA VPC
    participant DNS as 🌐 Route 53
    participant IAM as 👤 AWS IAM
    participant WS as 🌐 MWAA Web Server
    
    Note over EC2,WS: 🔐 Phase 1: Instance Authentication
    
    EC2->>IAM: 1. Get Instance Profile Credentials
    Note right of EC2: Metadata Service:<br/>• Role: EC2-BatchProcessor-Role<br/>• Instance ID: i-0987654321fedcba<br/>• VPC: vpc-b-67890<br/>• Subnet: subnet-private-b1
    
    IAM-->>EC2: 2. Return Temporary Credentials
    Note right of IAM: Instance Credentials:<br/>• AccessKeyId: ASIA...<br/>• SecretAccessKey: batch-key<br/>• Token: batch-token<br/>• Auto-rotation: Enabled
    
    Note over EC2,WS: 🌐 Phase 2: TGW DNS Resolution
    
    EC2->>DNS: 3. Resolve MWAA via TGW
    Note right of EC2: DNS Query:<br/>• Host: mwaa-prod.aws.internal<br/>• Resolver: VPC-B DNS (10.3.0.2)<br/>• TGW DNS forwarding enabled
    
    DNS-->>EC2: 4. Return MWAA Private IP
    Note right of DNS: TGW DNS Response:<br/>• IP: 10.1.10.100<br/>• Resolved via TGW routing<br/>• Cross-VPC DNS working<br/>• TTL: 300 seconds
    
    Note over EC2,WS: 🚀 Phase 3: TGW Routing Flow
    
    EC2->>VPCB: 5. Initiate Batch API Call
    Note right of EC2: Batch API Request:<br/>• URL: https://10.1.10.100/api/v1/dags/batch_etl/dagRuns<br/>• Method: POST<br/>• Payload: Batch job parameters<br/>• Source: 10.3.1.50
    
    VPCB->>TGW: 6. Route via Transit Gateway
    Note right of VPCB: VPC-B Route Table:<br/>• Destination: 10.1.0.0/16<br/>• Target: tgw-12345<br/>• Attachment: tgw-attach-vpc-b<br/>• Propagation: Enabled
    
    TGW->>MWAAVPC: 7. Forward to MWAA VPC
    Note right of TGW: TGW Route Table:<br/>• Route: 10.1.0.0/16 → vpc-mwaa-service<br/>• Association: tgw-rtb-main<br/>• Propagation: Active<br/>• Next hop: MWAA VPC attachment
    
    MWAAVPC->>WS: 8. Deliver to Web Server
    Note right of MWAAVPC: MWAA Security Group:<br/>• Rule: Allow 443 from 10.3.0.0/16<br/>• Source: VPC-B CIDR<br/>• Protocol: HTTPS<br/>• Action: ALLOW
    
    Note over EC2,WS: 🔍 Phase 4: Batch Processing Authorization
    
    WS->>IAM: 9. Validate Batch Processor Role
    Note right of WS: Role Validation:<br/>• Principal: arn:aws:sts::123456789012:assumed-role/EC2-BatchProcessor-Role/i-0987654321fedcba<br/>• Action: airflow:CreateDagRun<br/>• Same account trust
    
    IAM-->>WS: 10. Authorize Batch Operation
    Note right of IAM: Batch Authorization:<br/>• Policy: BatchProcessorMWAAAccess<br/>• Resource: arn:aws:airflow:us-east-1:123456789012:environment/mwaa-prod<br/>• Condition: VPC source check<br/>• Result: ALLOW
    
    WS->>WS: 11. Trigger Batch DAG
    Note right of WS: Batch DAG Execution:<br/>• DAG ID: batch_etl<br/>• Run Type: API triggered<br/>• Run ID: batch__2024-01-15T15:00:00+00:00<br/>• Worker queue: batch_queue
    
    WS-->>MWAAVPC: 12. Return Batch Response
    Note right of WS: Batch API Response:<br/>• Status: 201 Created<br/>• Batch job queued<br/>• Estimated duration: 45 minutes<br/>• Response time: 220ms
    
    MWAAVPC-->>TGW: 13. Route Back via TGW
    TGW-->>VPCB: 14. Return to VPC-B
    VPCB-->>EC2: 15. Deliver to Batch Instance
    
    Note right of EC2: Batch Job Success:<br/>• DAG triggered for batch processing<br/>• TGW routing efficient<br/>• Same account simplicity<br/>• Scalable architecture
```

**4. EC2 Instance-C (VPC-C, Different Account) → MWAA REST API**

### 🏗️ **Cross-Account Architecture**

```mermaid
graph TB
    ACCOUNTC[🏢 AWS Account C<br/>• Account ID: 987654321098<br/>• Cross-account access<br/>• Shared TGW attachment]:::accountc
    EC2C[💻 EC2 Instance-C<br/>• External application<br/>• VPC-C (10.4.0.0/16)<br/>• Cross-account role]:::ec2
    VPCC[🏠 VPC-C<br/>• Different AWS account<br/>• 10.4.0.0/16 CIDR<br/>• Shared TGW access]:::vpcc
    RAM[🤝 Resource Access Manager<br/>• Cross-account TGW sharing<br/>• Resource share invitation<br/>• Trust relationship]:::ram
    TGW[🌐 Transit Gateway<br/>• Shared resource<br/>• Cross-account routing<br/>• Account A owned]:::tgw
    ACCOUNTA[🏢 AWS Account A<br/>• Account ID: 123456789012<br/>• MWAA environment<br/>• TGW owner]:::accounta
    MWAAVPC[🏠 MWAA Service VPC<br/>• 10.1.0.0/16 CIDR<br/>• Account A managed<br/>• Cross-account access]:::mwaavpc
    WEBSERVER[🌐 MWAA Web Server<br/>• REST API endpoints<br/>• Cross-account auth<br/>• 10.1.10.100]:::webserver
    IAM[👤 Cross-Account IAM<br/>• AssumeRole trust<br/>• External ID validation<br/>• Cross-account policy]:::iam
    
    ACCOUNTC --> EC2C
    EC2C --> VPCC
    VPCC --> RAM
    RAM --> TGW
    TGW --> ACCOUNTA
    ACCOUNTA --> MWAAVPC
    MWAAVPC --> WEBSERVER
    
    IAM -.->|Cross-Account Trust| EC2C
    IAM -.->|Authorization| WEBSERVER
    
    classDef accountc fill:#E74C3C,stroke:#C0392B,stroke-width:4px,color:#fff
    classDef ec2 fill:#FF6B35,stroke:#FF4500,stroke-width:4px,color:#fff
    classDef vpcc fill:#8E44AD,stroke:#7D3C98,stroke-width:4px,color:#fff
    classDef ram fill:#F39C12,stroke:#E67E22,stroke-width:4px,color:#fff
    classDef tgw fill:#45B7D1,stroke:#1E90FF,stroke-width:4px,color:#fff
    classDef accounta fill:#27AE60,stroke:#229954,stroke-width:4px,color:#fff
    classDef mwaavpc fill:#96CEB4,stroke:#32CD32,stroke-width:4px,color:#fff
    classDef webserver fill:#FF9FF3,stroke:#FF69B4,stroke-width:4px,color:#fff
    classDef iam fill:#FFD700,stroke:#FFA500,stroke-width:4px,color:#000
```

### 🔄 **Cross-Account API Sequence**

```mermaid
sequenceDiagram
    participant EC2 as 💻 EC2 Instance-C
    participant VPCC as 🏠 VPC-C (Account C)
    participant RAM as 🤝 Resource Manager
    participant TGW as 🌐 Transit Gateway
    participant MWAAVPC as 🏠 MWAA VPC (Account A)
    participant IAMA as 👤 IAM Account A
    participant IAMC as 👤 IAM Account C
    participant WS as 🌐 MWAA Web Server
    
    Note over EC2,WS: 🔐 Phase 1: Cross-Account Authentication
    
    EC2->>IAMC: 1. Get Instance Profile (Account C)
    Note right of EC2: Account C Credentials:<br/>• Role: EC2-CrossAccount-Role<br/>• Account: 987654321098<br/>• Instance: i-0abcdef123456789<br/>• VPC: vpc-c-external
    
    IAMC-->>EC2: 2. Return Account C Credentials
    Note right of IAMC: Local Credentials:<br/>• AccessKeyId: ASIA...<br/>• SecretAccessKey: account-c-key<br/>• Token: account-c-token<br/>• Account: 987654321098
    
    EC2->>IAMA: 3. AssumeRole Cross-Account
    Note right of EC2: Cross-Account AssumeRole:<br/>• Target Role: arn:aws:iam::123456789012:role/CrossAccountMWAAAccess<br/>• External ID: external-app-12345<br/>• Session Name: cross-account-mwaa-session<br/>• Duration: 3600 seconds
    
    IAMA-->>EC2: 4. Return Cross-Account Credentials
    Note right of IAMA: Cross-Account STS:<br/>• AccessKeyId: ASIA... (Account A)<br/>• SecretAccessKey: cross-account-key<br/>• SessionToken: cross-account-token<br/>• AssumedRoleUser: arn:aws:sts::123456789012:assumed-role/CrossAccountMWAAAccess/cross-account-mwaa-session
    
    Note over EC2,WS: 🌐 Phase 2: Cross-Account Network Routing
    
    EC2->>VPCC: 5. Initiate Cross-Account API Call
    Note right of EC2: Cross-Account Request:<br/>• URL: https://10.1.10.100/api/v1/dags/external_integration/dagRuns<br/>• Method: POST<br/>• Headers: Authorization (Account A credentials)<br/>• Source: 10.4.1.75 (Account C)
    
    VPCC->>RAM: 6. Access Shared TGW
    Note right of VPCC: Resource Share Access:<br/>• Shared Resource: tgw-12345<br/>• Resource Share: MWAA-TGW-Share<br/>• Owner Account: 123456789012<br/>• Consumer Account: 987654321098
    
    RAM->>TGW: 7. Route via Shared TGW
    Note right of RAM: Cross-Account Routing:<br/>• TGW Owner: Account A (123456789012)<br/>• TGW Consumer: Account C (987654321098)<br/>• Route Table: Cross-account routes enabled<br/>• Destination: 10.1.0.0/16 (MWAA VPC)
    
    TGW->>MWAAVPC: 8. Forward to MWAA VPC
    Note right of TGW: TGW Cross-Account Route:<br/>• Source Account: 987654321098<br/>• Source VPC: vpc-c-external<br/>• Target Account: 123456789012<br/>• Target VPC: vpc-mwaa-service
    
    MWAAVPC->>WS: 9. Deliver to Web Server
    Note right of MWAAVPC: Cross-Account Security:<br/>• Security Group: Allow 443 from 10.4.0.0/16<br/>• Source: Account C VPC CIDR<br/>• Cross-account trust required<br/>• Action: ALLOW
    
    Note over EC2,WS: 🔍 Phase 3: Cross-Account Authorization
    
    WS->>IAMA: 10. Validate Cross-Account Role
    Note right of WS: Cross-Account Validation:<br/>• Principal: arn:aws:sts::123456789012:assumed-role/CrossAccountMWAAAccess/cross-account-mwaa-session<br/>• Original Account: 987654321098<br/>• External ID: external-app-12345<br/>• Trust Policy: Verified
    
    IAMA-->>WS: 11. Authorize Cross-Account Access
    Note right of IAMA: Cross-Account Policy:<br/>• Action: airflow:CreateDagRun<br/>• Resource: arn:aws:airflow:us-east-1:123456789012:environment/mwaa-prod<br/>• Condition: External ID match<br/>• Principal: Account C assumed role<br/>• Result: ALLOW
    
    WS->>WS: 12. Process External Integration DAG
    Note right of WS: External DAG Execution:<br/>• DAG ID: external_integration<br/>• Trigger: Cross-account API<br/>• Run ID: external__2024-01-15T15:15:00+00:00<br/>• Source Account: 987654321098
    
    WS-->>MWAAVPC: 13. Return Cross-Account Response
    Note right of WS: Cross-Account Response:<br/>• Status: 201 Created<br/>• Cross-account DAG triggered<br/>• External integration successful<br/>• Response time: 350ms
    
    MWAAVPC-->>TGW: 14. Route Back via Shared TGW
    TGW-->>RAM: 15. Return via Resource Share
    RAM-->>VPCC: 16. Deliver to Account C VPC
    VPCC-->>EC2: 17. Return to External Instance
    
    Note right of EC2: Cross-Account Success:<br/>• External system integrated<br/>• Cross-account trust working<br/>• Shared TGW routing efficient<br/>• Secure cross-account access
```

#### 🔐 **Authentication & Security Requirements**

### 🛡️ **IAM Configuration Overview**

```mermaid
graph TB
    ONPREM[🏢 On-Premises<br/>• Gitlab CI/CD Role<br/>• External ID validation<br/>• SigV4 authentication]:::onprem
    EC2A[💻 EC2 Instance-A<br/>• Instance profile<br/>• Same account access<br/>• Simplified trust]:::ec2a
    EC2B[💻 EC2 Instance-B<br/>• Instance profile<br/>• TGW routing<br/>• Same account trust]:::ec2b
    EC2C[💻 EC2 Instance-C<br/>• Cross-account role<br/>• AssumeRole required<br/>• External ID validation]:::ec2c
    
    IAMROLES[👤 IAM Roles & Policies<br/>• GitlabMWAARole<br/>• EC2-MWAA-Access-Role<br/>• CrossAccountMWAAAccess<br/>• BatchProcessorRole]:::iam
    
    MWAAPOLICY[📋 MWAA Permissions<br/>• airflow:CreateDagRun<br/>• airflow:GetDag<br/>• airflow:GetDagRuns<br/>• airflow:CreateWebLoginToken]:::policy
    
    SECGROUPS[🛡️ Security Groups<br/>• sg-mwaa-webserver<br/>• Allow 443 from sources<br/>• Cross-VPC access rules<br/>• Account-specific rules]:::security
    
    ONPREM --> IAMROLES
    EC2A --> IAMROLES
    EC2B --> IAMROLES
    EC2C --> IAMROLES
    
    IAMROLES --> MWAAPOLICY
    IAMROLES --> SECGROUPS
    
    classDef onprem fill:#FF6B35,stroke:#FF4500,stroke-width:3px,color:#fff
    classDef ec2a fill:#32CD32,stroke:#228B22,stroke-width:3px,color:#fff
    classDef ec2b fill:#9B59B6,stroke:#8E44AD,stroke-width:3px,color:#fff
    classDef ec2c fill:#E74C3C,stroke:#C0392B,stroke-width:3px,color:#fff
    classDef iam fill:#FFD700,stroke:#FFA500,stroke-width:3px,color:#000
    classDef policy fill:#54A0FF,stroke:#4169E1,stroke-width:3px,color:#fff
    classDef security fill:#FF9FF3,stroke:#FF69B4,stroke-width:3px,color:#fff
```

#### 🚀 **Alternative: API Gateway Integration**

### 🌐 **API Gateway Proxy Architecture**

```mermaid
graph TB
    CLIENTS[🔗 API Clients<br/>• On-premises apps<br/>• EC2 instances<br/>• External systems<br/>• Unified access point]:::clients
    
    APIGW[🚪 API Gateway<br/>• Private REST API<br/>• Custom authentication<br/>• Rate limiting<br/>• Request/response transformation]:::apigw
    
    VPCENDPOINT[🔌 VPC Endpoint<br/>• Interface endpoint<br/>• Private connectivity<br/>• DNS resolution<br/>• Security group control]:::endpoint
    
    LAMBDA[⚡ Lambda Authorizer<br/>• Custom authentication<br/>• Token validation<br/>• Fine-grained access<br/>• Audit logging]:::lambda
    
    MWAAPROXY[🔄 MWAA Proxy<br/>• Request forwarding<br/>• Response handling<br/>• Error management<br/>• Logging integration]:::proxy
    
    MWAAWEBSERVER[🌐 MWAA Web Server<br/>• REST API endpoints<br/>• PRIVATE_ONLY mode<br/>• Internal access only<br/>• IAM authentication]:::mwaa
    
    CLIENTS --> APIGW
    APIGW --> VPCENDPOINT
    APIGW --> LAMBDA
    LAMBDA --> APIGW
    VPCENDPOINT --> MWAAPROXY
    MWAAPROXY --> MWAAWEBSERVER
    
    classDef clients fill:#FF6B35,stroke:#FF4500,stroke-width:3px,color:#fff
    classDef apigw fill:#96CEB4,stroke:#32CD32,stroke-width:3px,color:#fff
    classDef endpoint fill:#4ECDC4,stroke:#20B2AA,stroke-width:3px,color:#fff
    classDef lambda fill:#FF9F43,stroke:#E67E22,stroke-width:3px,color:#fff
    classDef proxy fill:#45B7D1,stroke:#1E90FF,stroke-width:3px,color:#fff
    classDef mwaa fill:#FF9FF3,stroke:#FF69B4,stroke-width:3px,color:#fff
```

#### 📋 **Implementation Roadmap**

### 🗺️ **Deployment Strategy Overview**

```mermaid
graph TB
    PHASE1[🔧 Phase 1: Network Foundation<br/>• VPN/Direct Connect setup<br/>• Transit Gateway configuration<br/>• VPC peering (if needed)<br/>• DNS resolution setup]:::phase1
    
    PHASE2[🛡️ Phase 2: Security Configuration<br/>• IAM roles and policies<br/>• Security group rules<br/>• Cross-account trust setup<br/>• External ID validation]:::phase2
    
    PHASE3[🧪 Phase 3: Testing & Validation<br/>• Network connectivity tests<br/>• API authentication tests<br/>• DAG trigger validation<br/>• Performance benchmarking]:::phase3
    
    PHASE4[📊 Phase 4: Monitoring & Operations<br/>• CloudWatch logging<br/>• VPC Flow Logs<br/>• CloudTrail API logging<br/>• Alerting configuration]:::phase4
    
    PHASE5[🚀 Phase 5: Production Migration<br/>• Gitlab CI/CD integration<br/>• Application deployment<br/>• Load testing<br/>• Documentation handover]:::phase5
    
    PHASE1 --> PHASE2
    PHASE2 --> PHASE3
    PHASE3 --> PHASE4
    PHASE4 --> PHASE5
    
    classDef phase1 fill:#FF6B35,stroke:#FF4500,stroke-width:3px,color:#fff
    classDef phase2 fill:#FFD700,stroke:#FFA500,stroke-width:3px,color:#000
    classDef phase3 fill:#32CD32,stroke:#228B22,stroke-width:3px,color:#fff
    classDef phase4 fill:#45B7D1,stroke:#1E90FF,stroke-width:3px,color:#fff
    classDef phase5 fill:#9B59B6,stroke:#8E44AD,stroke-width:3px,color:#fff
```

#### 🎯 **Success Criteria & Validation**

### ✅ **Validation Matrix**

```mermaid
graph TB
    NETWORK[🌐 Network Validation<br/>• Ping tests to MWAA VPC<br/>• DNS resolution working<br/>• Route table verification<br/>• Security group testing]:::network
    
    AUTH[🔐 Authentication Validation<br/>• SigV4 signature working<br/>• IAM role assumption<br/>• Cross-account trust<br/>• Token expiration handling]:::auth
    
    API[🔌 API Functionality<br/>• DAG listing successful<br/>• DAG triggering working<br/>• Status monitoring active<br/>• Error handling proper]:::api
    
    PERF[⚡ Performance Validation<br/>• Response time < 500ms<br/>• Concurrent request handling<br/>• Network latency acceptable<br/>• Throughput requirements met]:::perf
    
    MONITOR[📊 Monitoring Validation<br/>• CloudWatch logs flowing<br/>• Metrics collection active<br/>• Alerting rules working<br/>• Dashboard visibility]:::monitor
    
    NETWORK --> AUTH
    AUTH --> API
    API --> PERF
    PERF --> MONITOR
    
    classDef network fill:#4ECDC4,stroke:#20B2AA,stroke-width:3px,color:#fff
    classDef auth fill:#FFD700,stroke:#FFA500,stroke-width:3px,color:#000
    classDef api fill:#96CEB4,stroke:#32CD32,stroke-width:3px,color:#fff
    classDef perf fill:#FF9F43,stroke:#E67E22,stroke-width:3px,color:#fff
    classDef monitor fill:#45B7D1,stroke:#1E90FF,stroke-width:3px,color:#fff
```

**Key Benefits:**
- **Seamless Migration:** Existing Gitlab CI/CD workflows migrate with minimal changes
- **Security Maintained:** PRIVATE_ONLY mode preserved with proper network access
- **Scalable Architecture:** Supports multiple access patterns and future growth
- **Operational Excellence:** Comprehensive monitoring and troubleshooting capabilities

This architecture enables secure, scalable REST API access to MWAA from various network environments while maintaining the private security posture.
    
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
```nation: 10.1.0.0/16 (MWAA VPC)<br/>• Next hop: VPN Gateway<br/>• Protocol: HTTPS (443)<br/>• Source: On-premises subnet
    
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
