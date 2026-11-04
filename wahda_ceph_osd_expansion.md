# Adding New Disks to an Existing Ceph Cluster as OSDs

## 1. Task background

A few days ago, we added one new 1.9 TB-class disk to each of our three
Wahda server nodes. Today, our senior asked us to add those disks to the
existing Ceph cluster as new OSDs.

Our goal was to add the new disks without disturbing the existing OSDs,
then verify that Ceph had deployed them successfully and was
redistributing data.

## 2. Existing Wahda cluster

We performed the task from `thewahda4`, which has the `_admin` label and
the Ceph administrative configuration.

  Host          Existing OSD   Existing disk   New disk
  ------------- -------------- --------------- ------------
  `thewahda4`   `osd.0`        `/dev/sda`      `/dev/sdc`
  `thewahda5`   `osd.1`        `/dev/sda`      `/dev/sdc`
  `thewahda6`   `osd.2`        `/dev/sda`      `/dev/sdc`

The cluster was running Ceph 19.2.3 and was managed through the Ceph
Orchestrator.

> **Important:** Verify the device on each host. Never assume `/dev/sdc`
> refers to the intended new disk on every server.

## 3. Precautions before adding a disk

1.  Confirm that the disk is attached to the correct server.
2.  Confirm its size, model and serial number using `lsblk`.
3.  Make sure it is not the operating-system disk or an existing Ceph
    OSD disk.
4.  Confirm that it is empty and contains no data that must be
    preserved.
5.  Inspect filesystem signatures without modifying the disk.
6.  Check current Ceph health and OSD status.
7.  Preserve the existing OSD device entry in the YAML file.

Do not use destructive commands such as `wipefs -a`, `mkfs`, `pvcreate`,
`sgdisk` or disk-zap commands during inspection. Do not manually format
the disk before the Orchestrator provisions it.

## 4. Step 1 --- Check the cluster

Run these commands from `thewahda4`, the Ceph admin node.

### Overall cluster health

``` bash
ceph -s
```

Shows cluster health, MON/MGR/RGW/OSD status, pools, PGs, capacity,
client I/O and recovery activity.

### Existing OSDs and host placement

``` bash
ceph osd tree
```

Shows OSD IDs, their hosts, status (`up`/`down`), membership
(`in`/`out`) and weights.

### OSD service specifications

``` bash
ceph orch ls --service_type osd
```

Shows OSD services managed by the Orchestrator and their placement.

### OSD daemon status

``` bash
ceph orch ps --daemon_type osd
```

Shows where OSD daemons are running and their runtime status.

## 5. Step 2 --- Verify the new disk on every host

Run these checks on `thewahda4`, `thewahda5` and `thewahda6`.

### List disks

``` bash
lsblk
```

For a focused view:

``` bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,TYPE
```

This helps identify the new disk by device name, capacity, model and
serial number.

### Inspect signatures without erasing them

After confirming the correct device, for example `/dev/sdc`:

``` bash
wipefs -n /dev/sdc
```

The `-n` option performs a no-action inspection. It displays signatures
if present; it does not erase them.

If the disk has unexpected partitions, signatures, LVM data or other
contents, stop and investigate before proceeding.

## 6. Step 3 --- Inspect the existing YAML files

On `thewahda4`, list the OSD files:

``` bash
ls -l *_osd.yml
```

Read the three specifications:

``` bash
cat thewahda4_osd.yml
cat thewahda5_osd.yml
cat thewahda6_osd.yml
```

The original `thewahda4_osd.yml` looked like this:

``` yaml
service_type: osd
service_id: thewahda4_osd
placement:
  host_pattern: 'thewahda4'
spec:
  data_devices:
    paths:
    - /dev/sda
```

What the fields mean:

-   `service_type: osd` --- defines an OSD service.
-   `service_id` --- identifies this OSD service specification.
-   `host_pattern` --- selects the host where the service is placed.
-   `data_devices.paths` --- lists the device path or paths selected for
    OSD provisioning.

## 7. Step 4 --- Update the YAML files

We preserved the existing `/dev/sda` entry and added the new `/dev/sdc`
entry.

### `thewahda4_osd.yml`

``` yaml
service_type: osd
service_id: thewahda4_osd
placement:
  host_pattern: 'thewahda4'
spec:
  data_devices:
    paths:
    - /dev/sda
    - /dev/sdc
```

### `thewahda5_osd.yml`

``` yaml
service_type: osd
service_id: thewahda5_osd
placement:
  host_pattern: 'thewahda5'
spec:
  data_devices:
    paths:
    - /dev/sda
    - /dev/sdc
```

### `thewahda6_osd.yml`

``` yaml
service_type: osd
service_id: thewahda6_osd
placement:
  host_pattern: 'thewahda6'
spec:
  data_devices:
    paths:
    - /dev/sda
    - /dev/sdc
```

Edit a file using `nano`:

``` bash
nano thewahda4_osd.yml
```

Save with `Ctrl+O`, press `Enter`, then exit with `Ctrl+X`. Repeat for
the other two files.

Verify the contents:

``` bash
cat thewahda4_osd.yml
cat thewahda5_osd.yml
cat thewahda6_osd.yml
```

Before applying, confirm that each file has the correct hostname and
device paths.

> **Production precaution:** Applying a changed OSD specification can
> cause the Orchestrator to reconcile the service and provision eligible
> matching devices. Confirm that it targets only the intended new disk
> and that the existing disk entry is correct. Follow the approved
> cluster procedure.

