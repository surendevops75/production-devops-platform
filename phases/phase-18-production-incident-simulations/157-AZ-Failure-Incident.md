# 157 — Availability Zone (AZ) Failure Production Incident

> **Phase 18 — Incident Management**  
> **Incident #157**  
> **Environment:** Production  
> **Primary Region:** `ap-south-1`  
> **Architecture:** Multi-AZ EKS  
> **Example affected services:** `order-service`, `payment-service`, `inventory-service`

---

# 1. Objective

This chapter explains how to handle a complete **Availability Zone failure** in a production AWS/EKS environment.

An AZ failure is significantly larger than a single-node failure.

A node failure might look like:

```text
Node 3 failed
    ↓
Replace Node 3
    ↓
Reschedule Pods
```

An AZ failure can look like:

```text
AZ-2 unavailable
    ↓
Multiple EC2 nodes unavailable
    ↓
Multiple Pods unavailable
    ↓
Subnets unavailable
    ↓
Load-balancer targets change
    ↓
Applications lose capacity
    ↓
Database/cache/message workloads may be affected
```

The objective is therefore:

```text
Detect
  ↓
Confirm AZ impact
  ↓
Protect customer traffic
  ↓
Shift workloads
  ↓
Restore capacity in surviving AZs
  ↓
Protect RDS/Redis/Kafka/SQS
  ↓
Validate application
  ↓
Monitor
  ↓
Recover/failback
  ↓
RCA
```

---

# 2. Production Multi-AZ Architecture

Our target production architecture:

```text
                         AWS Region
                        ap-south-1
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
        AZ-1              AZ-2              AZ-3
          │                 │                 │
     ┌─────────┐       ┌─────────┐       ┌─────────┐
     │ Nodes   │       │ Nodes   │       │ Nodes   │
     └────┬────┘       └────┬────┘       └────┬────┘
          │                 │                 │
        Pods              Pods              Pods
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                         Services
                            │
                           ALB
                            │
                         Clients
```

Critical services should have replicas distributed across all three AZs.

Example:

```text
order-service:

AZ-1 → 2 Pods
AZ-2 → 2 Pods
AZ-3 → 2 Pods
```

If AZ-2 fails:

```text
AZ-1 → 2 Pods
AZ-2 → 0 Pods
AZ-3 → 2 Pods
```

Four replicas remain.

---

# 3. What Is an AZ Failure?

An AZ failure means that infrastructure or connectivity within one Availability Zone becomes unavailable or severely degraded.

Potentially affected:

- EC2
- EKS worker nodes
- subnets
- NAT Gateway
- EBS resources associated with the AZ
- load-balancer targets
- private connectivity
- application workloads
- regional dependencies using that AZ
- databases/caches if incorrectly designed

An AZ failure does **not** necessarily mean the entire AWS region has failed.

---

# 4. Node Failure vs AZ Failure

## Node Failure

```text
AZ-1
 ├── Node 1 ❌
 ├── Node 2 ✅
 └── Node 3 ✅
```

Most workloads can continue normally.

## AZ Failure

```text
AZ-1 ❌

Node 1 ❌
Node 2 ❌
Node 3 ❌
Subnet ❌
AZ-specific resources ❌
```

Multiple workloads can disappear simultaneously.

Therefore AZ failure requires capacity planning at the **AZ level**, not just node level.

---

# 5. Failure Scenario

Our production cluster:

```text
AZ-1:
  4 nodes

AZ-2:
  4 nodes

AZ-3:
  4 nodes
```

Total:

```text
12 nodes
```

Suppose AZ-2 fails.

Available capacity:

```text
AZ-1 = 4 nodes
AZ-2 = 0 nodes
AZ-3 = 4 nodes

Total = 8 nodes
```

We have immediately lost:

```text
4 / 12
= 33%
```

of node capacity.

If the cluster was already running at 75–80% utilization, the surviving AZs may not have enough capacity.

---

# 6. Important Production Principle

> **Multi-AZ availability requires spare capacity to survive the loss of an AZ.**

Having:

```text
3 AZs
```

does not automatically mean:

```text
survive one AZ failure
```

You need enough capacity in the remaining AZs.

---

# 7. AZ Failure Detection

Potential alerts:

```text
Multiple nodes NotReady
Multiple Pods unavailable
ALB target failures
EC2 instance failures concentrated in one AZ
Subnet/network failures
NAT connectivity failures
RDS events
Redis failover events
Kafka broker issues
```

