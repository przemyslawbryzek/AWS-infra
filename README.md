# AWS-infra
```mermaid
flowchart TB
    User(["Użytkownik / Internet"])
    Admin(["Administrator"])
    Mail(["E-mail admina"])

    subgraph AWS["AWS Region"]
        WAF["AWS WAF"]
        ACM["ACM - certyfikat TLS"]
        KMS["KMS - klucze szyfrujące"]
        IGW["Internet Gateway"]

        subgraph VPC["VPC 10.0.0.0/16"]
            ALB["Application Load Balancer - HTTPS 443, health checks, SG-ALB"]

            subgraph AZA["Availability Zone A"]
                subgraph PUBA["Public subnet / DMZ 10.0.1.0/24"]
                    NATA["NAT Gateway"]
                end
                subgraph APPA["Private app subnet 10.0.11.0/24"]
                    EC2A["EC2 web 1 - Auto Scaling Group, SG-WEB, IAM Role, IMDSv2, szyfrowany EBS"]
                end
                subgraph DBA["Private DB subnet 10.0.21.0/24 - bez trasy do internetu"]
                    RDSP[("RDS PRIMARY - SG-DB, szyfrowanie KMS")]
                end
            end

            subgraph AZB["Availability Zone B"]
                subgraph PUBB["Public subnet / DMZ 10.0.2.0/24"]
                    NATB["NAT Gateway"]
                end
                subgraph APPB["Private app subnet 10.0.12.0/24"]
                    EC2B["EC2 web 2 - Auto Scaling Group, SG-WEB, IAM Role, IMDSv2, szyfrowany EBS"]
                end
                subgraph DBB["Private DB subnet 10.0.22.0/24 - bez trasy do internetu"]
                    RDSS[("RDS STANDBY - Multi-AZ")]
                end
            end

            subgraph VPCE["VPC Endpoints - ruch prywatny"]
                EPSSM["SSM / SSMMessages / EC2Messages"]
                EPSEC["Secrets Manager"]
                EPS3["S3 - gateway"]
            end
        end

        SSM["Systems Manager Session Manager - IAM + MFA, brak SSH i bastionu"]
        SM["Secrets Manager - hasło DB z automatyczną rotacją"]

        subgraph DET["Warstwa detekcyjna i logowanie"]
            CT["CloudTrail"]
            GD["GuardDuty"]
            CFG["AWS Config"]
            SH["Security Hub"]
            INS["Inspector"]
            FL["VPC Flow Logs"]
            CW["CloudWatch Logs i Alarms"]
            EB["EventBridge"]
            SNS["SNS Topic"]
            S3L[("S3 - logi, szyfrowane KMS")]
        end
    end

    User -->|HTTPS| WAF --> IGW --> ALB
    ACM -.-> ALB
    ALB -->|HTTP z SG-ALB| EC2A
    ALB -->|HTTP z SG-ALB| EC2B
    EC2A -->|PostgreSQL TLS z SG-WEB| RDSP
    EC2B -->|PostgreSQL TLS z SG-WEB| RDSP
    RDSP <-->|replikacja synchroniczna| RDSS
    RDSP -.->|failover| EC2A

    EC2A -->|pobranie hasła| EPSEC
    EC2B -->|pobranie hasła| EPSEC
    EPSEC --> SM
    SM -.->|rotacja hasła| RDSP

    Admin -->|HTTPS + MFA| SSM
    SSM --> EPSSM
    EPSSM -.-> EC2A
    EPSSM -.-> EC2B

    EC2A -->|aktualizacje| NATA --> IGW
    EC2B -->|aktualizacje| NATB --> IGW

    KMS -.-> RDSP
    KMS -.-> EC2A
    KMS -.-> EC2B
    KMS -.-> SM
    KMS -.-> S3L

    CT --> S3L
    FL --> CW
    GD --> EB
    CFG --> EB
    SH --> EB
    INS --> EB
    CW --> EB
    EB --> SNS --> Mail
```

### Security Groups

**SG-ALB** (Application Load Balancer, public subnets)

| Kierunek | Protokół / port | Źródło / cel | Uwagi |
|---|---|---|---|
| Inbound | TCP 443 | 0.0.0.0/0 | HTTPS od użytkowników |
| Inbound | TCP 80 | 0.0.0.0/0 | HTTPS Redirect |
| Outbound | TCP 80 (port aplikacji) | SG-WEB | tylko do instancji web |

