# ROSA HCP Disaster Recovery Demo: OADP and ACM Patterns

**Authors:** Kevin Collins, Diana Sari, Kumudu Herath
**Validated on:** OpenShift 4.22

This guide provides a scripted demo that walks through two ROSA HCP disaster recovery patterns in a single failure event. One simulated region outage exercises both recovery paths so the audience sees each approach in action:

- **OADP (backup and restore):** manual EFS volume recovery, Velero restore, DNS switch. Recovery takes approximately 5 minutes.
- **ACM with OpenShift GitOps (deploy to both):** application already running on the DR cluster. DNS switch only. Recovery takes approximately 90 seconds.

The demo takes 10-12 minutes of active time.

## Architecture

Both patterns share the same infrastructure: two ROSA HCP clusters in different AWS Regions, S3 Cross-Region Replication, and EFS replication.

- **OADP pattern:** application runs on the primary cluster only. A Velero backup captures Kubernetes resources. During failover, the backup is restored to the DR cluster and EFS volumes are manually recreated from a pre-recorded mapping.
- **ACM pattern:** an ArgoCD ApplicationSet deploys the application to both clusters simultaneously. ACM monitors cluster health. During failover, only DNS needs to change because the application is already running on the DR cluster.

Both applications use the same EFS and S3 infrastructure, so a single failure event demonstrates both recovery paths.

## Pre-Demo Setup

Complete these guides before the demo. The setup takes approximately 2 hours.

