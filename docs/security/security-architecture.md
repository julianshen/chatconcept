# Security Architecture

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Overview](#1-overview)
2. [Security Principles](#2-security-principles)
3. [Data Encryption](#3-data-encryption)
4. [Network Security](#4-network-security)
5. [Access Control](#5-access-control)
6. [Secrets Management](#6-secrets-management)
7. [Audit Logging](#7-audit-logging)
8. [Security Monitoring](#8-security-monitoring)
9. [Incident Response](#9-incident-response)
10. [Application Security](#10-application-security)
11. [Compliance](#11-compliance)
12. [Security Testing](#12-security-testing)

---

## 1. Overview

This document defines the security architecture for the chat platform, covering defense-in-depth strategies across all layers: network, infrastructure, application, and data.

### Security Goals

| Goal | Description |
|------|-------------|
| **Confidentiality** | Protect data from unauthorized access |
| **Integrity** | Ensure data accuracy and prevent tampering |
| **Availability** | Maintain service access and resilience |
| **Accountability** | Track and audit all security-relevant actions |
| **Non-repudiation** | Prove origin and delivery of messages |

### Threat Model

| Threat Actor | Capabilities | Primary Targets |
|--------------|--------------|-----------------|
| External Attacker | Network access, credential theft | Authentication, API endpoints |
| Malicious Insider | Internal access, elevated privileges | Data exfiltration, privilege escalation |
| Compromised Bot | API access, automated operations | Mass messaging, data harvesting |
| Supply Chain | Malicious dependencies, compromised builds | Code execution, backdoors |

### Security Layers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Security Architecture                              │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Layer 1: Network Security                                           │   │
│  │  • WAF, DDoS protection, firewall rules, VPC isolation              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Layer 2: Transport Security                                         │   │
│  │  • TLS 1.3, mTLS for service-to-service, certificate management     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Layer 3: Application Security                                       │   │
│  │  • Authentication, authorization, input validation, rate limiting   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Layer 4: Data Security                                              │   │
│  │  • Encryption at rest, field-level encryption, key management       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Layer 5: Operational Security                                       │   │
│  │  • Audit logging, monitoring, incident response, access reviews     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Security Principles

### 2.1 Defense in Depth

No single security control is sufficient. Multiple overlapping controls protect against failures:

| Layer | Primary Control | Backup Control |
|-------|-----------------|----------------|
| Network | Firewall | WAF |
| Authentication | OIDC/JWT | Session validation |
| Authorization | Scope checks | Channel membership |
| Data | Encryption at rest | Field-level encryption |

### 2.2 Least Privilege

Every component operates with minimum required permissions:

```yaml
# Example: Message Writer Worker permissions
permissions:
  nats:
    - stream: MESSAGES
      operations: [consume]
  cassandra:
    - keyspace: chat
      tables: [messages]
      operations: [insert, update]
  # No MongoDB access
  # No Redis access
  # No S3 access
```

### 2.3 Zero Trust

No implicit trust based on network location:

- All service-to-service calls authenticated via mTLS
- All API requests validated regardless of source
- Internal services cannot bypass authentication
- Network segmentation does not replace authentication

### 2.4 Fail Secure

Security controls fail closed, not open:

```go
// Example: Authorization check
func (h *Handler) SendMessage(ctx context.Context, req *SendMessageRequest) error {
    // If authorization service is unavailable, deny the request
    allowed, err := h.authz.Check(ctx, req.UserID, req.ChannelID, "write")
    if err != nil {
        // Fail secure: deny on error
        return ErrAuthorizationFailed
    }
    if !allowed {
        return ErrForbidden
    }
    // ...
}
```

---

## 3. Data Encryption

### 3.1 Encryption at Rest

All persistent data is encrypted at rest:

| Data Store | Encryption Method | Key Management |
|------------|-------------------|----------------|
| **Cassandra** | Transparent Data Encryption (TDE) | AWS KMS / HashiCorp Vault |
| **MongoDB** | Encrypted Storage Engine | AWS KMS / HashiCorp Vault |
| **Redis** | Redis TLS + OS-level disk encryption | Instance-level encryption |
| **S3** | SSE-S3 or SSE-KMS | AWS KMS |
| **NATS JetStream** | OS-level disk encryption | Instance-level encryption |
| **Elasticsearch** | Encrypted at rest (X-Pack) | Elasticsearch keystore |

### 3.2 Encryption Configuration

**Cassandra TDE:**
```yaml
# cassandra.yaml
transparent_data_encryption_options:
  enabled: true
  chunk_length_kb: 64
  cipher: AES/CBC/PKCS5Padding
  key_alias: cassandra_key
  key_provider:
    - class_name: org.apache.cassandra.security.KmsKeyProvider
      parameters:
        - kms_key_id: alias/cassandra-chat
          region: us-east-1
```

**MongoDB Encrypted Storage:**
```yaml
# mongod.conf
security:
  enableEncryption: true
  encryptionCipherMode: AES256-CBC
  encryptionKeyFile: /etc/mongodb/encryption-key
  kmip:
    serverName: vault.example.com
    port: 5696
    clientCertificateFile: /etc/mongodb/client.pem
```

**S3 Bucket Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::chat-uploads/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

### 3.3 Field-Level Encryption

Sensitive fields encrypted before storage:

| Data Type | Encrypted Fields | Algorithm |
|-----------|------------------|-----------|
| **Refresh Tokens** | `token` | AES-256-GCM |
| **Bot Tokens** | N/A (stored hashed) | SHA-256 |
| **Webhook Secrets** | `secret` | AES-256-GCM |
| **Device Push Tokens** | `token` | AES-256-GCM |

```go
// Field-level encryption for refresh tokens
type TokenEncryptor struct {
    key []byte // 32 bytes for AES-256
}

func (e *TokenEncryptor) Encrypt(plaintext string) (string, error) {
    block, _ := aes.NewCipher(e.key)
    gcm, _ := cipher.NewGCM(block)

    nonce := make([]byte, gcm.NonceSize())
    io.ReadFull(rand.Reader, nonce)

    ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func (e *TokenEncryptor) Decrypt(encrypted string) (string, error) {
    data, _ := base64.StdEncoding.DecodeString(encrypted)

    block, _ := aes.NewCipher(e.key)
    gcm, _ := cipher.NewGCM(block)

    nonceSize := gcm.NonceSize()
    nonce, ciphertext := data[:nonceSize], data[nonceSize:]

    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    return string(plaintext), err
}
```

### 3.4 Encryption in Transit

All network communication encrypted with TLS 1.3:

| Connection | Protocol | Certificate |
|------------|----------|-------------|
| Client → API Gateway | TLS 1.3 | Public CA (Let's Encrypt) |
| API Gateway → Services | mTLS | Internal CA |
| Service → Database | TLS 1.2+ | Internal CA |
| Service → NATS | TLS 1.2+ | Internal CA |
| Cross-region NATS | TLS 1.3 | Internal CA |

**Envoy TLS Configuration:**
```yaml
# API Gateway downstream TLS
tls_context:
  common_tls_context:
    tls_params:
      tls_minimum_protocol_version: TLSv1_3
      tls_maximum_protocol_version: TLSv1_3
      cipher_suites:
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256
    tls_certificates:
      - certificate_chain:
          filename: /etc/envoy/certs/server.crt
        private_key:
          filename: /etc/envoy/certs/server.key
```

### 3.5 Key Management

**Key Hierarchy:**

```
┌─────────────────────────────────────────────┐
│           Master Key (AWS KMS)              │
│  • Never leaves KMS                         │
│  • Used to encrypt data keys                │
└────────────────────┬────────────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Cassandra   │ │  MongoDB    │ │    S3       │
│ Data Key    │ │ Data Key    │ │ Data Key    │
└─────────────┘ └─────────────┘ └─────────────┘
```

**Key Rotation:**

| Key Type | Rotation Period | Procedure |
|----------|-----------------|-----------|
| KMS Master Key | Annual | Automatic via KMS |
| Database Data Keys | Quarterly | Re-encrypt with new key |
| TLS Certificates | 90 days | Automatic via cert-manager |
| Service Account Keys | Monthly | Rolling deployment |

---

## 4. Network Security

### 4.1 Network Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Internet                                        │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                          ┌─────────▼─────────┐
                          │   CloudFlare      │
                          │   (WAF + DDoS)    │
                          └─────────┬─────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                         Public Subnet                                        │
│                   ┌───────────────┴───────────────┐                         │
│                   │      Load Balancer            │                         │
│                   │    (Network Load Balancer)    │                         │
│                   └───────────────┬───────────────┘                         │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                         DMZ Subnet (10.0.1.0/24)                            │
│                   ┌───────────────┴───────────────┐                         │
│                   │       API Gateway             │                         │
│                   │         (Envoy)               │                         │
│                   └───────────────┬───────────────┘                         │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                       Application Subnet (10.0.2.0/24)                      │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐    │
│   │  Command    │   │   Query     │   │  Fan-Out    │   │   Notif     │    │
│   │  Service    │   │  Service    │   │  Service    │   │  Service    │    │
│   └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘    │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                         Data Subnet (10.0.3.0/24)                           │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐    │
│   │  Cassandra  │   │   MongoDB   │   │    NATS     │   │   Redis     │    │
│   └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Security Groups

**API Gateway Security Group:**
```hcl
resource "aws_security_group" "api_gateway" {
  name   = "api-gateway-sg"
  vpc_id = aws_vpc.main.id

  # Inbound: HTTPS from Load Balancer only
  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.load_balancer.id]
  }

  # Outbound: Application subnet only
  egress {
    from_port   = 8080
    to_port     = 8090
    protocol    = "tcp"
    cidr_blocks = ["10.0.2.0/24"]
  }
}
```

**Application Services Security Group:**
```hcl
resource "aws_security_group" "app_services" {
  name   = "app-services-sg"
  vpc_id = aws_vpc.main.id

  # Inbound: From API Gateway only
  ingress {
    from_port       = 8080
    to_port         = 8090
    protocol        = "tcp"
    security_groups = [aws_security_group.api_gateway.id]
  }

  # Inbound: Inter-service communication
  ingress {
    from_port = 8080
    to_port   = 8090
    protocol  = "tcp"
    self      = true
  }

  # Outbound: Data subnet
  egress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["10.0.3.0/24"]
  }
}
```

**Data Stores Security Group:**
```hcl
resource "aws_security_group" "data_stores" {
  name   = "data-stores-sg"
  vpc_id = aws_vpc.main.id

  # Cassandra: From app services only
  ingress {
    from_port       = 9042
    to_port         = 9042
    protocol        = "tcp"
    security_groups = [aws_security_group.app_services.id]
  }

  # MongoDB: From app services only
  ingress {
    from_port       = 27017
    to_port         = 27017
    protocol        = "tcp"
    security_groups = [aws_security_group.app_services.id]
  }

  # NATS: From app services only
  ingress {
    from_port       = 4222
    to_port         = 4222
    protocol        = "tcp"
    security_groups = [aws_security_group.app_services.id]
  }

  # Redis: From app services only
  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app_services.id]
  }

  # No outbound to internet
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["10.0.0.0/16"]
  }
}
```

### 4.3 WAF Rules

**CloudFlare WAF Configuration:**

| Rule | Action | Description |
|------|--------|-------------|
| OWASP Core Ruleset | Block | SQL injection, XSS, path traversal |
| Rate Limiting | Challenge | > 100 req/10s per IP |
| Bot Management | Challenge | Known bad bots |
| Geo Blocking | Block | Sanctioned countries |
| File Upload Scanning | Block | Malicious file signatures |

### 4.4 DDoS Protection

| Layer | Protection | Capacity |
|-------|------------|----------|
| L3/L4 | CloudFlare Spectrum | 100+ Tbps |
| L7 | CloudFlare WAF + Rate Limiting | Adaptive |
| Application | Per-user rate limiting | Configurable |

### 4.5 mTLS for Service-to-Service

All internal service communication uses mutual TLS:

```yaml
# Service mesh configuration (Istio/Linkerd)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: chat
spec:
  mtls:
    mode: STRICT
```

---

## 5. Access Control

### 5.1 Authentication Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Authentication Methods                                │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Users                                                               │   │
│  │  • Keycloak OIDC (Authorization Code + PKCE)                        │   │
│  │  • JWT access tokens (15 min TTL)                                   │   │
│  │  • Refresh tokens (7 day TTL, server-side)                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Bots                                                                │   │
│  │  • Long-lived bot tokens (xbot-*)                                   │   │
│  │  • Scope-based permissions                                          │   │
│  │  • Channel installation required                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Services (Internal)                                                 │   │
│  │  • mTLS certificates                                                │   │
│  │  • Service account tokens                                           │   │
│  │  • SPIFFE/SPIRE identities                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Webhooks (Inbound)                                                  │   │
│  │  • HMAC-SHA256 signature verification                               │   │
│  │  • Timestamp validation (5 min freshness)                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Authorization Model

**Role-Based Access Control (RBAC):**

| Role | Scope | Permissions |
|------|-------|-------------|
| `user` | Global | Read public channels, join channels, send messages |
| `channel_admin` | Channel | Manage members, edit settings, pin messages |
| `channel_moderator` | Channel | Delete messages, kick members |
| `team_admin` | Team | Manage team channels, add/remove members |
| `admin` | Global | All permissions, user management |

**Permission Checks:**

```go
// Authorization service interface
type Authorizer interface {
    // Check if user can perform action on resource
    Check(ctx context.Context, subject, resource, action string) (bool, error)

    // Get user's roles for a resource
    GetRoles(ctx context.Context, subject, resource string) ([]string, error)
}

// Permission check in Command Service
func (h *Handler) SendMessage(ctx context.Context, req *SendMessageRequest) error {
    userID := ctx.Value("user_id").(string)

    // Check channel membership
    isMember, err := h.authz.Check(ctx, userID, req.ChannelID, "read")
    if err != nil || !isMember {
        return ErrForbidden
    }

    // Check write permission (not in read-only channels)
    canWrite, err := h.authz.Check(ctx, userID, req.ChannelID, "write")
    if err != nil || !canWrite {
        return ErrForbidden
    }

    // Check rate limit
    if !h.rateLimiter.Allow(userID, "message") {
        return ErrRateLimited
    }

    // Proceed with message
    // ...
}
```

### 5.3 Channel Permission Model

```javascript
// MongoDB: Channel permissions
{
  channel_id: "ch_engineering",
  type: "private",  // "public" | "private" | "dm"

  // Default permissions for members
  default_permissions: {
    read: true,
    write: true,
    react: true,
    upload: true,
    pin: false,
    delete_own: true,
    delete_any: false
  },

  // Role overrides
  role_permissions: {
    "channel_admin": {
      pin: true,
      delete_any: true,
      manage_members: true,
      edit_settings: true
    },
    "channel_moderator": {
      delete_any: true,
      kick_members: true
    }
  },

  // Member list with roles
  members: [
    { user_id: "usr_alice", roles: ["channel_admin"] },
    { user_id: "usr_bob", roles: ["channel_moderator"] },
    { user_id: "usr_carol", roles: [] }  // default permissions
  ]
}
```

### 5.4 Bot Scopes

| Scope | Permissions |
|-------|-------------|
| `messages:read` | Read messages in installed channels |
| `messages:write` | Send messages, reply to threads |
| `messages:history` | Access message history |
| `channels:read` | List channels, get channel info |
| `channels:write` | Create channels, update settings |
| `users:read` | List users, get user profiles |
| `reactions:write` | Add/remove reactions |
| `files:read` | Access file metadata |
| `files:write` | Upload files |

---

## 6. Secrets Management

### 6.1 Secret Types

| Secret Type | Storage | Rotation |
|-------------|---------|----------|
| OIDC Client Secret | HashiCorp Vault | 90 days |
| Database Passwords | HashiCorp Vault | 30 days |
| NATS NKeys | HashiCorp Vault | 90 days |
| S3 Access Keys | IAM Roles (no static keys) | N/A |
| TLS Certificates | cert-manager | 90 days |
| Encryption Keys | AWS KMS | Annual |

### 6.2 Vault Configuration

```hcl
# Vault secret engine for database credentials
path "database/creds/chat-app" {
  capabilities = ["read"]
}

# Dynamic database credentials
resource "vault_database_secret_backend_role" "chat_app" {
  backend             = vault_mount.db.path
  name                = "chat-app"
  db_name             = vault_database_secret_backend_connection.cassandra.name
  creation_statements = ["CREATE USER '{{name}}' WITH PASSWORD '{{password}}' SUPERUSER"]
  default_ttl         = "1h"
  max_ttl             = "24h"
}
```

### 6.3 Secret Injection

**Kubernetes Secrets via Vault Agent:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: command-service
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "command-service"
        vault.hashicorp.com/agent-inject-secret-db: "database/creds/chat-app"
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "database/creds/chat-app" -}}
          export CASSANDRA_USERNAME="{{ .Data.username }}"
          export CASSANDRA_PASSWORD="{{ .Data.password }}"
          {{- end -}}
```

### 6.4 Secret Detection

**Pre-commit Hook:**
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

**CI Pipeline Secret Scanning:**
```yaml
# GitHub Actions
- name: Scan for secrets
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.repository.default_branch }}
    head: HEAD
```

---

## 7. Audit Logging

### 7.1 Audit Events

| Category | Events | Retention |
|----------|--------|-----------|
| **Authentication** | Login, logout, token refresh, failed auth | 1 year |
| **Authorization** | Permission denied, role changes | 1 year |
| **Data Access** | Message read (admin), bulk export | 1 year |
| **Data Modification** | Message delete, channel delete, user deactivate | 2 years |
| **Admin Actions** | Config changes, user management | 2 years |
| **Security Events** | Rate limit triggers, suspicious activity | 2 years |

### 7.2 Audit Log Schema

```json
{
  "event_id": "audit_01HZ3K...",
  "timestamp": "2026-02-01T10:30:00.123Z",
  "event_type": "auth.login.success",
  "severity": "INFO",

  "actor": {
    "type": "user",
    "id": "usr_alice",
    "username": "alice",
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "session_id": "sess_01HZ3K..."
  },

  "resource": {
    "type": "session",
    "id": "sess_01HZ3K..."
  },

  "action": {
    "name": "create",
    "result": "success"
  },

  "context": {
    "auth_method": "oidc",
    "provider": "keycloak",
    "device_type": "web",
    "geo": {
      "country": "US",
      "region": "CA",
      "city": "San Francisco"
    }
  },

  "request": {
    "id": "req_abc123",
    "trace_id": "trace_xyz789"
  }
}
```

### 7.3 Audit Log Pipeline

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Services   │────►│  AUDIT NATS  │────►│   Audit      │────►│   S3 +       │
│              │     │   Stream     │     │   Processor  │     │ Elasticsearch│
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                │
                                                │ Real-time alerts
                                                ▼
                                         ┌──────────────┐
                                         │    SIEM      │
                                         │  (Splunk/    │
                                         │   Datadog)   │
                                         └──────────────┘
```

### 7.4 Log Integrity

**Tamper-evident logging:**

```go
// Each log entry includes hash of previous entry
type AuditEntry struct {
    EventID      string    `json:"event_id"`
    Timestamp    time.Time `json:"timestamp"`
    // ... other fields
    PreviousHash string    `json:"previous_hash"`
    EntryHash    string    `json:"entry_hash"`
}

func (e *AuditEntry) ComputeHash(previousHash string) string {
    e.PreviousHash = previousHash
    data, _ := json.Marshal(e)
    hash := sha256.Sum256(data)
    e.EntryHash = hex.EncodeToString(hash[:])
    return e.EntryHash
}
```

---

## 8. Security Monitoring

### 8.1 Security Metrics

| Metric | Threshold | Alert |
|--------|-----------|-------|
| Failed logins per user | > 5 in 5 min | Warning |
| Failed logins per IP | > 20 in 5 min | Critical |
| Rate limit triggers | > 100 in 1 min | Warning |
| 4xx responses | > 10% of requests | Warning |
| 5xx responses | > 1% of requests | Critical |
| Certificate expiry | < 14 days | Warning |
| Unusual data access | Anomaly detection | Investigation |

### 8.2 Alerting Rules

```yaml
# Prometheus alerting rules
groups:
  - name: security
    rules:
      - alert: HighFailedLoginRate
        expr: |
          sum(rate(auth_login_failed_total[5m])) by (ip) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High failed login rate from {{ $labels.ip }}"

      - alert: SuspiciousDataAccess
        expr: |
          sum(rate(admin_data_access_total[1h])) > 100
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Unusual admin data access pattern detected"

      - alert: CertificateExpiringSoon
        expr: |
          (cert_expiry_timestamp_seconds - time()) < 86400 * 14
        labels:
          severity: warning
        annotations:
          summary: "Certificate {{ $labels.cn }} expires in < 14 days"
```

### 8.3 Anomaly Detection

**Behavioral Analysis:**

| Behavior | Normal | Anomaly |
|----------|--------|---------|
| Login location | Consistent geo | New country |
| Login time | Business hours | 3 AM |
| Message rate | < 10/min | > 100/min |
| Channel joins | < 5/day | > 50/day |
| File uploads | < 10 MB/day | > 1 GB/day |

```python
# Example: Login location anomaly detection
def check_login_anomaly(user_id: str, login_event: dict) -> bool:
    recent_logins = get_recent_logins(user_id, days=30)
    known_countries = set(l.country for l in recent_logins)

    if login_event.country not in known_countries:
        # New country - trigger verification
        trigger_mfa_challenge(user_id)
        create_security_alert(
            type="new_login_location",
            user_id=user_id,
            details=login_event
        )
        return True
    return False
```

---

## 9. Incident Response

### 9.1 Incident Classification

| Severity | Definition | Response Time | Examples |
|----------|------------|---------------|----------|
| **P1 Critical** | Active breach, data exposure | 15 min | Unauthorized access, data leak |
| **P2 High** | Potential breach, service impact | 1 hour | Brute force attack, DDoS |
| **P3 Medium** | Security weakness, no active threat | 4 hours | Vulnerable dependency, misconfig |
| **P4 Low** | Minor issue, minimal risk | 24 hours | Policy violation, audit finding |

### 9.2 Incident Response Playbook

**Phase 1: Detection & Triage (0-15 min)**
1. Acknowledge alert
2. Assess scope and severity
3. Notify incident commander
4. Begin evidence preservation

**Phase 2: Containment (15-60 min)**
1. Isolate affected systems
2. Block malicious IPs/accounts
3. Revoke compromised credentials
4. Enable enhanced logging

**Phase 3: Eradication (1-4 hours)**
1. Identify root cause
2. Remove malicious artifacts
3. Patch vulnerabilities
4. Verify system integrity

**Phase 4: Recovery (4-24 hours)**
1. Restore services gradually
2. Monitor for recurrence
3. Validate security controls
4. Update detection rules

**Phase 5: Post-Incident (1-7 days)**
1. Complete incident report
2. Conduct root cause analysis
3. Implement preventive measures
4. Update runbooks

### 9.3 Emergency Procedures

**Account Compromise:**
```bash
# Immediately revoke all user sessions
POST /api/v1/admin/users/{user_id}/revoke-sessions
Authorization: Bearer <admin_token>

# Disable account
POST /api/v1/admin/users/{user_id}/disable

# Force password reset in Keycloak
keycloak-admin update users/{keycloak_user_id} -s "requiredActions=[UPDATE_PASSWORD]"
```

**Bot Token Compromise:**
```bash
# Rotate bot token immediately
POST /api/v1/bots/{bot_id}/rotate-token
Authorization: Bearer <admin_token>

# Review bot activity logs
GET /api/v1/admin/audit?actor_type=bot&actor_id={bot_id}&since=24h
```

---

## 10. Application Security

### 10.1 Input Validation

**All user input validated:**

```go
// Message content validation
func validateMessage(msg *MessageRequest) error {
    // Length limits
    if len(msg.Text) > 50000 {
        return ErrMessageTooLong
    }

    // No null bytes
    if strings.ContainsRune(msg.Text, '\x00') {
        return ErrInvalidContent
    }

    // Validate mentions format
    for _, mention := range extractMentions(msg.Text) {
        if !isValidUserID(mention) && !isValidSpecialMention(mention) {
            return ErrInvalidMention
        }
    }

    // Validate attachments
    for _, att := range msg.Attachments {
        if err := validateAttachment(att); err != nil {
            return err
        }
    }

    return nil
}
```

### 10.2 Output Encoding

**Context-aware encoding for all outputs:**

| Context | Encoding | Example |
|---------|----------|---------|
| HTML body | HTML entities | `<` → `&lt;` |
| HTML attribute | Attribute encoding | `"` → `&quot;` |
| JavaScript | JS encoding | `'` → `\'` |
| URL | URL encoding | `&` → `%26` |
| JSON | JSON encoding | Native |

### 10.3 Security Headers

**API Gateway Response Headers:**

```yaml
response_headers_to_add:
  - header:
      key: "Strict-Transport-Security"
      value: "max-age=31536000; includeSubDomains; preload"
  - header:
      key: "X-Content-Type-Options"
      value: "nosniff"
  - header:
      key: "X-Frame-Options"
      value: "DENY"
  - header:
      key: "X-XSS-Protection"
      value: "1; mode=block"
  - header:
      key: "Content-Security-Policy"
      value: "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' https://cdn.example.com data:; connect-src 'self' wss://ws.example.com"
  - header:
      key: "Referrer-Policy"
      value: "strict-origin-when-cross-origin"
  - header:
      key: "Permissions-Policy"
      value: "geolocation=(), microphone=(), camera=()"
```

### 10.4 Dependency Security

**Automated vulnerability scanning:**

```yaml
# GitHub Actions - Dependency Scanning
- name: Run Snyk
  uses: snyk/actions/golang@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high

- name: Run Trivy
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

**Dependency Update Policy:**

| Severity | SLA | Action |
|----------|-----|--------|
| Critical | 24 hours | Emergency patch |
| High | 7 days | Priority update |
| Medium | 30 days | Regular update |
| Low | 90 days | Scheduled update |

---

## 11. Compliance

### 11.1 Data Classification

| Classification | Examples | Handling |
|----------------|----------|----------|
| **Public** | Channel names, public profiles | No restrictions |
| **Internal** | Message content, file metadata | Authenticated access |
| **Confidential** | DMs, private channels | Encryption, access control |
| **Restricted** | Tokens, passwords, PII | Field-level encryption, audit |

### 11.2 GDPR Compliance

| Requirement | Implementation |
|-------------|----------------|
| **Right to Access** | Data export API (`GET /api/v1/users/me/export`) |
| **Right to Erasure** | User deletion with cascade (`DELETE /api/v1/users/{id}`) |
| **Data Portability** | JSON export of all user data |
| **Consent** | Keycloak consent management |
| **Data Minimization** | Retention policies, auto-deletion |
| **Breach Notification** | Incident response < 72 hours |

### 11.3 Data Retention

| Data Type | Default Retention | Configurable |
|-----------|-------------------|--------------|
| Messages | 2 years | Per-channel |
| Files | 1 year | Per-channel |
| Audit Logs | 2 years | No |
| Session Data | 7 days | No |
| Deleted Content | 30 days (soft delete) | No |

### 11.4 SOC 2 Controls

| Control | Implementation |
|---------|----------------|
| **CC6.1** Access Control | RBAC, MFA via Keycloak |
| **CC6.2** System Boundaries | Network segmentation, firewalls |
| **CC6.3** Change Management | GitOps, PR reviews, CI/CD |
| **CC7.1** Security Monitoring | SIEM integration, alerting |
| **CC7.2** Incident Response | Documented playbooks |

---

## 12. Security Testing

### 12.1 Testing Schedule

| Test Type | Frequency | Scope |
|-----------|-----------|-------|
| SAST (Static Analysis) | Every commit | All code |
| DAST (Dynamic Analysis) | Weekly | Staging environment |
| Dependency Scan | Daily | All dependencies |
| Penetration Test | Quarterly | Full platform |
| Red Team Exercise | Annual | Organization-wide |

### 12.2 Penetration Testing Scope

**In Scope:**
- All API endpoints
- Authentication flows
- WebSocket connections
- File upload/download
- Bot integrations
- Admin interfaces

**Out of Scope:**
- Third-party services (Keycloak, AWS)
- Physical security
- Social engineering (without approval)
- Denial of service

### 12.3 Vulnerability Disclosure

**Responsible Disclosure Policy:**

1. Report to: security@example.com
2. PGP key available at: https://example.com/.well-known/security.txt
3. Response within 48 hours
4. Coordinated disclosure after 90 days
5. Recognition in security hall of fame

---

## Related Documents

- [Authentication](../features/authentication.md) — OIDC integration details
- [Bot Integration](../features/bot-integration.md) — Bot security
- [File Uploads](../features/file-uploads.md) — File security
- [API Reference](../api/api-reference.md) — Rate limiting
- [Risks](../appendices/risks.md) — Risk assessment
