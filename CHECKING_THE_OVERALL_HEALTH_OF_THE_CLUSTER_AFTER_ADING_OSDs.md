**Environment:** Wahda Ceph production cluster  
**Execution host:** `thewahda4`(can be admin nodein your case)  
**Execution user:** `root`  
**Purpose:** Perform read-only checks after the Ceph OSD expansion.

> Run these commands from `thewahda4` as root. These checks are intended for verification; do not make configuration or recovery changes as part of this checklist.

---

## 1. Essential production health checks

### A. Detailed health warnings

```bash
ceph health detail
```

**Why:** Shows detailed warnings and errors that may not be obvious from the short `ceph -s` output.

**Expected:** No outstanding health warnings.

### B. OSD distribution and status

```bash
ceph osd tree
```

**Why:** Confirms which OSDs belong to which hosts and whether they are `up` and `in`.

**Expected host-to-OSD mapping:**

- `thewahda4`: OSD.0 and OSD.3
- `thewahda5`: OSD.1 and OSD.4
- `thewahda6`: OSD.2 and OSD.5

### C. OSD utilization

```bash
ceph osd df tree
```

**Why:** Shows the utilization and capacity of individual OSDs and hosts.

**Check for:**

- Uneven utilization.
- An OSD approaching its capacity limit.
- Unexpected differences between OSD weights and actual utilization.

### D. Storage capacity

```bash
ceph df
```

**Why:** Gives pool-level and overall cluster capacity information.

**Check:** Compare the available capacity with the `ceph -s` output.

### E. OSD performance

```bash
ceph osd perf
```

**Why:** Displays OSD commit and apply latency.

This helps identify OSDs experiencing unusually high disk-operation latency. Compare values across OSDs rather than judging one number without workload context.

---

## 2. Check whether all services are running

### A. Orchestrator daemon status

```bash
ceph orch ps
```

**Why:** Lists daemons managed by the Ceph Orchestrator, including MON, MGR, OSD and RGW.

**Check:** Confirm that the expected daemons are running on the appropriate hosts.

### B. Host inventory

```bash
ceph orch host ls
```

**Why:** Confirms that all three hosts are registered and available to the orchestrator.

### C. Ceph version consistency

```bash
ceph versions
```

**Why:** Helps identify whether the cluster daemons are running consistent Ceph versions.

---

## 3. Check pool configuration and PG autoscaling

### A. Pool autoscaler status

```bash
ceph osd pool autoscale-status
```

**Why:** Shows the autoscaler status and recommendations for individual pools.

### B. Pool configuration

```bash
ceph osd pool ls detail
```

**Why:** Displays pool configuration, including replication size, minimum size, PG settings and CRUSH rules.

These checks matter because a cluster can be `HEALTH_OK` while its pool configuration still deserves operational review.

---

## 4. Additional PG checks

### A. PG status

```bash
ceph pg stat
```

**Why:** Gives a focused view of PG states, misplaced objects and recovery activity.

**Expected for the current cluster:**

```text
385 pgs: 385 active+clean
```

### B. Stuck PGs

```bash
ceph pg dump_stuck
```

**Expected:** No stuck PGs.

---

## 5. Production verification checklist

Use this checklist to track the post-expansion verification.

- [ ] **Cluster health** — `ceph health detail`
- [ ] **OSD distribution** — `ceph osd tree`
- [ ] **OSD utilization** — `ceph osd df tree`
- [ ] **Storage capacity** — `ceph df`
- [ ] **OSD latency** — `ceph osd perf`
- [ ] **Daemon status** — `ceph orch ps`
- [ ] **Host inventory** — `ceph orch host ls`
- [ ] **Ceph versions** — `ceph versions`
- [ ] **Pool autoscaling** — `ceph osd pool autoscale-status`
- [ ] **Pool configuration** — `ceph osd pool ls detail`
- [ ] **PG status** — `ceph pg stat`
- [ ] **Stuck PGs** — `ceph pg dump_stuck`

**Progress:** 0/12 checks completed.

---

## Important production note

`ceph -s` provides a high-level cluster summary, but it does not replace detailed checks of OSD utilization, latency, daemon placement, pool configuration and PG state.

The commands in this document are intended for inspection. If a check reveals an unexpected condition, review it with the responsible senior or Ceph administrator before making changes. Do not run destructive disk commands or modify OSD, pool, CRUSH or recovery settings merely as part of verification.