1. **[Create ROSA HCP Disaster Recovery Infrastructure](https://cloud.redhat.com/experts/rosa/rosa-dr-infra/)** -- EFS CSI Driver, S3 replication, EFS replication
2. **[Disaster Recovery with OADP](https://cloud.redhat.com/experts/rosa/oadp-efs-s3/)** -- complete through step 5 (backup created, DNS failover configured)
3. **[Disaster Recovery with ACM and OpenShift GitOps](https://cloud.redhat.com/experts/rosa/rosa-acm-dr/)** -- complete all steps (ApplicationSet deployed to both clusters)

### Pre-Demo Checklist

Run through this checklist 15 minutes before the demo starts.

```bash
export PRIMARY_CLUSTER_NAME=<primary-cluster-name>
export DR_CLUSTER_NAME=<dr-cluster-name>
export AWS_PAGER=""
export HOSTED_ZONE_ID=<your-hosted-zone-id>
export CLUSTER_ACM=<acm-hub-cluster-name>
export OADP_DOMAIN=<your-oadp-domain>          # e.g. mission-control.mobb.cloud
export ACM_DOMAIN=<your-acm-domain>            # e.g. acm-mission-control.mobb.cloud
```

Derive remaining variables from the cluster names and infrastructure:

```bash
export PRIMARY_REGION=$(rosa describe cluster -c $PRIMARY_CLUSTER_NAME -o json | jq -r '.region.id')
export DR_REGION=$(rosa describe cluster -c $DR_CLUSTER_NAME -o json | jq -r '.region.id')

export APP_BUCKET_PRIMARY=${PRIMARY_CLUSTER_NAME}-app-data
export APP_BUCKET_DR=${DR_CLUSTER_NAME}-app-data

export APP_S3_ROLE_ARN_DR=$(aws iam get-role \
  --role-name ${DR_CLUSTER_NAME}-dr-demo-s3 \
  --query 'Role.Arn' --output text)

export PRIMARY_EFS=$(aws efs describe-file-systems \
  --region $PRIMARY_REGION \
  --query "FileSystems[?Name!=\`null\` && ends_with(Name, '-dr-efs')].FileSystemId | [0]" \
  --output text)

export DR_EFS=$(aws efs describe-file-systems \
  --region $DR_REGION \
  --query "FileSystems[?Name!=\`null\` && ends_with(Name, '-dr-efs')].FileSystemId | [0]" \
  --output text)

echo "PRIMARY_REGION=$PRIMARY_REGION"
echo "DR_REGION=$DR_REGION"
echo "APP_BUCKET_PRIMARY=$APP_BUCKET_PRIMARY"
echo "APP_BUCKET_DR=$APP_BUCKET_DR"
echo "APP_S3_ROLE_ARN_DR=$APP_S3_ROLE_ARN_DR"
echo "PRIMARY_EFS=$PRIMARY_EFS"
echo "DR_EFS=$DR_EFS"
```

Verify both clusters are healthy, both apps are running, and DNS is pointing to the primary:

```bash
echo "=== OADP App ==="
curl -sk https://${OADP_DOMAIN}/healthz

echo "=== ACM App ==="
curl -sk https://${ACM_DOMAIN}/healthz

echo "=== ACM Managed Clusters ==="
oc get managedcluster -o custom-columns=\
'NAME:.metadata.name,AVAILABLE:.status.conditions[?(@.type=="ManagedClusterConditionAvailable")].status'

echo "=== ArgoCD Applications ==="
oc get applications.argoproj.io -n openshift-gitops
```

Verify the OADP backup exists and has replicated to the DR bucket:
** log into the primary cluster **

```bash
BACKUP_NAME=$(oc get backup -n openshift-adp \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1].metadata.name}')
echo "Latest backup: $BACKUP_NAME"

oc get backup $BACKUP_NAME -n openshift-adp \
  -o jsonpath='Phase: {.status.phase}'
```

Generate the EFS PVC mapping from the primary cluster:

```bash
./scripts/record-efs-mapping.sh \
  --namespace dr-demo \
  --region $PRIMARY_REGION \
  --output efs-pvc-map.csv

cat efs-pvc-map.csv
```

Verify EFS replication is active:

```bash
aws efs describe-replication-configurations \
  --file-system-id $PRIMARY_EFS --region $PRIMARY_REGION \
  --query 'Replications[0].Destinations[0].Status' --output text
```

Get the DR cluster router IP for DNS failover:

```bash
DR_CONSOLE_HOST=$(rosa describe cluster -c $DR_CLUSTER_NAME -o json \
  | jq -r '.console.url' | sed 's|^https://||')
DR_IP=$(host $DR_CONSOLE_HOST | awk '/has address/{print $NF; exit}')
echo "DR_IP=$DR_IP"
```

---

## Demo Script

### Act 1: Show the Running Applications (2 minutes)

> **Talking point:** "We have a mission-critical application running in us-east-1. We've configured two different DR strategies. Let me show you both apps are live."

Open both application URLs in browser tabs:

```bash
echo "OADP app:  https://${OADP_DOMAIN}"
echo "ACM app:   https://${ACM_DOMAIN}"
```

Show the ACM console -- both clusters healthy, both ArgoCD applications synced:

```bash
echo "=== Cluster Health ==="
oc get managedcluster -o custom-columns=\
'NAME:.metadata.name,AVAILABLE:.status.conditions[?(@.type=="ManagedClusterConditionAvailable")].status'

echo ""
echo "=== ArgoCD ==="
oc get applications.argoproj.io -n openshift-gitops
```

> **Talking point:** "The OADP app runs on the primary cluster only, with periodic backups. The ACM app runs on both clusters simultaneously via an ArgoCD ApplicationSet. Let's see what happens when the region goes down."

### Act 2: Simulate Region Failure (2 minutes)

Write validation data so the audience can verify data survived the failover:

```bash
export VALIDATION_ID=demo-$(date +%Y%m%d%H%M%S)

oc exec -n dr-demo sts/flight-recorder -- \
  sh -c "echo efs-$VALIDATION_ID > /data/flight-recorder/validation-$VALIDATION_ID.txt"

printf '%s\n' "s3-$VALIDATION_ID" | aws s3 cp - \
  "s3://$APP_BUCKET_PRIMARY/validation/$VALIDATION_ID.txt" \
  --region "$PRIMARY_REGION"

echo "Validation marker: $VALIDATION_ID"
```

Delete EFS replication to promote the DR replica to read-write. This step must happen before the failure because the DR application needs writable EFS:

```bash
aws efs delete-replication-configuration \
  --source-file-system-id "$PRIMARY_EFS" \
  --region "$PRIMARY_REGION"
```

Disable auto-repair and stop the primary workers:

```bash
for MP in $(rosa list machinepools -c $PRIMARY_CLUSTER_NAME -o json | jq -r '.[].id'); do
  rosa edit machinepool $MP --cluster $PRIMARY_CLUSTER_NAME --autorepair=false
done

PRIMARY_INSTANCE_IDS=($(aws ec2 describe-instances --region ${PRIMARY_REGION} \
  --filters "Name=tag:api.openshift.com/name,Values=${PRIMARY_CLUSTER_NAME}" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' --output text))

aws ec2 stop-instances \
  --instance-ids "${PRIMARY_INSTANCE_IDS[@]}" \
  --region $PRIMARY_REGION

echo "Instances stopping. Timer started: $(date +%H:%M:%S)"
```

> **Talking point:** "We've just stopped all worker nodes in us-east-1. Both applications are now unreachable. Let's see how each DR pattern responds."

Show both apps are down:

```bash
curl -sk --connect-timeout 3 https://${OADP_DOMAIN}/healthz || echo "OADP app: unreachable"
curl -sk --connect-timeout 3 https://${ACM_DOMAIN}/healthz || echo "ACM app: unreachable"
```

### Act 3: ACM Detects and Recovers (2 minutes)

> **Talking point:** "ACM monitors cluster health via klusterlet heartbeats. With our tuned lease, it detects failure in about 40 seconds."

Watch ACM detect the failure:

```bash
watch -n5 "echo '=== Cluster Health ===' && \
  oc get managedcluster -o custom-columns=\
'NAME:.metadata.name,AVAILABLE:.status.conditions[?(@.type==\"ManagedClusterConditionAvailable\")].status' && \
  echo '' && echo '=== Placement Decision ===' && \
  oc get placementdecision -n openshift-gitops \
    -l cluster.open-cluster-management.io/placement=acm-demo-placement \
    -o jsonpath='{range .items[0].status.decisions[*]}{.clusterName}{end}' && \
  echo '' && echo '' && echo '=== ArgoCD Applications ===' && \
  oc get applications.argoproj.io -n openshift-gitops"
```

Wait until the primary shows `Available: Unknown` and ArgoCD shows the primary application as `Degraded`. Press `Ctrl+C`.

> **Talking point:** "ACM detected the failure. ArgoCD shows the primary app as Degraded and the DR app as Healthy. The app is already running on the DR cluster. All we need is a DNS switch."

Switch DNS for the ACM app:

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"${ACM_DOMAIN}\",
        \"Type\": \"A\",
        \"TTL\": 30,
        \"ResourceRecords\": [{\"Value\": \"${DR_IP}\"}]
      }
    }]
  }"