## 8. Step 5 --- Apply the YAML specifications

We ran the Orchestrator commands from `thewahda4`, the admin node. We
did not need to SSH into each target host to apply its YAML file.

The command format is:

``` bash
ceph orch apply -i <filename>
```

We applied the specifications one at a time:

### Add the OSD on `thewahda4`

``` bash
ceph orch apply -i thewahda4_osd.yml
```

### Add the OSD on `thewahda5`

``` bash
ceph orch apply -i thewahda5_osd.yml
```

### Add the OSD on `thewahda6`

``` bash
ceph orch apply -i thewahda6_osd.yml
```

The Orchestrator provisions the matching device and deploys the OSD
daemon on the host selected in the YAML.

### When to apply the next specification

Our senior advised that once the newly created OSD is `up` and `in`, we
can proceed to the next host, provided the cluster remains healthy and
there is no unexpected deployment or recovery problem.

We do not have to wait for every PG to become `active+clean` between
each OSD addition. Recovery can continue in the background.

Apply one specification, verify the new OSD, and then proceed according
to the senior's deployment procedure.

## 9. Step 6 --- Verify that the new OSD joined

After applying a specification, run these commands from `thewahda4`.

### Check OSD placement and status

``` bash
ceph osd tree
```

Verify that the new OSD appears under the intended host and is `up` and
`in`.

### Check OSD daemon status

``` bash
ceph orch ps --daemon_type osd
```

Verify that the new OSD daemon appears on the intended host and has a
running status.

### Check cluster status

``` bash
ceph -s
```

Verify that the OSD count has increased, the new OSDs are up/in, and no
unexpected health error has appeared. Recovery activity can be monitored
here too.

## 10. Step 7 --- Monitor redistribution and recovery

After new OSDs join, Ceph recalculates data placement using CRUSH. PGs
can be remapped, and Ceph moves objects through recovery and backfill so
that data reaches its expected locations.

### Overall status

``` bash
ceph -s
```

Shows health, OSD membership, PG states, misplaced objects, client I/O
and recovery throughput.

For continuous monitoring:

``` bash
watch -n 5 ceph -s
```

Press `Ctrl+C` to stop watching.

### PG summary

``` bash
ceph pg stat
```

Shows the total PG count, PG states, misplaced objects and recovery
activity.

### OSD capacity and utilization

``` bash
ceph osd df tree
```

Shows capacity and utilization in the OSD host hierarchy. Use it to
check whether the new OSDs are receiving data and how utilization is
distributed.

### Recovery progress

``` bash
ceph progress
```

Shows reported progress events, including the global recovery event when
available. The estimated remaining time can change; it is not a
guaranteed completion time.

### Health details

``` bash
ceph health detail
```

Shows details behind health warnings or errors.

## 11. Understanding important PG states

  -----------------------------------------------------------------------
  PG state                            Meaning
  ----------------------------------- -----------------------------------
  `active+clean`                      PG is active and its required
                                      replicas are synchronized.

  `active+remapped`                   Expected OSD placement has changed.

  `active+remapped+backfill_wait`     PG is waiting for a backfill
                                      opportunity.

  `active+remapped+backfilling`       Backfill is actively moving
                                      objects.
  -----------------------------------------------------------------------

The misplaced-object percentage can rise or fall while placement changes
and backfill are in progress. It is a snapshot of placement convergence,
not a counter that must decrease at every observation.

`HEALTH_OK` does not by itself mean that recovery has finished.

## 12. Final verification checklist

After recovery has settled, verify:

-   [ ] All six OSDs appear in `ceph osd tree`.
-   [ ] All six OSDs are `up` and `in`.
-   [ ] The new OSDs are running on their intended hosts.
-   [ ] Misplaced objects have reached 0%.
-   [ ] All PGs are `active+clean`.
-   [ ] No recovery or backfill work remains pending.
-   [ ] Cluster health is `HEALTH_OK`.
-   [ ] OSD utilization and cluster capacity look reasonable.

The expansion can be considered complete after these checks pass and the
final state is confirmed with the senior/team.

## 13. Result of our task

We added one new OSD to each of the three Wahda hosts.

  Host          Original OSD     New OSD       Final OSD count
  ------------- ---------------- ----------- -----------------
  `thewahda4`   `osd.0`          `osd.3`                     2
  `thewahda5`   `osd.1`          `osd.4`                     2
  `thewahda6`   `osd.2`          `osd.5`                     2
  **Total**     **3 existing**   **3 new**               **6**

At our last verification, all six OSDs were `up` and `in`. The cluster
was still redistributing data, so final recovery completion had to be
monitored separately.

## Quick command reference

  Purpose                      Command
  ---------------------------- -----------------------------------
  Cluster health               `ceph -s`
  OSD hierarchy and status     `ceph osd tree`
  OSD service specifications   `ceph orch ls --service_type osd`
  OSD daemon status            `ceph orch ps --daemon_type osd`
  Disk identification          `lsblk`
  Disk signature inspection    `wipefs -n /dev/sdc`
  Apply an OSD specification   `ceph orch apply -i <filename>`
  PG summary                   `ceph pg stat`
  OSD utilization              `ceph osd df tree`
  Recovery progress            `ceph progress`
  Health details               `ceph health detail`
