# Client capacity measurements

A small server must be measured with the complete deployed service set: Odoo 19,
PostgreSQL, Oduflow, Traefik, IDE, Salt Minion, Tailscale and Docker/containerd.
An idle Odoo process alone does not establish usable client capacity.

## Procedure

The probe source is `scripts/capacity_probe.py` in the platform repository.
Transfer the reviewed standalone script to the target client and run it there
as root; the commands below assume its directory is the working directory.

1. Sample `/proc/meminfo`, `/proc/vmstat`, `/proc/stat`, load averages and CPU,
   memory and I/O pressure every two seconds during cold installation. Keep the
   sample duration bounded and save only these counters in a root-owned file.
2. Record actual RAM, swap size/location, `vm.swappiness`, root disk and attached
   data volume. Provider images may preconfigure swap independently of Salt.
3. After provisioning completes, run an idle baseline on the client:

   ```sh
   python3 capacity_probe.py --instance-uuid CLIENT_UUID --samples 7 --interval 10
   ```

4. Run a small HTTP readiness workload separately:

   ```sh
   python3 capacity_probe.py --instance-uuid CLIENT_UUID --samples 11 --interval 10 --workload
   ```

   This sends twenty serial GET requests to the production Odoo login page via
   local Traefik. TLS validates the real production hostname. It does not log in,
   modify business records, create environments, or bypass authentication.
5. Continue sampling while running one authenticated Odoo read-only workflow and
   one real IDE/OpenCode coding task. Record their start/end timestamps and
   results separately. Upstream model/DNS failures are not local capacity proof.
6. Inspect effective Odoo workers and memory thresholds, PostgreSQL settings,
   container memory limits, systemd memory and restart counters, and filesystem
   utilization. Do not print environment variables or full application configs.

The probe accepts only the expected Salt instance identity, requires the data
volume to be mounted, and obtains the production hostname from its existing
configuration. Its output includes selected service and container metadata, not
credentials or application log contents. Redirect stdout to a private file if
retaining measurements.

## Interpretation

Report minimum observed `MemAvailable`, maximum observed swap usage, swap-in/out
and OOM counter changes, CPU utilization/steal, pressure and HTTP latency. State
the sampling interval: short peaks between samples can be missed. Linux cache is
reclaimable, so `MemAvailable` is more useful than simply subtracting reported
used memory from total memory. Process RSS sums can double-count shared pages;
the probe reports service/container measurements separately. Systemd cgroup
`MemoryCurrent` includes file cache as well as anonymous memory; inspect the
reported `memory_stat_bytes` before treating the whole cgroup as unreclaimable.

A 2 GB server with 6 GB of swap is a different configuration from a server with
2 GB of RAM and no swap. Successful completion with swap does not establish that
there is enough RAM to avoid latency during concurrent workloads. Check whether
swap activity continues during steady-state work and whether memory pressure or
latency grows. CPU saturation during package extraction is distinct from a
server that remains saturated during a single ordinary request.

The committed client configuration does not currently set explicit systemd
memory caps for Oduflow/IDE or pass per-production resource limits in the create
request. Effective Odoo/PostgreSQL defaults come from the pinned Oduflow package
and generated stack and must be inspected on the deployed client. The control
Odoo's `.oduflow/odoo.conf` is not the client production configuration.

The Salt template permits five development environments and three services when
slot values are absent, but normal provisioning supplies the frozen plan values.
Lifecycle auto-stop and auto-delete also come from that plan (the standard plan
uses 12 and 72 hours); the template fallback of zero applies only when omitted.
Inspect the actual plan and rendered runtime configuration. Slot counts and
lifecycle settings are allocation rules, not a capacity guarantee. A small-server
result should name exactly which stacks and agents were running; it must not
imply that all available slots fit in memory.
