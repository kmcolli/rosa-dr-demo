# Architecture: ROSA HCP Disaster Recovery with ACM and OADP

## Steady-State Architecture

```mermaid
graph TB
    subgraph Users["End Users"]
        USER["👤 Users / Demo Audience"]
    end

    subgraph DNS["AWS Route 53"]
        R53_OADP["mission-control.mobb.cloud<br/><i>Failover routing + health check</i>"]
        R53_ACM["acm-mission-control.mobb.cloud<br/><i>A record → primary</i>"]
    end

    subgraph ACM_HUB["ACM Hub Cluster"]
        ACM_CTL["Advanced Cluster Management"]
        GITOPS["OpenShift GitOps / ArgoCD"]
        APPSET["ApplicationSet<br/><i>Deploys to both clusters</i>"]
        PLACEMENT["Placement + PlacementDecision<br/><i>Cluster health-aware</i>"]
        ACM_CTL --> PLACEMENT
        GITOPS --> APPSET
    end

    subgraph PRIMARY["AWS us-east-1 (Primary Region)"]
        subgraph ROSA1["ROSA HCP — Primary Cluster"]
            OADP_APP_P["OADP App ✅<br/><i>Active</i>"]
            ACM_APP_P["ACM App ✅<br/><i>Active</i>"]
            VELERO_P["Velero / OADP<br/><i>Scheduled backups</i>"]
            KLUSTERLET_P["Klusterlet<br/><i>Heartbeat → ACM</i>"]
        end
        EFS_P["EFS<br/><i>Primary — read/write</i>"]
        S3_APP_P["S3: app-data"]
        S3_OADP_P["S3: oadp-backups"]
    end

    subgraph DR["AWS us-west-2 (DR Region)"]
        subgraph ROSA2["ROSA HCP — DR Cluster"]
            ACM_APP_D["ACM App ✅<br/><i>Active (via GitOps)</i>"]
            OADP_APP_D["OADP App ⏸️<br/><i>Not deployed</i>"]
            VELERO_D["Velero / OADP<br/><i>Backup target synced via S3 CRR</i>"]
            KLUSTERLET_D["Klusterlet<br/><i>Heartbeat → ACM</i>"]
        end
        EFS_D["EFS<br/><i>Replica — read-only</i>"]
        S3_APP_D["S3: app-data"]
        S3_OADP_D["S3: oadp-backups"]
    end

    USER --> R53_OADP
    USER --> R53_ACM
    R53_OADP -->|"Active"| OADP_APP_P
    R53_OADP -.->|"Standby"| OADP_APP_D
    R53_ACM --> ACM_APP_P

    KLUSTERLET_P -->|heartbeat| ACM_CTL
    KLUSTERLET_D -->|heartbeat| ACM_CTL
    APPSET -->|sync| ACM_APP_P
    APPSET -->|sync| ACM_APP_D

    OADP_APP_P --> EFS_P
    OADP_APP_P --> S3_APP_P
    ACM_APP_P --> EFS_P
    ACM_APP_P --> S3_APP_P
    ACM_APP_D --> EFS_D
    ACM_APP_D --> S3_APP_D
    VELERO_P -->|backup| S3_OADP_P

    EFS_P ==>|"EFS Replication"| EFS_D
    S3_APP_P ==>|"S3 Cross-Region Replication"| S3_APP_D
    S3_OADP_P ==>|"S3 Cross-Region Replication"| S3_OADP_D

    classDef primary fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    classDef dr fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef acm fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef dns fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef repl stroke:#ff6f00,stroke-width:3px,stroke-dasharray: 5 5

    class ROSA1,OADP_APP_P,ACM_APP_P,VELERO_P,EFS_P,S3_APP_P,S3_OADP_P primary
    class ROSA2,ACM_APP_D,OADP_APP_D,VELERO_D,EFS_D,S3_APP_D,S3_OADP_D dr
    class ACM_HUB,ACM_CTL,GITOPS,APPSET,PLACEMENT acm
    class R53_OADP,R53_ACM dns
```

## Failover: What Happens When us-east-1 Goes Down

