# ⭐ Storage — PV, PVC, StorageClass, StatefulSets

---

## Q1: Explain PersistentVolume, PersistentVolumeClaim, and StorageClass. How do they relate?

**Senior Answer:**

These three work together to decouple storage provisioning from storage consumption:

**PersistentVolume (PV)** — the actual storage. A piece of storage provisioned by an admin or automatically. It exists cluster-wide (not namespace-scoped). It could be an AWS EBS volume, a GCP Persistent Disk, an NFS mount, or a local disk.

**PersistentVolumeClaim (PVC)** — a request for storage by a user/pod. "I need 10Gi of ReadWriteOnce storage." K8s finds a PV that satisfies the claim and binds them. The pod then mounts the PVC — it doesn't care where the actual storage lives.

**StorageClass** — defines how to dynamically provision PVs. When a PVC is created and no existing PV matches, the StorageClass's provisioner creates one automatically. It defines storage type, performance tier, replication policy, etc.

```
Pod → mounts PVC
          ↓
     PVC (bound to)
          ↓
     PV (backed by)
          ↓
     Actual Storage (EBS, NFS, etc.)
```

**Example flow with dynamic provisioning:**
```yaml
# PVC — what the pod asks for
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-storage
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd    # Which StorageClass to use
  resources:
    requests:
      storage: 50Gi
```

When applied:
1. K8s sees no existing PV matches
2. `fast-ssd` StorageClass provisioner creates an AWS EBS gp3 volume
3. A PV is created and bound to this PVC
4. Pod mounts the PVC — it gets 50Gi on EBS

**Access modes:**
- `ReadWriteOnce` — mounted read-write by one node at a time (most databases)
- `ReadOnlyMany` — mounted read-only by many nodes (shared config files)
- `ReadWriteMany` — mounted read-write by many nodes simultaneously (NFS, EFS — needed for shared uploads)

---

## Q2: What happens to a PVC when its pod is deleted? What about when the PV is deleted?

**Senior Answer:**

This is controlled by the **Reclaim Policy** on the PV (or StorageClass):

**Retain:** PV is not deleted when PVC is deleted. Data persists. Manual cleanup required. Use for critical databases — you don't want data deleted accidentally.

**Delete:** PV (and the backing storage) is automatically deleted when the PVC is deleted. Convenient but dangerous for production databases. Fine for ephemeral test data.

**Recycle (deprecated):** Basic scrub (rm -rf) and make available again. Don't use.

**The scenario that trips people up in interviews:**
```
1. Pod is deleted → PVC remains (PVCs persist independently of pods)
2. PVC is deleted → PV status changes to "Released" (or is deleted if policy=Delete)
3. A Released PV with Retain policy is NOT automatically available to new PVCs
   → You must manually clear the claimRef before it can be rebound
```

**StatefulSets and PVCs:**
StatefulSets create PVCs automatically via `volumeClaimTemplates`. When a StatefulSet pod is deleted and recreated (even on a different node), it reattaches to its original PVC — `db-0` always gets `data-db-0`. This is the statefulness guarantee. When you delete a StatefulSet, the PVCs are NOT deleted — you must delete them manually. This protects your data.

---

## Q3: When would you NOT run a database on Kubernetes?

**Senior Answer:**

This is an opinion question — I'd give an honest, nuanced answer:

**Arguments for running DB on K8s:**
- Consistent platform — one tool for everything
- K8s can provide self-healing (pod restarts)
- StatefulSets handle stable network identity
- Useful for development/staging environments

**Arguments against (why most production databases run outside K8s):**

1. **Storage complexity:** Database performance is highly sensitive to storage I/O. K8s storage provisioning introduces abstraction layers that can increase latency. Raw EBS volumes attached directly to EC2 are simpler and often faster than EBS backed PVCs.

2. **StatefulSet failure behavior:** As we discussed — when a node fails, StatefulSet pods don't automatically reschedule. For a database, this means manual intervention during node failures. Managed databases (RDS, CloudSQL) handle this transparently.

3. **Backup complexity:** Database backups need point-in-time consistency, often requiring the DB to be quiesced. This is tricky in K8s. Managed services have built-in backup mechanisms.

4. **Upgrade complexity:** Upgrading a clustered database (PostgreSQL HA, MySQL InnoDB Cluster) running on K8s requires careful coordination that's easy to get wrong.

5. **Operational expertise:** Running databases on K8s requires both K8s expertise AND database expertise. Most teams have one or the other, rarely both.

**My honest recommendation:** Use managed cloud databases (RDS, CloudSQL, PlanetScale, Neon) for production databases. Run K8s for stateless services. Running K8s for databases is reasonable if you have a dedicated platform team with DBA experience, but it's a significant operational investment.

**Exception:** In-cluster caches (Redis for session storage), message queues that can tolerate some data loss (development Kafka), and development databases are totally fine on K8s.

---

## Q4: What is the difference between emptyDir and a PersistentVolume?

**Senior Answer:**

**emptyDir:**
- Temporary storage created when a pod starts, deleted when the pod is terminated
- Stored on the node's local disk (or memory if `medium: Memory`)
- Shared between all containers in the same pod — great for sidecar patterns
- Use cases: temporary cache, sharing files between app container and logging sidecar, scratch space for batch processing

```yaml
volumes:
- name: tmp-cache
  emptyDir:
    medium: Memory    # tmpfs — very fast, counts against container memory limit
    sizeLimit: 512Mi
```

**PersistentVolume:**
- Survives pod restarts and rescheduling
- Data exists independently of the pod lifecycle
- Backed by durable storage (EBS, NFS, GCP PD)
- Use cases: databases, user uploads, stateful application data

**The subtle use cases for emptyDir:**
- Sharing a Unix socket between containers in a pod (e.g., app container + Envoy proxy sharing `/var/run/envoy.sock`)
- Generating config files in an initContainer and passing them to the main container
- Temporary files during a data processing job where the result goes elsewhere (S3, DB)

**What interviewers check:** Do you know that emptyDir data is lost on pod restart? And that containers within the same pod share emptyDir volumes but pods on different nodes do not?