A strong AZ alert should detect **correlated failures**, not just individual nodes.

For example:

```text
AZ-2:
10 nodes → 0 healthy
```

is much more important than:

```text
1 node NotReady
```

---

# 8. First Incident Question

Ask:

> **Are multiple failures correlated to the same Availability Zone?**

For example:

```text
Node 1 → AZ-2 → NotReady
Node 2 → AZ-2 → NotReady
Node 3 → AZ-2 → NotReady
Node 4 → AZ-2 → NotReady
```

This is a strong AZ-failure signal.

---

# 9. Check Nodes by AZ

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Example:

```text
NAME          STATUS     ZONE
node-1        Ready      ap-south-1a
node-2        Ready      ap-south-1a
node-3        NotReady   ap-south-1b
node-4        NotReady   ap-south-1b
node-5        NotReady   ap-south-1b
node-6        Ready      ap-south-1c
```

Now the pattern is obvious:

```text
ap-south-1b
```

is affected.

---

# 10. Check Pod Distribution

```bash
kubectl get pods \
  -A \
  -o wide
```

For a specific service:

```bash
kubectl get pods \
  -n ecommerce \
  -l app=order-service \
  -o wide
```

Example:

```text
NAME              NODE       STATUS
order-1           node-1     Running
order-2           node-3     Unknown
order-3           node-5     Running
order-4           node-6     Running
```

This tells us:

```text
AZ-1 → healthy
AZ-2 → failed
AZ-3 → healthy
```

---

# 11. Check EKS Node Capacity

After identifying the failed AZ:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Determine:

```text
Healthy nodes per AZ
CPU allocatable
Memory allocatable
Pod capacity
DaemonSet overhead
```

Example:

```text
AZ-1:
CPU available = 12 cores

AZ-3:
CPU available = 10 cores

Total surviving capacity = 22 cores
```

Compare this with:

```text
Required workload capacity
```

---

# 12. Capacity Calculation

Suppose before failure:

```text
Total required CPU = 30 cores
```

Surviving capacity:

```text
AZ-1 = 12 cores
AZ-3 = 10 cores

Total = 22 cores
```

We are short:

```text
30 - 22
= 8 cores
```

Therefore:

```text
Pods will remain Pending
```

until new capacity is created.

---

# 13. Karpenter During AZ Failure

If Karpenter is configured to use multiple AZs:

```text
Pending Pods
     ↓
Karpenter
     ↓
Evaluate available capacity
     ↓
Select healthy AZ
     ↓
Launch EC2
     ↓
Node joins EKS
     ↓
Pods scheduled
```

This is one reason multi-AZ subnet and NodePool configuration is important.

---

# 14. Karpenter Must Not Be AZ-Locked

Bad configuration:

```text
NodePool
   ↓
Only AZ-2
```

If AZ-2 fails:

```text
Karpenter
   ↓
Cannot create capacity
```

Better:

```text
NodePool
 ├── AZ-1
 ├── AZ-2
 └── AZ-3
```

with appropriate subnet/security-group discovery and scheduling constraints.

---

# 15. Check Pending Pods

```bash
kubectl get pods \
  -A \
  --field-selector=status.phase=Pending
```

Then:

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
Insufficient cpu
Insufficient memory
No matching node
Topology constraints
Node affinity
Taints
PVC
```

---

# 16. Topology Spread During AZ Failure

Example:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: order-service
```

This is useful during normal operation.

But during an AZ failure, strict:

```text
DoNotSchedule
```

constraints can sometimes prevent replacement Pods from being scheduled if the surviving AZs cannot satisfy the requested topology.

Therefore production teams must test:

```text
normal operation
+
node failure
+
AZ failure
```

and decide which constraints should be strict versus flexible.

---

# 17. PDB During AZ Failure

Suppose:

```yaml
minAvailable: 2
```

for a service with:

```text
3 replicas
```

An unexpected AZ failure can still cause:

```text
3 → 2 → 1
```

or even:

```text
3 → 0
```

depending on placement.

PDB does not magically protect against infrastructure loss.

The combination matters:

```text
Replicas
+
Topology spread
+
PDB
+
Multi-AZ nodes
+
Capacity headroom
```

---

# 18. ALB During AZ Failure

The ALB itself is regional and designed to operate across AZs when configured appropriately.

Our application path:

```text
Internet
   ↓
Route 53
   ↓
CloudFront/WAF
   ↓
ALB
   ↓
Healthy targets
   ↓
EKS Pods
```

If AZ-2 targets disappear:

```text
ALB
 ├── AZ-1 → healthy
 ├── AZ-2 → unhealthy/unavailable
 └── AZ-3 → healthy
```

Traffic should continue through surviving capacity.

However, if surviving Pods cannot handle the increased traffic:

```text
Latency ↑
5xx ↑
```

---

# 19. AZ Failure Creates Capacity Pressure

Suppose normal traffic:

```text
1,000 RPS
```

Before failure:

```text
AZ-1 → 333 RPS
AZ-2 → 333 RPS
AZ-3 → 334 RPS
```

AZ-2 fails.

Now surviving capacity must handle approximately:

```text
1,000 RPS
```

instead of:

```text
667 RPS
```

That is roughly:

```text
50%
```

more traffic per surviving capacity compared with the previous two-AZ load.

This is why headroom matters.

---

# 20. HPA During AZ Failure

HPA may detect:

```text
CPU ↑
Latency ↑
```

and increase replicas.

Example:

```text
order-service:
6 → 12 replicas
```

But if no physical capacity exists:

```text
12 desired
8 available
4 Pending
```

Therefore:

```text
HPA
```

must work together with:

```text
Karpenter / Node Groups
```

---

# 21. AZ Failure and Karpenter Feedback

Potential recovery:

```text
AZ failure
   ↓
Pods unavailable
   ↓
HPA increases replicas
   ↓
Pods Pending
   ↓
Karpenter launches nodes in surviving AZs
   ↓
Nodes Ready
   ↓
Pods scheduled
   ↓
Traffic stabilizes
```

This should be tested before production.

---

# 22. RDS During AZ Failure

RDS Multi-AZ is designed for high availability.

For example:

```text
RDS
 ├── Primary
 └── Standby
```

If an AZ hosting the primary becomes unavailable, RDS can perform a Multi-AZ failover according to the configured deployment architecture.

Important distinction:

```text
EKS AZ failure
```

does not automatically mean:

```text
RDS failure
```

and vice versa.

Investigate RDS independently.

---

# 23. RDS Dependency

Suppose:

```text
AZ-2 fails
```

EKS applications shift to:

```text
AZ-1
AZ-3
```

They may still connect to the RDS endpoint.

Applications should use:

```text
RDS DNS endpoint
```

not hardcoded IP addresses.

After failover, DNS continues to represent the current endpoint.

---

# 24. Redis During AZ Failure

For production ElastiCache:

```text
Primary
Replica
```

should be distributed appropriately across AZs where the chosen architecture supports it.

Possible behavior:

```text
AZ-2 failure
      ↓
Redis primary unavailable
      ↓
Managed failover
      ↓
Replica promoted
```

During the incident monitor:

```text
Redis latency
Connections
Errors
Failover status
Hit ratio
Evictions
```

Remember:

```text
Redis failure
    ↓
Cache misses
    ↓
RDS traffic
```

can create a secondary RDS incident.

---

# 25. Kafka During AZ Failure

For Kafka/MSK, broker distribution and replication matter.

Example:

```text
Broker 1 → AZ-1
Broker 2 → AZ-2
Broker 3 → AZ-3
```

If AZ-2 fails:

```text
Broker 2 unavailable
```

Kafka can continue if replication and ISR configuration provide sufficient redundancy.

Monitor:

```text
Under-replicated partitions
Offline partitions
ISR
Producer errors
Consumer lag
Broker health
```

An AZ incident can therefore become:

```text
AZ failure
 ↓
Kafka broker loss
 ↓
Replication changes
 ↓
Consumer/producer impact
```

---

# 26. SQS During AZ Failure

SQS is an AWS managed regional service.

The EKS consumer Pods may fail in one AZ:

```text
AZ-2 consumers ❌
```

but:

```text
AZ-1 consumers ✅
AZ-3 consumers ✅
```

The queue continues retaining messages.

This is one reason asynchronous architecture can improve resilience.

The remaining consumers can process the backlog.

---

# 27. S3 During AZ Failure

S3 is also an AWS regional managed service.

Application Pods can continue accessing S3 if:

- IAM works
- network path works
- DNS works
- required endpoints/NAT are available

Do not confuse:

```text
EKS AZ failure
```

with:

```text
S3 regional failure
```

They are separate failure domains.

---