```mermaid
sequenceDiagram
    participant Demo as Demo Operator
    participant Primary as Primary Cluster<br/>(us-east-1)
    participant ACM as ACM Hub
    participant DR as DR Cluster<br/>(us-west-2)
    participant R53 as Route 53
    participant EFS as EFS Replication
    participant User as End Users

    Note over Primary,DR: Steady State — both apps serving from Primary

    rect rgb(255, 235, 238)
        Note over Demo,Primary: Simulate Region Failure
        Demo->>EFS: Delete replication<br/>(promote DR to read-write)
        Demo->>Primary: Stop all worker nodes
        Primary--xUser: ❌ Both apps unreachable
    end

    rect rgb(227, 242, 253)
        Note over ACM,DR: ACM Recovery Path (~90 seconds)
        Primary--xACM: Klusterlet heartbeat stops
        ACM->>ACM: Detect failure (~40s)<br/>ManagedCluster → Unknown
        ACM->>ACM: PlacementDecision updates
        Note over DR: ACM App already running ✅
        Demo->>R53: Update ACM DNS → DR IP
        R53->>DR: acm-mission-control.mobb.cloud
        DR->>User: ✅ ACM App recovered
        Note right of User: ~90 seconds total
    end

    rect rgb(232, 245, 233)
        Note over Demo,DR: OADP Recovery Path (~5 minutes)
        Demo->>DR: Log into DR cluster
        Demo->>DR: Run recover-efs-volumes.sh<br/>(create access points + static PVs)
        Demo->>DR: Velero restore from backup<br/>(deployments, services, routes)
        Demo->>DR: Update env vars for DR region<br/>(S3 bucket, IAM role, region)
        Note over R53: Health check detects<br/>primary down → auto-failover
        R53->>DR: mission-control.mobb.cloud
        DR->>User: ✅ OADP App recovered
        Note right of User: ~5 minutes total
    end

    Note over Primary,DR: Both apps now serving from us-west-2
```

## Component Summary

```mermaid
graph LR
    subgraph Products["Red Hat Products"]
        ROSA["ROSA HCP<br/>━━━━━━━━━━<br/>Managed OpenShift<br/>Hosted Control Plane<br/>Multi-region clusters"]
        ACM_P["ACM<br/>━━━━━━━━━━<br/>Cluster lifecycle<br/>Health monitoring<br/>Placement decisions"]
        GITOPS_P["OpenShift GitOps<br/>━━━━━━━━━━<br/>ArgoCD<br/>ApplicationSet<br/>Multi-cluster deploy"]
        OADP_P["OADP<br/>━━━━━━━━━━<br/>Velero integration<br/>Backup / Restore<br/>K8s resource recovery"]
    end

    subgraph AWS["AWS Infrastructure"]
        EFS_I["EFS<br/>━━━━━━━━━━<br/>Cross-region<br/>replication<br/>Shared file storage"]
        S3_I["S3<br/>━━━━━━━━━━<br/>Cross-region<br/>replication<br/>Object storage"]
        R53_I["Route 53<br/>━━━━━━━━━━<br/>Failover routing<br/>Health checks<br/>DNS switching"]
        EC2_I["EC2<br/>━━━━━━━━━━<br/>Worker nodes<br/>Stop/start for<br/>failure simulation"]
    end

    ROSA --- ACM_P
    ROSA --- GITOPS_P
    ROSA --- OADP_P
    ROSA --- EFS_I
    ROSA --- S3_I
    ACM_P --- GITOPS_P

    classDef redhat fill:#ee0000,stroke:#cc0000,color:#fff,stroke-width:2px
    classDef aws fill:#ff9900,stroke:#cc7a00,color:#fff,stroke-width:2px
    class ROSA,ACM_P,GITOPS_P,OADP_P redhat
    class EFS_I,S3_I,R53_I,EC2_I aws
```

## Recovery Comparison

| | ACM + GitOps | OADP + Velero |
|---|---|---|
| **RTO** | ~90 seconds | ~5 minutes |
| **Steady-state** | App on both clusters | App on primary only |
| **Failover trigger** | ACM detects via klusterlet | Manual / health check |
| **Recovery steps** | DNS switch only | EFS volumes → Velero restore → env update → DNS |
| **Data strategy** | EFS replication + S3 CRR | EFS replication + S3 CRR + Velero backup |
| **Cost** | Higher (2 clusters active) | Lower (1 cluster active) |
| **Best for** | Low RTO, mission-critical | Cost-sensitive, moderate RTO |
