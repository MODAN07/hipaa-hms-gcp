# hipaa-hms-gcp
HIPAA-Compliant Hospital Management System — GCP 🏥
> Cloud architecture for a HIPAA-compliant Hospital Management System deployed on Google Cloud Platform. Implements zero-trust access, end-to-end encryption, and comprehensive audit logging to meet all HIPAA Technical Safeguard requirements.
![GCP](https://img.shields.io/badge/Cloud-GCP-4285F4?style=flat&logo=googlecloud)
![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?style=flat&logo=terraform)
![HIPAA](https://img.shields.io/badge/Compliance-HIPAA-green?style=flat)
---
📌 Project Overview
This architecture demonstrates how to deploy a multi-tier Hospital Management System (HMS) on GCP while satisfying HIPAA's Administrative, Physical, and Technical Safeguards for Protected Health Information (PHI).
Scope: Patient records management, appointment scheduling, lab results, and billing — all handling PHI subject to HIPAA requirements.
---
🏗️ Architecture
```
                            Internet
                               │
                        Cloud Armor WAF
                        (DDoS + OWASP rules)
                               │
                    ┌──────────▼──────────┐
                    │  Cloud Load Balancer │
                    │   (HTTPS only)       │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │        VPC — Private Network               │
         │                                            │
    ┌────▼────────┐      ┌─────────────┐      ┌──────▼──────┐
    │  Web Tier   │      │  App Tier   │      │  Data Tier  │
    │  GKE Pods   │─────▶│  GKE Pods  │─────▶│  Cloud SQL  │
    │  (Frontend) │      │  (API/BL)  │      │  (PHI Data) │
    └─────────────┘      └─────────────┘      └─────────────┘
                               │                      │
                    ┌──────────┘              KMS Encryption
                    │                          at rest
              Identity-Aware
              Proxy (IAP)
              Zero-trust access
                    │
              Cloud Audit
              Logging
              (all PHI access logged)
```
---
🔐 HIPAA Technical Safeguards Implemented
Access Controls (§164.312(a))
Safeguard	Implementation
Unique user identification	Cloud Identity with MFA enforced
Automatic logoff	IAP session timeout — 30 minutes
Encryption/decryption	Cloud KMS — all PHI encrypted in transit and at rest
Zero-trust access	Identity-Aware Proxy — no VPN needed, identity-verified per request
Audit Controls (§164.312(b))
Safeguard	Implementation
Audit logs	Cloud Audit Logs — Admin Activity, Data Access, System Event
PHI access log	All Cloud SQL queries logged and retained 6 years
Log integrity	Logs exported to immutable GCS bucket (WORM policy)
SIEM integration	Log export to SIEM via Pub/Sub
Integrity Controls (§164.312(c))
Safeguard	Implementation
PHI integrity	Cloud SQL checksums + point-in-time recovery
Transmission integrity	TLS 1.2+ enforced on all endpoints
WAF	Cloud Armor — OWASP Top 10 rule set enabled
Transmission Security (§164.312(e))
Safeguard	Implementation
Encryption in transit	TLS 1.2+ minimum — TLS 1.0/1.1 disabled
Internal encryption	Private Service Connect — PHI never traverses public internet
mTLS	Service-to-service authentication within GKE via Istio
---
📦 GCP Services Used
Service	Purpose	HIPAA Role
GKE (Autopilot)	Container orchestration	Workload isolation
Cloud SQL (PostgreSQL)	PHI data storage	Encrypted at rest, PITR
Cloud KMS	Key management	Envelope encryption for PHI
Identity-Aware Proxy	Zero-trust access	Access control
Cloud Armor	WAF + DDoS	Perimeter defence
Cloud Audit Logs	Activity logging	Audit trail
Secret Manager	Secrets storage	Credential management
VPC Service Controls	Data perimeter	PHI exfiltration prevention
Cloud Load Balancing	HTTPS termination	TLS enforcement
---
🏗️ Key Terraform Resources
```hcl
# Cloud SQL with encryption and private IP only
resource "google_sql_database_instance" "hms_db" {
  name             = "hms-postgresql"
  database_version = "POSTGRES_15"

  settings {
    tier = "db-n1-standard-2"

    ip_configuration {
      ipv4_enabled    = false  # No public IP — private only
      private_network = google_compute_network.vpc.id
    }

    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      retained_backups               = 30
    }

    database_flags {
      name  = "log_connections"
      value = "on"
    }

    database_flags {
      name  = "log_disconnections"
      value = "on"
    }
  }
}

# Cloud KMS key for PHI encryption
resource "google_kms_crypto_key" "phi_key" {
  name            = "phi-encryption-key"
  key_ring        = google_kms_key_ring.hms.id
  rotation_period = "7776000s"  # 90-day key rotation

  lifecycle {
    prevent_destroy = true  # Never accidentally delete PHI encryption keys
  }
}
```
---
📋 Compliance Checklist
[x] PHI encrypted at rest (Cloud KMS)
[x] PHI encrypted in transit (TLS 1.2+)
[x] All PHI access logged (Cloud Audit Logs)
[x] Logs retained 6 years (GCS WORM bucket)
[x] MFA enforced for all users (Cloud Identity)
[x] Automatic session timeout (IAP — 30 min)
[x] No public database exposure (Private IP only)
[x] WAF protecting web tier (Cloud Armor)
[x] Vulnerability scanning (Container Analysis on GKE)
[x] Business Associate Agreement signed with GCP
---
📚 Lessons Learned
VPC Service Controls perimeters need careful planning — overly strict rules break legitimate service communications
Cloud Armor OWASP rules require tuning — several rules triggered on legitimate API calls initially
Audit log data access logging significantly increases log volume and cost — filter to PHI-relevant services only
IAP + GKE integration requires specific OAuth consent screen configuration not covered in standard docs
---
👤 Author
Moses Gnamisan Daniel — Cloud Security & DevSecOps Engineer
![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat&logo=linkedin)