**SG-WEB** (EC2 w ASG, private app subnets)

| Kierunek | Protokół / port | Źródło / cel | Uwagi |
|---|---|---|---|
| Inbound | TCP 80 (lub 8080) | SG-ALB | wyłącznie ruch z load balancera |
| Outbound | TCP 5432 | SG-DB | dostęp do bazy |
| Outbound | TCP 443 | SG-VPCE | SSM, Secrets Manager przez VPC endpointy |
| Outbound | TCP 443 | 0.0.0.0/0 (przez NAT) | aktualizacje pakietów |

**SG-DB** (RDS primary + standby, private DB subnets)

| Kierunek | Protokół / port | Źródło / cel | Uwagi |
|---|---|---|---|
| Inbound | TCP 5432 | SG-WEB | jedyne źródło ruchu do bazy |
| Inbound | TCP 5432 | SG-ROTATION | rotacja haseł |
| Outbound | brak reguł | - | RDS nie inicjuje połączeń |

**SG-VPCE** (interface endpointy: ssm, ssmmessages, ec2messages, secretsmanager)

| Kierunek | Protokół / port | Źródło / cel | Uwagi |
|---|---|---|---|
| Inbound | TCP 443 | SG-WEB | tylko z instancji web |
| Inbound | TCP 443 | SG-ROTATION | rotacja haseł |
| Outbound | brak reguł | - | endpointy nie inicjują ruchu |

**SG-ROTATION** (Lambda rotująca hasło)

| Kierunek | Protokół / port | Źródło / cel |
|---|---|---|
| Inbound | brak | - |
| Outbound | TCP 5432 | SG-DB |
| Outbound | TCP 443 | SG-VPCE |


### Network ACL (druga warstwa, stateless)
**Public subnets (DMZ): 10.0.1.0/24, 10.0.2.0/24**

| Kierunek | Reguła | Protokół / port | Źródło / cel |
|---|---|---|---|
| Inbound | allow | TCP 443, 80 | 0.0.0.0/0 |
| Inbound | allow | TCP 1024-65535 | 0.0.0.0/0 (odpowiedzi z internetu dla NAT) |
| Outbound | allow | TCP 80 / 8080 | 10.0.11.0/24, 10.0.12.0/24 (ALB -> web) |
| Outbound | allow | TCP 443 | 0.0.0.0/0 (NAT -> internet) |
| Outbound | allow | TCP 1024-65535 | 0.0.0.0/0 (odpowiedzi do użytkowników) |
| Oba | deny | all | domyślnie |

**Private app subnets: 10.0.11.0/24, 10.0.12.0/24**

| Kierunek | Reguła | Protokół / port | Źródło / cel |
|---|---|---|---|
| Inbound | allow | TCP 80 / 8080 | 10.0.1.0/24, 10.0.2.0/24 (z ALB) |
| Inbound | allow | TCP 1024-65535 | 0.0.0.0/0 (odpowiedzi z NAT, endpointów, bazy) |
| Outbound | allow | TCP 3306 / 5432 | 10.0.21.0/24, 10.0.22.0/24 |
| Outbound | allow | TCP 443 | 0.0.0.0/0 (NAT i VPC endpointy) |
| Outbound | allow | TCP 1024-65535 | 10.0.1.0/24, 10.0.2.0/24 (odpowiedzi do ALB) |
| Oba | deny | all | domyślnie |

**Private DB subnets: 10.0.21.0/24, 10.0.22.0/24**

| Kierunek | Reguła | Protokół / port | Źródło / cel |
|---|---|---|---|
| Inbound | allow | TCP 3306 / 5432 | 10.0.11.0/24, 10.0.12.0/24 |
| Outbound | allow | TCP 1024-65535 | 10.0.11.0/24, 10.0.12.0/24 |
| Oba | deny | all | domyślnie |

### AWS WAF (Web ACL podpięty do ALB)

| Priorytet | Reguła | Akcja |
|---|---|---|
| 1 | AWSManagedRulesAmazonIpReputationList | Block |
| 2 | Rate-based rule: np. 2000 żądań / 5 min na IP | Block |
| 3 | AWSManagedRulesCommonRuleSet | Block |
| 4 | AWSManagedRulesKnownBadInputsRuleSet | Block |
| 5 | AWSManagedRulesSQLiRuleSet | Block |
| Domyślna | - | Allow |