```

Flush DNS and verify:

```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

curl -sk https://${ACM_DOMAIN}/healthz
echo ""
echo "ACM app recovered: $(date +%H:%M:%S)"
```

Refresh the ACM app browser tab to show it serving from the DR cluster.

> **Talking point:** "The ACM app is back. Total recovery time: about 90 seconds. No backup to restore, no volumes to recreate. The app was already running on both clusters."

### Act 4: OADP Recovers (5 minutes)

> **Talking point:** "The OADP app is still down. It runs on the primary cluster only, so we need to restore it from backup. Let me walk you through the manual recovery."

Log into the DR cluster:

```bash
oc login $(rosa describe cluster -c $DR_CLUSTER_NAME -o json | jq -r '.api.url') \
  --username cluster-admin --password '<password>'
```

Recover EFS volumes from the pre-recorded mapping:

```bash
export EFS_MAPPING_FILE=efs-pvc-map.csv
./scripts/recover-efs-volumes.sh
```

> **Talking point (while waiting):** "The script creates DR-side EFS access points that point to the replicated data paths, then creates static PersistentVolumes so the restored application mounts the replicated data instead of empty directories."

Restore the application from the Velero backup:

```bash
BACKUP_NAME=$(oc get backup -n openshift-adp \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1].metadata.name}')

export RESTORE_NAME="demo-restore-$(date +%Y%m%d-%H%M)"

cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: ${RESTORE_NAME}
  namespace: openshift-adp
spec:
  backupName: ${BACKUP_NAME}
  includedNamespaces:
    - dr-demo
  excludedResources:
    - pods
    - replicasets.apps
    - persistentvolumes
    - persistentvolumeclaims
  restorePVs: false
  existingResourcePolicy: update
EOF
```

Wait for the restore to complete:

```bash
watch "oc get restore $RESTORE_NAME -n openshift-adp \
  -o jsonpath='{.status.phase}' && echo '' && \
  oc get pods -n dr-demo"
```

Wait until the restore shows `Completed` and pods are `Running`. Press `Ctrl+C`.

Update the restored workload for the DR region:

```bash
oc annotate sa/s3-writer sa/dashboard -n dr-demo \
  eks.amazonaws.com/role-arn="$APP_S3_ROLE_ARN_DR" \
  --overwrite

oc set env deployment/telemetry-transmitter deployment/mission-control -n dr-demo \
  S3_BUCKET="$APP_BUCKET_DR" \
  AWS_REGION="$DR_REGION" \
  CLUSTER_NAME="$DR_CLUSTER_NAME" \
  AWS_ROLE_ARN="$APP_S3_ROLE_ARN_DR"
