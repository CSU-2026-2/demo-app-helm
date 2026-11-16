# Shared Strimzi Kafka

Managed by Argo CD Application `kafka-cluster`, namespace `kafka`.
The Strimzi operator and Kafbat UI are separate upstream Helm applications;
their bootstrap declarations are in `../kubepractice/kafka/` outside this repository.

- Kafka 4.3.2; Strimzi operator 1.3.1.
- Three combined controller/broker nodes, one per Kubernetes worker.
- Bootstrap: `shared-kafka-bootstrap.kafka.svc.cluster.local:9092`.
- Compatibility bootstrap alias: `kafka.kafka.svc.cluster.local:9092` (ExternalName Service).
- Internal PLAINTEXT listener accessible across namespaces; no external broker port.
- Topic `todos-created`: one partition, replication factor 3, minimum ISR 2.
- Default replication factor 3 and minimum ISR 2 for newly created topics.
- Kafbat UI: https://kafka.kubepractice.ru, login required, credentials in Kubernetes
  Secret `kafbat-auth`; no passwords are stored in this repository.

## Storage

`storage.yaml` declares three retained 10Gi local PVs and StorageClass `kafka-local`.
Each PV uses a dedicated directory `/var/lib/kafka-strimzi/shared` on its worker;
the directory is prepared by the administrator before initial deployment.
PVC names are `data-0-shared-dual-role-{0,1,2}`. Disks survive Pod recreation.
PV node affinity and required Pod anti-affinity place each replica on a different worker.
Deleting a worker destroys its local disk; recover through the surviving replicas
and explicitly replace the failed PV. These are worker-local disks, not shared CSI storage.
PV capacity is declared to Kubernetes; it is not a filesystem quota.
PV pruning is disabled and PVC `deleteClaim` is false to retain data on manifest removal.

## Verify

```sh
kubectl get kafka,kafkanodepool,kafkatopic -n kafka
kubectl get pods -n kafka -o wide
kubectl get pvc -n kafka
kubectl get applications -n argocd
```

All three replicas must be in sync before maintenance. Producers should use `acks=all`.
The three controller voters tolerate one unavailable voter; data writes require at least
two in-sync replicas. PodDisruptionBudget allows at most one voluntary disruption.