# 28. NAT Gateway Consideration

Suppose each AZ has:

```text
Private subnet
     ↓
NAT Gateway
```

A good production design often places NAT gateways per AZ for resilience.

Example:

```text
AZ-1 → NAT-1
AZ-2 → NAT-2
AZ-3 → NAT-3
```

If AZ-2 fails:

```text
AZ-1 → NAT-1
AZ-3 → NAT-3
```

continue operating.

With one NAT Gateway only in AZ-2:

```text
AZ-2 fails
   ↓
Surviving AZs
   ↓
Cross-AZ NAT dependency
   ↓
Additional failure risk
```

---

# 29. VPC Endpoints

For AWS services such as S3 and selected AWS APIs, VPC endpoints can reduce dependency on NAT.

Examples may include:

```text
S3 Gateway Endpoint
ECR API
ECR DKR
STS
Secrets Manager
CloudWatch
```

The exact endpoint set should be based on actual workloads and architecture.

Do not blindly create every possible endpoint.

---

# 30. AZ Failure and DNS

Internal Kubernetes DNS:

```text
service.namespace.svc.cluster.local
```

continues through CoreDNS if the CoreDNS deployment has healthy replicas in surviving capacity.

Public DNS:

```text
Route 53
```

does not automatically fix an AZ failure inside the same region if the application still has insufficient surviving regional capacity.

Route 53 is more relevant when failing between:

```text
Primary Region
        ↓
DR Region
```

or using appropriate regional routing/failover architectures.

---

# 31. AZ Failure vs Region Failure

### AZ Failure

```text
ap-south-1
 ├── AZ-1 ✅
 ├── AZ-2 ❌
 └── AZ-3 ✅
```

### Region Failure

```text
ap-south-1 ❌

DR:
ap-southeast-1 ✅
```

An AZ failure should normally be handled within the region.

A region failure may require:

```text
Route 53
+
DR region
+
data recovery
```

---

# 32. Production Incident Procedure

Use:

```text
Alert
  ↓
Confirm correlated AZ failure
  ↓
Identify affected nodes
  ↓
Identify affected Pods
  ↓
Measure surviving capacity
  ↓
Check ALB
  ↓
Check HPA
  ↓
Check Karpenter
  ↓
Check RDS
  ↓
Check Redis
  ↓
Check Kafka/SQS
  ↓
Protect downstream systems
  ↓
Restore capacity
  ↓
Reschedule workloads
  ↓
Validate customer traffic
  ↓
Monitor
  ↓
RCA
```

---

# 33. Do Not Immediately Fail Over to DR Region

If only one AZ is unavailable:

```text
AZ-2 ❌
AZ-1 ✅
AZ-3 ✅
```

Do not automatically conclude:

```text
Go to ap-southeast-1
```

First determine whether the primary region can safely operate with the surviving AZs.

Regional failover introduces:

- data replication concerns
- DNS changes
- application activation
- secrets
- ECR
- RDS
- Redis
- Kafka/SQS
- operational complexity

DR region failover should be reserved for appropriate regional failure scenarios.

---

# 34. Capacity Planning for AZ Failure

Suppose:

```text
Normal required capacity = 60 CPU
```

Three AZs:

```text
AZ-1 = 30 CPU
AZ-2 = 30 CPU
AZ-3 = 30 CPU
```

Total:

```text
90 CPU
```

If AZ-2 fails:

```text
60 CPU remains
```

Exactly equal to normal demand.

There is:

```text
0 CPU headroom
```

That is risky.

A safer architecture might provide:

```text
AZ-1 = 35
AZ-2 = 35
AZ-3 = 35

Total = 105
```

After AZ-2 failure:

```text
70 CPU
```

Normal demand:

```text
60 CPU
```

Remaining headroom:

```text
10 CPU
```

The exact headroom target must come from capacity testing and business requirements.

---

# 35. Critical vs Noncritical Workloads

During AZ failure, prioritize:

### Tier 1

```text
auth-service
customer-service
product-service
cart-service
order-service
payment-service
inventory-service
```

### Tier 2

```text
notification-service
shipping-service
delivery-service
tax-service
invoice-service
```

### Tier 3

```text
reporting-service
recommendation-service
analytics/background workloads
```

The exact business classification should be agreed with application and business teams.

Noncritical workloads can sometimes be reduced to protect the critical customer path.

---

# 36. Graceful Degradation

During an AZ failure, not every feature must remain fully available.

