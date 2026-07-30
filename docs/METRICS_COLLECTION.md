# Node & JVM Metrics Collection

This document describes exactly how node-level CPU and memory metrics, and Java
heap metrics, are collected during a benchmark run. It covers the tools used,
the precise commands, the rationale for the choices, the sampling cadence, and
the process-lifecycle (how the collectors are started and stopped).

All node-level metrics are collected by two bash scripts in `ansible/scripts/`,
launched via Ansible at benchmark start and killed at benchmark end. Both write
a CSV row every ~1 second until they receive `SIGTERM`.

| Metric | Script | Mechanism | Cadence | Started by | Stopped by |
|---|---|---|---|---|---|
| Service-tree CPU % (1 core = 100 %) | `collect_process_metrics.sh` | `/proc/<pid>/stat` jiffy delta + `date +%s%N` elapsed | bash `while true; sleep 1` | `nohup` from playbook | `kill $(cat <csv>.pid)` |
| Service-tree RSS / VSZ (MiB) | `collect_process_metrics.sh` | `/proc/<pid>/status` `VmRSS` / `VmSize` summed across tree | same loop, same row | same | same |
| Host CPU % (core-percent, max ≈ NCPU × 100) | `collect_process_metrics.sh` | `/proc/stat` `cpu` line, busy/total delta × NCPU | same loop, same row | same | same |
| OJP Java heap used / committed + GC | `collect_jvm_metrics.sh` | `jstat -gc <ojp_pid> 1000` piped into `while read` | `jstat`'s own 1000 ms tick | `nohup` from `run_benchmarks_ojp.yml` | `kill $(cat <csv>.pid)` |

---

## 1. Node CPU & memory — `ansible/scripts/collect_process_metrics.sh`

Run against a systemd unit name (e.g. `ojp-server`, `pgbouncer`,
`postgresql@16-main`) on each tier node. Produces a CSV with columns:

```
timestamp, pid, cpu_pct, host_cpu_pct, rss_mb, vsz_mb
```

It captures both the **service process tree** (`cpu_pct` / `rss_mb` / `vsz_mb`)
and the **whole host** (`host_cpu_pct`).

### 1.1 How the PID set is discovered

- `systemctl show -p MainPID <service>` returns the root PID of the unit
  (`collect_process_metrics.sh:37`).
- Each polling cycle, a BFS walks the tree using `pgrep -P <pid>` to collect
  every descendant (`collect_service_tree_pids`,
  `collect_process_metrics.sh:110-133`). This is re-done every second so forked
  children are picked up.

### 1.2 How CPU % is computed (no extra packages required)

- For every PID in the tree the script reads fields 14 (`utime`) and 15
  (`stime`) from `/proc/<pid>/stat`:

  ```bash
  awk '{print $14+$15}' /proc/<pid>/stat
  ```

  (`collect_process_metrics.sh:79-87`). Both fields are in clock ticks.
- A per-PID map of the previous reading (`prev_jiffies_by_pid`) is kept and the
  deltas (`delta_jiffies`) are summed across the tree.
- Wall-clock elapsed time is measured with `date +%s%N` (nanoseconds,
  `collect_process_metrics.sh:184`) — not assumed from `sleep 1` — to remain
  accurate when `sleep` drifts.
- Conversion (`collect_process_metrics.sh:191-196`):

  ```
  cpu_pct = delta_jiffies / CLK_TCK / elapsed_s * 100
  ```

  with `CLK_TCK=$(getconf CLK_TCK)`. The result is **normalised to 1 core =
  100 %**.

### 1.3 How host CPU % is computed

- The script reads `/proc/stat`'s first `cpu ` line (`read_host_cpu_totals`,
  `collect_process_metrics.sh:97-108`) and computes
  `busy = total − (idle + iowait)`.
- Each cycle it diffs total/idle and scales to **core-percent**
  (`collect_process_metrics.sh:200-205`):

  ```
  host_cpu_pct = (1 − idle_delta/total_delta) * 100 * HOST_CPU_COUNT
  ```

  where `HOST_CPU_COUNT=$(getconf _NPROCESSORS_ONLN)`. So 100 % = one fully
  busy CPU, and the maximum is `NCPU * 100`. This matches the cloud-dashboard
  scaling used in `ansible/scripts/generate_report.sh`.

### 1.4 How memory is computed

- For each PID in the tree the script parses `/proc/<pid>/status`
  (`read_kb_status_field`, `collect_process_metrics.sh:89-95`):
  - `VmRSS` → summed into `rss_kb_total` → MiB (resident RAM actually in use).
  - `VmSize` → summed into `vsz_kb_total` → MiB (virtual address space).
- Both are summed across the whole process tree
  (`collect_process_metrics.sh:173-176`) and divided by 1024 to get MiB.

### 1.5 Why these choices

Taken from the file's own header comments:

- **`/proc` over `pidstat` / `ps`** — `/proc` is on every Linux kernel without
  installing `sysstat`. `pidstat` reports per-CPU percentages; `/proc/stat` +
  `CLK_TCK` lets the script normalise to "1 CPU = 100 %" regardless of vCPU
  count.
