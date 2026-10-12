# Nutanix AHV and Prism Guide.


## Table of Contents

* [1. VM Templates](#1-vm-templates)
* [2. Metrics and Entities](#2-metrics-and-entities)
* [3. Metric Chart vs Entity Chart](#3-metric-chart-vs-entity-chart)
* [4. AHV High Availability](#4-ahv-high-availability)
* [5. N-1 Capacity Planning](#5-n-1-capacity-planning)
* [6. Rebuild Capacity Reservation](#6-rebuild-capacity-reservation)
* [7. Remote Support](#7-remote-support)
* [8. Proxy Server](#8-proxy-server)
* [9. Syslog](#9-syslog)
* [10. SMTP and Email Alerts](#10-smtp-and-email-alerts)
* [11. SSH Authentication](#11-ssh-authentication)
* [12. Cluster Lockdown](#12-cluster-lockdown)
* [13. Quick Revision](#13-quick-revision)
* [14. Official References](#14-official-references)

## 1. VM Templates

### What is a VM template?

A VM template is a reusable golden image or master copy of a virtual machine. It can contain:

* Guest operating system
* Common applications and packages
* Standard users and configuration
* Updates and security patches
* VM hardware configuration

A template allows administrators to deploy multiple consistent VMs without repeating the complete installation process.

### Example: Creating 100 Linux VMs

Without a template:

* Install Linux 100 times.
* Create users 100 times.
* Apply patches 100 times.
* Install common packages 100 times.
* Repeat testing and configuration for every VM.

With a template:

1. Create and configure one Linux VM.
2. Install the required applications and patches.
3. Configure the standard users and settings.
4. Save the VM as a Linux template.
5. Deploy new VMs from the template.
6. Customize each VM with its own hostname, IP address, and application settings.

Example deployed VMs:

```text
Linux-Template
├── Syslog-VM
├── App-VM-01
└── App-VM-02
```

### Template best practices

* Do not keep duplicate hostnames or static IP addresses in the template.
* Generalize Windows VMs with Sysprep when required.
* Remove or regenerate machine-specific Linux identity data when required by the deployment standard.
* Keep templates patched and use a clear version naming standard.
* Test the template before deploying production VMs.

## 2. Metrics and Entities

### What is an entity?

An entity is the resource being monitored or analyzed.

Examples:

* Virtual machine
* AHV host
* Cluster
* Storage container
* Disk
* Network interface

### What is a metric?

A metric is a measurable value that describes the performance, utilization, or behavior of an entity.

Examples:

* CPU utilization
* Memory usage
* Disk usage
* Storage IOPS
* Storage throughput
* Network received traffic
* Network transmitted traffic
* Latency

### Example

For the statement below:

> `Bastion-VM used 35% CPU`

* Entity: `Bastion-VM`
* Metric: CPU utilization
* Metric value: `35%`
* Time: The selected timestamp or time range

### Metric, alert, and event comparison

| Term | Meaning | Example |
|---|---|---|
| Metric | A performance or utilization measurement | CPU usage is 75% |
| Alert | A notification generated when a condition needs attention | CPU crossed a critical threshold |
| Event | A recorded activity or system change | A host became unavailable |

## 3. Metric Chart vs Entity Chart

In the Analysis custom-chart interface discussed during training:

| Chart type | Entity selection | Metric selection | Main purpose |
|---|---|---|---|
| Metric chart | Multiple entities | One metric | Compare the same metric across resources |
| Entity chart | One entity | Multiple metrics | Review several metrics for one resource |

### Metric chart

A metric chart compares one metric across multiple entities.

Example:

* Metric: Disk usage
* Entities: `Bastion-VM`, `PrismCentral-VM`, and `Syslog-VM`
* Question answered: “Which VM has the highest disk usage?”

Use a metric chart when the metric stays the same and the entities change.

### Entity chart

An entity chart reviews multiple metrics for one entity.

Example:

* Entity: `Bastion-VM`
* Metrics: CPU, memory, disk, network received, and network transmitted
* Question answered: “What is the overall performance of this VM?”

Use an entity chart when the entity stays the same and the metrics change.

### Easy way to remember

```text
Metric chart = one metric + multiple entities
Entity chart = one entity + multiple metrics
```

### Version caveat

Nutanix analytical chart behavior and terminology can vary across Prism and AOS releases. Current Nutanix documentation generally describes analytical charts around selected metrics and entities, so verify the exact chart behavior in the installed version.

### Common use cases

| Requirement | Chart to use |
|---|---|
| Compare CPU usage for five VMs | Metric chart |
| Compare memory usage for three hosts | Metric chart |
| Review CPU, memory, disk, and network for one VM | Entity chart |
| Investigate a problematic VM | Entity chart |
| Find the VM with the highest disk usage | Metric chart |

Charts can normally be viewed over time ranges such as the last hour, day, week, month, or a custom range, depending on the Prism version.

## 4. AHV High Availability

### What is VM High Availability?

AHV VM High Availability helps restart VMs on healthy AHV hosts when the original host becomes unavailable.

Important: after a host failure, this is a VM restart, not live migration. The guest operating system experiences a power-on/start cycle.

### Default or Best-effort mode

Best effort is the default behavior in AHV.

* No compute reservation or admission control is enforced for VM failover.
* AHV attempts to restart affected VMs on remaining hosts.
* If the remaining hosts do not have enough resources, some VMs may not restart.
* It provides higher normal capacity utilization but does not guarantee recovery of every affected VM.

### Guarantee mode with HA Reservation

Selecting **Enable HA Reservation** enables the nondefault Guarantee mode.

* Nutanix calculates and reserves resources across AHV hosts.
* The calculation considers VM configuration and the configured number of host failures to tolerate.
* The reservation method is segment-based in current AHV guidance.
* Usable capacity for normal VM placement is reduced.
* Admission control helps prevent placements that would violate the configured failure guarantee.

The reservation should not be described as simply reserving one complete host or exactly one-third of memory on each host.

### Three-host example

Assume:

```text
Host 1: 500 GB memory
Host 2: 500 GB memory
Host 3: 500 GB memory
Total: 1,500 GB before system overhead
```

With Best effort:

* The cluster can use more of the available capacity for normal workloads.
* After a host failure, recovery depends on the free resources remaining on the other hosts.

With Guarantee mode for one host failure:

* Nutanix calculates the required reservation.
* The exact reservation is not necessarily 500 GB.
* The reservation depends on VM memory configuration, failure tolerance, reservation segments, and system overhead.
* The reserved resources help the affected VMs restart on the remaining hosts.

Use the reservation amount and fault-tolerance status shown in Prism for actual capacity planning.

## 5. N-1 Capacity Planning

N-1 means planning for the cluster to continue operating after one node is unavailable.

For a three-node cluster:

```text
N = 3 nodes
N-1 = 2 nodes remain after one node fails
```

The remaining nodes must have enough capacity for:

* Required VM restarts
* CPU
* Memory
* Storage capacity
* Storage performance
* Network capacity
* CVM and system overhead
* Storage availability and resiliency requirements

N-1 is a planning principle. The actual AHV VMHA reservation depends on the configured failure tolerance and Nutanix's reservation calculation.

## 6. Rebuild Capacity Reservation

Rebuild Capacity Reservation reserves free capacity for storage rebuild operations after a disk, node, or other storage component failure.

### Why is it important?

When data must be rebuilt on healthy components, the cluster needs available capacity. Without sufficient capacity:

* Rebuild operations may be affected.
* Data resiliency may be reduced.
* Recovery and storage performance may be impacted.
* The cluster may reach a critical capacity condition.

### HA reservation versus rebuild reservation

| Feature | Protects | Reserved resource |
|---|---|---|
| HA Reservation | VM availability after an AHV host failure | Compute resources for VM restart |
| Rebuild Capacity Reservation | Storage resiliency after a storage failure | Free storage capacity for rebuild |

## 7. Remote Support

Remote support provides a controlled support connection that can help Nutanix support personnel investigate cluster issues, collect logs, and troubleshoot alerts.

Security and access behavior depend on the configured environment and product version.

Before enabling remote support:

* Follow the customer security policy.
* Obtain the required approval and change record.
* Confirm what access is enabled and for how long.
* Disable or restrict access when it is no longer required.

## 8. Proxy Server

A proxy server acts as an intermediary between a private Nutanix network and an external destination.

Example:

```text
Nutanix cluster  ->  Proxy server  ->  Internet
Nutanix cluster  <-  Proxy server  <-  Internet
```

A proxy can be used when the cluster cannot access Nutanix services directly.

Benefits include:

* Avoiding direct internet access from the cluster.
* Centralizing outbound access control.
* Allowing or blocking specific destinations.
* Supporting security monitoring and auditing.

A proxy does not replace firewall rules, authentication, certificate validation, or URL filtering.

## 9. Syslog

Syslog is a standard method for forwarding system logs to a centralized log server.

### Why use Syslog?

A Nutanix environment can contain multiple clusters, Controller VMs, and AHV hosts. Centralized Syslog helps administrators:

* Collect logs in one location.
* Search and correlate events.
* Retain logs according to policy.
* Integrate logs with monitoring and SIEM platforms.
* Troubleshoot multiple systems more efficiently.

### Common configuration items

* Syslog server IP address or hostname
* Port number
* Transport protocol
* Log modules or categories
* Severity level
* Retention and monitoring requirements

Syslog, metrics, alerts, and events are different:

```text
Metrics = performance measurements
Alerts  = notifications about conditions
Logs    = recorded messages and activities
```

## 10. SMTP and Email Alerts

SMTP stands for Simple Mail Transfer Protocol. Prism uses SMTP to send alert email notifications.

### Typical configuration items

* SMTP server hostname or IP address
* SMTP port
* TLS or encryption settings
* Authentication details, if required
* Sender address
* Recipient addresses
* Alert severity and impact filters
* Every-alert or daily-digest preference

Example notification conditions:

* Critical host failure
* Storage capacity warning
* Cluster resiliency issue
* Hardware or system failure

Avoid sending every low-priority event to every recipient. Configure recipients and severity filters according to the operational support model.

## 11. SSH Authentication

SSH provides secure remote access to Linux systems and infrastructure components.

### Password-based authentication

```bash
ssh user@192.168.10.20
```

The user authenticates with a username and password.

Limitations:

* Passwords can be guessed, reused, leaked, or stolen.
* Password management becomes difficult at scale.
* It may not meet stricter security requirements.

### Key-based authentication

Key-based authentication uses a key pair:

* Private key: kept securely by the client and never shared.
* Public key: added to the target system or Nutanix interface.

Example connection:

```bash
ssh -i /path/to/private_key user@192.168.10.20
```

General workflow:

1. Generate a key pair.
2. Add the public key to the target system.
3. Protect the private key with correct permissions and a passphrase where appropriate.
4. Test access using the private key.
5. Disable password access only after key-based access is confirmed.

## 12. Cluster Lockdown

Cluster Lockdown controls remote SSH access to Nutanix cluster resources.

In the common password-lockdown configuration:

* Password-based remote login is disabled.
* Approved public keys can be used for key-based access.

Nutanix has multiple lockdown and security modes across releases. Some modes can restrict both password-based and public-key SSH access, so verify the exact behavior for the installed AOS/Prism version.

### Before enabling lockdown

* Confirm that approved key-based access works.
* Verify that authorized administrators can connect.
* Check automation and monitoring dependencies.
* Confirm support and recovery procedures.
* Follow the customer change-management process.
* Do not disable all access without a tested recovery method.

## 13. Quick Revision

| Question | Answer |
|---|---|
| Why use a VM template? | To deploy multiple consistent VMs without repeating the full installation and configuration process. |
| What is an entity? | The resource being monitored, such as a VM, host, cluster, or container. |
| What is a metric? | A measurable performance or utilization value, such as CPU, memory, IOPS, or network traffic. |
| What is a metric chart? | One metric compared across multiple entities. |
| What is an entity chart? | Multiple metrics reviewed for one entity in the training interface. |
| What is Best-effort HA? | Default AHV VMHA behavior without a guaranteed resource reservation. |
| What is Guarantee mode? | AHV VMHA mode that reserves calculated resources for the configured host-failure tolerance. |
| What is N-1? | Planning for operation after one node is unavailable. |
| What is rebuild reservation? | Reserved storage capacity for rebuild operations after a failure. |
| Why use Syslog? | To centralize logs from multiple systems. |
| Why configure SMTP? | To send alert notifications by email. |
| What is key-based SSH? | SSH authentication using a public/private key pair. |
| What is Cluster Lockdown? | A security feature that controls password and/or key-based SSH access, depending on the configured mode and release. |


## 14. Official References

* [Nutanix Prism Metrics](https://portal.nutanix.com/page/documents/solutions/details?targetId=TN-2042-Prism%3Anutanix-prism-metrics.html)
* [Nutanix VM Template Management](https://portal.nutanix.com/page/documents/details?targetId=Prism-Central-Guide-vpc_2024_3%3Amul-vm-template-create-manage-pc-c.html)
* [Nutanix AHV Virtual Machine High Availability](https://portal.nutanix.com/page/documents/solutions/details?targetId=BP-2029-AHV%3Abp-nutanix-ahv-virtual-machine-high-availability.html)
* [Enabling High Availability Reservations](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide-v6_10%3Aahv-acropolis-cluster-setting-ha-t.html)
* [Reserving Rebuild Capacity](https://portal.nutanix.com/page/documents/details?targetId=Web-Console-Guide-Prism%3Awc-storage-enabling-rebuild-capacity-reservation-wc-t.html)
* [Controlling Remote SSH Access](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Security-Guide-v6_10%3Amul-security-cluster-lockdown-pc-t.html)
* [Configuring HTTP Proxy](https://portal.nutanix.com/page/documents/details?targetId=Web-Console-Guide-Prism-v6_10%3Awc-support-http-proxies-wc-t.html)
* [Configuring Alert Emails in Prism Central](https://portal.nutanix.com/page/documents/details?targetId=Prism-Central-Admin-Center-Guide-vpc_7_5%3Amul-alert-emails-configure-pc-t.html)