For example:

```text
Recommendation service
       ↓
Temporarily unavailable
       ↓
Product page still works
```

Similarly:

```text
Reporting
```

may be delayed while:

```text
Ordering
Payment
Inventory
```

remain healthy.

This is better than allowing a noncritical workload to consume the capacity needed by the order path.

---

# 37. Example Capacity Protection

During an AZ incident:

```text
Scale down:
reporting-service
recommendation-service
large batch workloads
```

Protect:

```text
order-service
payment-service
inventory-service
auth-service
```

This is a deliberate incident-management decision.

---

# 38. Example RCA — AZ Failure

## Incident Summary

| Field | Details |
|---|---|
| Incident ID | `INC-2026-157` |
| Severity | SEV-1 |
| Date | 2026-10-04 |
| Duration | 13:10–14:02 IST |
| Affected AZ | `ap-south-1b` |
| Affected Services | Multiple ecommerce services |
| Root Cause | AZ infrastructure/network failure |
| Customer Impact | Increased latency and intermittent 5xx |
| Permanent Data Loss | No |

---

# 39. Customer Impact

During the incident:

```text
AZ-2
```

became unavailable.

Multiple EKS nodes were lost.

The surviving AZs initially handled customer traffic, but available capacity was insufficient for the complete workload.

Customers experienced:

- increased API latency
- intermittent HTTP 5xx
- delayed inventory updates
- delayed notifications

Critical payment data remained protected by application idempotency and database durability.

---

# 40. Timeline

```text
13:00  All three AZs healthy

13:10  Multiple nodes in ap-south-1b become unreachable

13:11  NodeNotReady alerts trigger

13:12  Multiple Pods unavailable

13:13  AZ correlation identified

13:15  SEV-1 declared

13:17  Surviving AZ capacity assessed

13:20  order-service and payment-service begin scaling

13:22  Several replacement Pods remain Pending

13:24  Karpenter begins provisioning additional nodes
       in surviving AZs

13:28  New nodes begin joining EKS

13:31  Critical application Pods become Ready

13:34  ALB target health improves

13:37  RDS connection count increases
       due to workload concentration

13:39  Noncritical workloads reduced

13:42  RDS pressure stabilizes

13:45  Kafka lag begins recovering

13:50  Customer API latency returns toward baseline

13:56  Inventory processing catches up

14:02  Incident resolved
```

---

# 41. Investigation

### Kubernetes

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Result:

```text
AZ-1 → Ready
AZ-2 → NotReady
AZ-3 → Ready
```

Multiple failed nodes were concentrated in:

```text
ap-south-1b
```

---

# 42. Capacity Evidence

Before failure:

```text
Required:
60 CPU

Available:
90 CPU
```

After AZ-2 failure:

```text
Available:
60 CPU
```

There was effectively no spare capacity.

As HPA attempted to recover workloads:

```text
Pods Pending
```

until Karpenter launched additional nodes.

---

# 43. Secondary Impact

The AZ failure caused:

```text
Workloads concentrated into AZ-1/AZ-3
```

This increased:

```text
RDS connections
CPU
network traffic
Kafka consumer load
```

RDS connection count:

```text
125 → 290
```

Kafka lag:

```text
2,000 → 80,000
```

Therefore the incident became:

```text
AZ failure
    ↓
EKS capacity loss
    ↓
Pod loss
    ↓
Traffic concentration
    ↓
RDS/Kafka pressure
    ↓
Latency
```

---

# 44. Root Cause

The direct root cause was an infrastructure failure affecting:

```text
ap-south-1b
```

The customer impact was amplified because the production cluster did not have sufficient capacity headroom to lose one complete AZ while continuing to run the normal workload.

---

# 45. Amplification Mechanism

```text
AZ-2 failure
      ↓
33% node capacity lost
      ↓
Pods rescheduled
      ↓
Surviving AZs reach high utilization
      ↓
HPA scales
      ↓
Pods Pending
      ↓
Karpenter provisions new nodes
      ↓
Traffic concentrates
      ↓
RDS connections increase
      ↓
Kafka consumers slow
      ↓
Latency/5xx increase
```

---

# 46. Contributing Factors

1. AZ-level infrastructure failure.
2. Insufficient spare capacity after losing one AZ.
3. Some workloads had less-than-ideal AZ distribution.
4. HPA scaling increased pressure on RDS.
5. Background workloads competed with critical workloads.
6. Karpenter provisioning added recovery time.
7. Capacity planning focused on normal traffic rather than AZ-loss traffic.