```

The OADP app uses Route 53 failover routing with health checks. The health check detects the primary is down and automatically switches DNS to the DR cluster. Verify the OADP app is serving from the DR cluster:

```bash
curl -sk https://${OADP_DOMAIN}/healthz
echo ""
echo "OADP app recovered: $(date +%H:%M:%S)"
```

Refresh the OADP app browser tab to show it serving from the DR cluster.

### Act 5: Recovery Summary (1 minute)

> **Talking point:** "Both apps are now running from us-west-2. Let's review how each pattern recovered."

| | OADP | ACM |
|---|---|---|
| Recovery time | ~5 minutes | ~90 seconds |
| Manual steps | EFS volume recovery, Velero restore, env var update | DNS switch only |
| Data continuity | EFS replication + S3 CRR, static PV mapping | EFS replication + S3 CRR, pre-staged PVs |
| Steady-state cost | Single cluster | Both clusters running |
| Deployment delay | Pods must start from scratch | App already running |
| Best for | Cost-sensitive, infrequent failover | Low RTO, fast failover |

> **Talking point:** "ACM eliminates the deployment delay because the application runs on both clusters. The trade-off is cost. If your RTO allows 5 minutes of recovery, OADP is more cost-effective. If you need sub-2-minute recovery, ACM with GitOps is the answer."

---

## Reset for Next Demo

Bring the primary cluster back, re-establish replication, and restore DNS.

Start the primary workers:

```bash
PRIMARY_INSTANCE_IDS=($(aws ec2 describe-instances --region ${PRIMARY_REGION} \
  --filters "Name=tag:api.openshift.com/name,Values=${PRIMARY_CLUSTER_NAME}" \
            "Name=instance-state-name,Values=stopped" \
  --query 'Reservations[*].Instances[*].InstanceId' --output text))

aws ec2 start-instances \
  --instance-ids "${PRIMARY_INSTANCE_IDS[@]}" \
  --region $PRIMARY_REGION
```

Wait for the primary cluster to rejoin ACM and for both ArgoCD applications to show `Healthy`. This takes 3-5 minutes:

```bash
oc login $(rosa describe cluster -c $CLUSTER_ACM -o json | jq -r '.api.url') \
  --username cluster-admin --password '<password>'

watch -n10 "oc get managedcluster -o custom-columns=\
'NAME:.metadata.name,AVAILABLE:.status.conditions[?(@.type==\"ManagedClusterConditionAvailable\")].status' && \
  echo '' && \
  oc get applications.argoproj.io -n openshift-gitops"
```

Clean up the OADP restore on the DR cluster (so the next demo starts clean):

```bash
oc login $(rosa describe cluster -c $DR_CLUSTER_NAME -o json | jq -r '.api.url') \
  --username cluster-admin --password '<password>'

oc delete namespace dr-demo --wait=false
oc delete pv -l app.kubernetes.io/managed-by=recover-efs-volumes
```

Re-establish EFS replication:

```bash
aws efs update-file-system-protection \
  --file-system-id $DR_EFS \
  --region $DR_REGION \
  --replication-overwrite-protection DISABLED

aws efs create-replication-configuration \
  --region $PRIMARY_REGION \
  --source-file-system-id $PRIMARY_EFS \
  --destinations "[{\"Region\": \"${DR_REGION}\", \"FileSystemId\": \"${DR_EFS}\"}]"
```

Switch DNS back to the primary:

```bash
PRIMARY_CONSOLE_HOST=$(rosa describe cluster -c $PRIMARY_CLUSTER_NAME -o json \
  | jq -r '.console.url' | sed 's|^https://||')
PRIMARY_IP=$(host $PRIMARY_CONSOLE_HOST | awk '/has address/{print $NF; exit}')

aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"${ACM_DOMAIN}\",
        \"Type\": \"A\",
        \"TTL\": 30,
        \"ResourceRecords\": [{\"Value\": \"${PRIMARY_IP}\"}]
      }
    }]
  }"
```

Log into the primary cluster and redeploy the OADP app:

```bash
oc login $(rosa describe cluster -c $PRIMARY_CLUSTER_NAME -o json | jq -r '.api.url') \
  --username cluster-admin --password '<password>'
./scripts/deploy-phoenix.sh
```

Take a fresh OADP backup and re-record the EFS mapping:

```bash
./scripts/record-efs-mapping.sh \
  --namespace dr-demo \
  --region "$PRIMARY_REGION" \
  --output efs-pvc-map.csv

export BACKUP_NAME="dr-demo-$(date +%Y%m%d-%H%M)"
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: ${BACKUP_NAME}
  namespace: openshift-adp
spec:
  includedNamespaces:
    - dr-demo
  excludedResources:
    - pods
    - replicasets.apps
    - persistentvolumes
    - persistentvolumeclaims
  storageLocation: dr-demo-dpa-1
  defaultVolumesToFsBackup: false
  snapshotVolumes: false
EOF

./scripts/create-dr-backup.sh
```

Verify both apps are accessible:

```bash
curl -sk https://${OADP_DOMAIN}/healthz
curl -sk https://${ACM_DOMAIN}/healthz
```

The environment is now ready for the next demo run.