- **Per-PID delta map (not a single combined read)** — a forked child that
  exits between samples would otherwise produce a huge negative delta. Failed
  `/proc/<pid>/stat` reads emit *nothing* and the PID is skipped that cycle
  (`collect_process_metrics.sh:79-87`, `162-168`).

### 1.6 Polling cadence and process model

- Main loop (`collect_process_metrics.sh:141-219`):

  ```bash
  while true; do
      sleep 1
      … sample tree, compute, append CSV row …
  done
  ```

- One CSV row per second. The loop exits when the root PID's
  `/proc/<pid>/stat` disappears or the tree has no live PIDs
  (`collect_process_metrics.sh:145-154`).
- Launched by the playbooks with `nohup`, recording its own PID to
  `<csv>.pid` (`collect_process_metrics.sh:49` and e.g.
  `ansible/playbooks/run_benchmarks_ojp.yml:342-344`). At end-of-run the
  playbook reads that file and `kill`s it
  (`ansible/playbooks/run_benchmarks_ojp.yml:514-516`, `538-540`).
- Instances per run: on the DB node for the Postgres service, plus one per
  proxy node for the proxy service (`ojp-server`, `pgbouncer`, etc.).

---

## 2. Java heap (OJP only) — `ansible/scripts/collect_jvm_metrics.sh`

Produces a CSV with columns:

```
timestamp, heap_used_mb, heap_committed_mb, ygc_count, ygct_s, fgc_count, fgct_s, gct_s
```

### 2.1 Tool and exact command

- The OJP server PID comes from `systemctl show -p MainPID ojp-server`
  (`collect_jvm_metrics.sh:24`).
- `jstat` is located by resolving `which java` → `JAVA_HOME/bin/jstat`
  (`collect_jvm_metrics.sh:36-43`), so it always matches the JDK that is
  actually running the server.
- The single command that drives the entire collection
  (`collect_jvm_metrics.sh:72`):

  ```bash
  jstat -gc <OJP_PID> 1000
  ```

  i.e. `jstat -gc` with a 1000 ms sampling interval. `jstat` itself prints a
  new row every second indefinitely; the script just pipes its output into a
  `while read line` loop.

### 2.2 Why `jstat -gc` and not OS RSS

Quoted directly from the script header:

> Java acquires memory from the OS and does NOT return it after GC, so OS RSS
> (as reported by `ps`/`top`) significantly overstates actual heap usage.
> `jstat -gc` reports actual in-use heap from the JVM's own memory pools.

That is why OJP's "heap" comes from `jstat` while `rss_mb` from
`collect_process_metrics.sh` is kept as a separate (and known-to-be-higher)
signal.

### 2.3 How `heap_used_mb` / `heap_committed_mb` are derived

`jstat -gc` emits these columns:

```
S0C S1C S0U S1U EC EU OC OU MC MU CCSC CCSU YGC YGCT FGC FGCT CGC CGCT GCT
```

(sizes in KiB, times in seconds). The script does
(`collect_jvm_metrics.sh:85-86`):

```
heap_used_mb      = (S0U + S1U + EU + OU) / 1024
heap_committed_mb = (S0C + S1C + EC + OC) / 1024
```

i.e. survivor 0 + survivor 1 + Eden + Old (used and committed respectively).
Metaspace (`MC` / `MU` / `CCSC` / `CCSU`) is intentionally excluded from
"heap". The GC counters (`YGC`, `YGCT`, `FGC`, `FGCT`, `GCT`) are passed
through verbatim.

Lines starting with letters (header lines `jstat` reprints) are skipped, and
rows without a numeric `GCT` are discarded as a parse guard
(`collect_jvm_metrics.sh:73-83`).

### 2.4 Polling cadence and process model

- The 1-second cadence is driven by `jstat` itself
  (`jstat -gc <pid> 1000`), not by `sleep`. There is no busy loop in bash —
  the `while IFS= read -r line` simply consumes `jstat`'s stdout one sample at
  a time.
- The script writes its own PID to `<csv>.pid`
  (`collect_jvm_metrics.sh:47`) and is launched by
  `ansible/playbooks/run_benchmarks_ojp.yml` with `nohup`
  (`ansible/playbooks/run_benchmarks_ojp.yml:292-301`). The same playbook
  `kill`s it via that PID file at end-of-run
  (`ansible/playbooks/run_benchmarks_ojp.yml:462-464`).
- It runs on every OJP proxy node, alongside that node's
  `collect_process_metrics.sh`. It is **not** run for the Hikari or PgBouncer
  playbooks (no JVM proxy there to sample).

---

## 3. How the collected CSVs are consumed

Both scripts run for the full duration of a benchmark run (warmup +
steady-state) and produce one CSV row per second. `ansible/scripts/generate_report.sh`
then aggregates those rows into the steady-state tier statistics
(avg / p50 / p95 / p99 / peak) that appear in the final report. See
[`docs/METRICS.md`](METRICS.md) for the report-level metric definitions.