---

# 47. What Went Well

- Multi-AZ architecture prevented complete regional outage.
- AZ-1 and AZ-3 remained operational.
- Kubernetes automatically recreated Pods.
- Karpenter provided additional capacity.
- RDS remained available.
- Kafka retained events.
- SQS retained messages.
- Critical transactions remained durable.
- Observability allowed correlation across EKS, RDS and Kafka.

---

# 48. What Went Wrong

- Remaining AZs initially lacked sufficient spare capacity.
- Some Pods were not ideally distributed.
- Noncritical workloads competed for capacity.
- Database connection pressure increased.
- Kafka lag increased during recovery.

---

# 49. Immediate Mitigation

```text
1. Confirm AZ failure
2. Stop unnecessary deployments
3. Protect critical services
4. Reduce noncritical workloads
5. Increase surviving capacity
6. Allow Karpenter to provision nodes
7. Monitor RDS connections
8. Monitor Kafka lag
9. Verify ALB targets
10. Validate order/payment/inventory
```

---

# 50. Corrective Actions

| Action | Owner |
|---|---|
| Add formal AZ-loss capacity target | Platform |
| Maintain spare capacity for one-AZ loss | Platform |
| Improve topology spread | Platform |
| Review Karpenter multi-AZ configuration | Platform |
| Add AZ-level dashboards | SRE |
| Add correlated AZ failure alert | SRE |
| Create critical/noncritical workload priorities | Architecture |
| Protect DB connection budget during AZ failure | Platform + DBA |
| Add AZ failure load test | SRE |
| Run quarterly AZ game day | SRE |

---

# 51. Five Whys

### Why did customers experience latency and 5xx?

Application capacity dropped after one AZ became unavailable.

### Why did capacity drop significantly?

A complete AZ worth of nodes and Pods was lost.

### Why couldn't the surviving AZs immediately absorb the workloads?

There was insufficient spare capacity.

### Why was spare capacity insufficient?

Normal capacity planning did not include sufficient headroom for complete AZ loss.

### Why was AZ-loss capacity not included?

Capacity planning focused on normal/peak traffic rather than infrastructure failure scenarios.

---

# 52. Final Root Cause Statement

> The incident was caused by an Availability Zone infrastructure failure that removed approximately one-third of the EKS worker capacity. The customer impact was amplified because the surviving AZs did not have sufficient spare capacity to immediately absorb the failed AZ's workloads, resulting in temporary Pod shortages, database pressure, Kafka lag and increased application latency.

---

# 53. Improved Architecture

Before:

```text
AZ-1       AZ-2       AZ-3

30 CPU     30 CPU     30 CPU
```

Normal:

```text
60 CPU required
```

After AZ failure:

```text
AZ-1       AZ-2       AZ-3

30 CPU       X        30 CPU

60 CPU available
60 CPU required
```

No headroom.

Improved:

```text
AZ-1       AZ-2       AZ-3

35 CPU     35 CPU     35 CPU
```

After AZ failure:

```text
35 + 35
= 70 CPU
```

Normal:

```text
60 CPU
```

Remaining:

```text
10 CPU headroom
```

Again, actual numbers must come from real capacity testing.

---

# 54. AZ Failure Runbook

Create:

```text
runbooks/az-failure.md
```

Recommended procedure:

```text
1. Confirm correlated AZ failure
2. Identify affected nodes
3. Identify affected Pods
4. Check surviving capacity
5. Check Pending Pods
6. Check HPA
7. Check Karpenter
8. Check ALB
9. Check RDS
10. Check Redis
11. Check Kafka
12. Check SQS
13. Protect critical workloads
14. Reduce noncritical workloads
15. Provision capacity
16. Validate applications
17. Monitor dependencies
18. Measure recovery time
19. Restore normal capacity
20. RCA
```

---

# 55. Production Checklist

### AZ

- [ ] Failed AZ confirmed
- [ ] AWS infrastructure status checked
- [ ] Nodes in failed AZ identified
- [ ] Subnets identified
- [ ] AZ-specific dependencies checked

### EKS

- [ ] Remaining nodes checked
- [ ] Pending Pods checked
- [ ] HPA checked
- [ ] Karpenter checked
- [ ] Topology spread checked
- [ ] PDB checked

### Application

