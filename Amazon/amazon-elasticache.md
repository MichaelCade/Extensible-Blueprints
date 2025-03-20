# Amazon Elasticache (Redis)

# Create 

Create a Redis Cluster 

```
aws elasticache create-replication-group \
    --replication-group-id veeam-demo-cluster \
    --replication-group-description "Veeam-Elasticache-Demo" \
    --engine redis \
    --cache-node-type cache.t3.micro \
    --num-node-groups 1 \
    --replicas-per-node-group 1 \
    --automatic-failover-enabled \
    --region us-east-1

```


Verify with 
`aws elasticache describe-replication-groups`
`aws elasticache describe-replication-groups --replication-group-id veeam-demo-cluster`


# ConfigMap (amazon-elasticache-cm.yaml)
You will also need the following for your configmap 

```
aws elasticache describe-replication-groups \
    --query "ReplicationGroups[*].ReplicationGroupId" \
    --region us-east-1
```

Instead of the above you will need Cache Clusters 

```
aws elasticache describe-cache-clusters \
  --region us-east-1 \
  --query "CacheClusters[*].CacheClusterId" \
  --output text
```

In your configmap you will also need to provide Restored Replication group ID this is because a snapshot restore in Amazon ElastiCache does not overwrite the original cluster. Instead, it creates a new replication group using the snapshot as the data source.




```
kubectl create namespace elasticache-backup
```

```
kubectl --namespace kasten-io annotate configmap/elasticache-backup-config \
    kanister.kasten.io/blueprint=elasticache-backup-blueprint
```

create a Kasten K10 Policy to back up all resources in the elasticache-backup namespace.

Create Policy via Kasten UI
Open Kasten K10 UI (https://<your-kasten-ui>)
Navigate to Policies → Click Create New Policy
Set the Policy Name (e.g., elasticache-backup-policy)
Choose "Backup" as the action
Target the Namespace:
Click "Select Applications"
Use the Label Selector: 

`kubernetes.io/metadata.name=elasticache-backup`

Attach the Kanister Blueprint:
Scroll down to Application Configuration
Select elasticache-backup-blueprint
Define Schedule & Retention
Example: Run backup every 12 hours, keep last 7 snapshots
Click Create Policy 


`echo -n "your-access-key" | base64`
`echo -n "your-secret-key" | base64`



`kubectl apply -f elasticache-backup-config.yaml`
`kubectl apply -f elasticache-backup-secret.yaml`
`kubectl apply -f elasticache-backup-blueprint.yaml`


## Check snapshots from AWS CLI 

We can check the amount of snapshots we have on our cache cluster with the following command: 

```
aws elasticache describe-snapshots \
  --region us-east-1 \
  --cache-cluster-id veeam-demo-cluster-001 \
  --query "Snapshots | length(@)" --output text
```

We can get a list of the snapshots with: 

```
aws elasticache describe-snapshots \
  --region us-east-1 \
  --cache-cluster-id veeam-demo-cluster-001 \
  --query "Snapshots[*].SnapshotName" --output text
```

# Delete Snapshots 

For demo purposes we may wish to clean down all snapshots 

``` 
aws elasticache describe-snapshots \
  --region us-east-1 \
  --cache-cluster-id veeam-demo-cluster-001 \
  --query "Snapshots[].SnapshotName" \
  --output text | tr '\t' '\n' | while read snapshot_name; do
    echo "Deleting snapshot: $snapshot_name"
    aws elasticache delete-snapshot --region us-east-1 --snapshot-name "$snapshot_name"
done
```

## clean up 
We can then run the following when we have completed the demo, we need to remove our initial cluster and our restored clusters, if you have performed a restore then the original cluster will have been removed already. 

Delete all replication groups 
```
aws elasticache describe-replication-groups \
  --region us-east-1 \
  --query "ReplicationGroups[*].ReplicationGroupId" \
  --output text | xargs -I {} aws elasticache delete-replication-group \
  --region us-east-1 --replication-group-id {}
```

Delete any standalone clusters (restored ones)
```
aws elasticache describe-cache-clusters \
  --region us-east-1 \
  --query "CacheClusters[?ReplicationGroupId==null].CacheClusterId" \
  --output text | xargs -I {} aws elasticache delete-cache-cluster \
  --region us-east-1 --cache-cluster-id {}
```

## useful notes 
whilst these actions are taking place a good place to check the status of things that are possibly waiting, would be to check `kubectl get pods -n kasten-io` and then take the kanister-svc pod and get logs. `kubectl logs kanister-svc-5cfc4c6d66-d5nm9 -n kasten-io` 


manual aws command to restore and create new cluster from snapshot 

aws elasticache create-replication-group \
    --replication-group-id veeam-demo-cluster-restored \
    --replication-group-description "Restored ElastiCache replication group" \
    --snapshot-name snapshot-20250319143713 \
    --engine redis \
    --cache-node-type cache.t3.micro \
    --num-node-groups 1 \
    --replicas-per-node-group 1 \
    --automatic-failover-enabled \
    --region us-east-1