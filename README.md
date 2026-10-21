# PHP2Shell exposure dashboard for Dynatrace

**CVE-2026-109618 / CVE-2026-109619 — PHP version control (Kubernetes + Compute)**
Provided by PacketPerf - L'expertise 100 % Dynatrace

## What it is

A ready-to-import Dynatrace dashboard that answers three questions about the PHP2Shell flaw disclosed on 21 October 2026 (unauthenticated remote code execution in the PHP core, CVSS 4.0: 9.2):

1. Are we exposed?
2. Where: which Kubernetes namespaces, which workloads, which servers?
3. What is left to fix, and is the remediation progressing?

It relies only on data Dynatrace already collects (Smartscape topology and OneAgent process detection). No additional agent, no extension, no data leaves your tenant.

Fixed PHP releases: **8.5.12, 8.4.27, 8.3.36, 8.2.35**. Branches 8.1 and older are end of life and have no official fix.

## Prerequisites

- Dynatrace SaaS with Grail
- OneAgent on the hosts and Kubernetes nodes running PHP. Processes without OneAgent are not visible.
- Permission to create dashboards and to run DQL queries on Smartscape (standard for an environment user).
- For the Kubernetes columns (cluster, workload, image): Kubernetes API monitoring connected on the cluster. Without it, containers appear with "unknown cluster" / "unknown workload".

## Import

1. Open the **Dashboards** app, click **Upload**, select `php2shell-cve-2026.upload.json`.
2. Two variables appear at the top, all values selected by default:
   - **Namespace**: Kubernetes namespaces in scope.
   - **Host**: classic servers (VMs, bare metal, containers outside Kubernetes) in scope.
3. The dashboard timeframe only drives the two history charts. Every other tile looks at processes seen in the last 15 minutes, whatever the timeframe.

## How to read it

**Statuses**, derived from the PHP runtime version detected by OneAgent:

| Status | Meaning | Action |
|---|---|---|
| VULNERABLE | Supported branch (8.2–8.5), fix not applied | Update to the fixed release, restart |
| VULNERABLE_EOL | Branch 8.1 or older, no official fix | Migrate to 8.2+ (or distribution backport meanwhile) |
| PATCHED | Fixed release or newer | Nothing |
| UNKNOWN | Version not readable | Check manually (see the dedicated table) |

**Top row**: vulnerable running processes, **workloads / hosts to fix** (the actionable number: images to rebuild and servers to update), impacted environments, compliance rate.

**Second row**: split Kubernetes / Compute, unreadable versions, total running PHP processes.

**Sanity tiles**: *Smartscape freshness* must stay below 15 minutes, otherwise the zero values cannot be trusted; *Containerized processes not attached to Kubernetes* should stay near 0.

**Tables**: status by environment (one row per namespace or host), remediation targets (one row per workload or host, with workload kind and image), vulnerable processes in detail, unreadable versions with process lifetime, Application Security findings for the two CVEs.

**Charts**: vulnerable processes over time, by scope and by environment. They count every process that existed during each time bucket, so a frequent CronJob inflates the height; watch the trend, which must fall to zero as updates roll out.

### Why "running" processes only

Smartscape keeps every process that existed during the selected timeframe. A CronJob creates a new process on every run, so a namespace can show hundreds of "vulnerable" processes that ended days ago. The dashboard therefore counts only processes seen in the last 15 minutes; the charts keep the history. A few recent CronJob runs may still appear, which does not change the number of workloads to fix.

## Remediation

1. Update PHP to the fixed release of your branch and restart PHP-FPM / Apache. Kubernetes: rebuild the image and redeploy the workload, CronJobs included (each run restarts the old image until it is replaced). Compute: distribution package and service restart.
2. Branches 8.1 and older: migrate to 8.2 or newer; a distribution backport (Ubuntu, sury.org) only bridges the gap.
3. Interim mitigation, which reduces exposure without removing it: disable `session.upload_progress.enabled` in php.ini; prefer PHP-FPM over mod_php, the most exposed setup; enable your WAF vendor's temporary rules.
4. UNKNOWN processes: a lifetime of a few minutes usually means a job that ended before detection; otherwise run `php -v` on the machine or inside the container.

## Limits

- The dashboard reads the PHP **runtime** version. PHP extensions and Composer dependencies are not covered.
- Processes without OneAgent do not appear.
- Attribution to a namespace requires Kubernetes monitoring; otherwise the process is counted under its host, in the Compute scope.

## Updating the fixed versions

When PHP publishes a new security release, the thresholds must be updated. Ask your PacketPerf contact for a regenerated file (the fixed versions are a single constant in our generator). If you prefer to edit it yourself, each tile query carries the four patch numbers in the expression `v_minor == 2 and v_patch < 35`, and so on for 8.3, 8.4 and 8.5.

## Support

Questions, a Kubernetes cluster that shows "unknown workload", a version that is not detected: contact your usual PacketPerf representative. We can also adapt the dashboard to another runtime (Java, Node.js, .NET) with the same approach.