- [ ] ALB targets healthy
- [ ] Critical services healthy
- [ ] 5xx checked
- [ ] Latency checked
- [ ] Business transactions validated

### Dependencies

- [ ] RDS checked
- [ ] Redis checked
- [ ] Kafka checked
- [ ] SQS checked
- [ ] S3 checked
- [ ] Secrets/KMS checked

### Recovery

- [ ] Replacement capacity available
- [ ] Pods rescheduled
- [ ] Capacity headroom restored
- [ ] Kafka lag drained
- [ ] SQS backlog drained
- [ ] RDS connections normalized

---

# 56. Important Commands

### Nodes by AZ

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

### Pods

```bash
kubectl get pods \
  -A \
  -o wide
```

### Pending Pods

```bash
kubectl get pods \
  -A \
  --field-selector=status.phase=Pending
```

### Node details

```bash
kubectl describe node <node>
```

### PDB

```bash
kubectl get pdb -A
```

### HPA

```bash
kubectl get hpa -A
```

### Karpenter

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter
```

### Services

```bash
kubectl get svc -A
```

### EndpointSlices

```bash
kubectl get endpointslice -A
```

---

# 57. Interview Answer

If asked:

> **"What happens if an entire AWS Availability Zone fails in your EKS production environment?"**

Answer:

> "First I would confirm that the failures are correlated to one AZ by checking node status and the topology zone label. Then I would identify the Pods lost in that AZ and calculate the capacity remaining in the surviving AZs. I would check Pending Pods, HPA, Karpenter or managed node-group scaling, topology constraints and PDBs. At the same time I would monitor ALB health and downstream systems such as RDS, Redis and Kafka because traffic concentration can create secondary bottlenecks. I would prioritize critical services such as authentication, ordering, payment and inventory, and reduce noncritical workloads if necessary. Once replacement capacity is available, I would verify Pod readiness, ALB targets, application latency and real business transactions. Finally I would perform an RCA covering both the AZ failure and the capacity or architecture gaps that amplified the impact."

---

# 58. Key Production Lessons

### Lesson 1

> **Three AZs do not automatically guarantee survival of one AZ failure.**

### Lesson 2

> **You need enough capacity in the remaining AZs to absorb the failed AZ.**

### Lesson 3

> **AZ-level capacity planning is different from node-level capacity planning.**

### Lesson 4

> **Traffic concentration can overload RDS, Redis and Kafka after an AZ failure.**

### Lesson 5

> **HPA cannot create physical infrastructure capacity.**

### Lesson 6

> **Karpenter/node groups must be capable of provisioning in surviving AZs.**

### Lesson 7

> **Critical workloads need topology distribution.**

### Lesson 8

> **Noncritical workloads should be degradable during capacity emergencies.**

### Lesson 9

> **An AZ failure should not automatically trigger regional DR.**

### Lesson 10

> **The real HA test is not "Do we have three AZs?" but "Can we continue serving customers after losing one AZ?"**

---

# 59. Final Architecture

```text
                         AWS REGION
                        ap-south-1
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
      AZ-1                 AZ-2                AZ-3
        │                   │                   │
   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │ Nodes   │         │ Nodes   │         │ Nodes   │
   └────┬────┘         └────┬────┘         └────┬────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                    Critical Workloads
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
       Topology             PDB          Capacity
        Spread                            Headroom
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                         Karpenter
                            │
                 ┌──────────┴──────────┐
                 │                     │
             Surviving AZ-1        Surviving AZ-3
                 │                     │
                 └──────────┬──────────┘
                            │
                      Pods Recover
                            │
                    ┌───────┼────────┐
                    ▼       ▼        ▼
                   ALB     RDS      Kafka
                    │       │        │
                    └───────┼────────┘
                            ▼
                    Customer Traffic
```

---

## Phase 18 Progress

```text
143 CrashLoopBackOff              ✅
144 OOMKilled                     ✅
145 CPU Saturation                ✅
146 Memory Saturation             ✅
147 Disk Full                     ✅
148 ImagePullBackOff              ✅
149 ALB 5xx                       ✅
150 Service Connectivity Failure  ✅
151 DNS Failure                   ✅
152 RDS Connection Exhaustion     ✅
153 Redis Failure                 ✅
154 SQS Backlog                   ✅
155 Kafka Consumer Lag            ✅
156 Node Failure                  ✅
157 AZ Failure                    ✅
```

**Next: `158 — ArgoCD Drift Production Incident`**