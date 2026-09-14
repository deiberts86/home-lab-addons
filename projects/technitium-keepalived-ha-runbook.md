# Technitium DNS HA with Keepalived

**NOTE** This was just a fun homelab setup to make my DNS for my home and lab to be more reliable.

This runbook describes how to install Technitium DNS Server and place a highly available IPv4 virtual IP (VIP) in front of two clustered Technitium nodes using Keepalived/VRRP and UFW.

The architecture is intentionally **OS-agnostic**, but the implementation commands in this guide are written for **Debian-family Linux distributions** that use APT and systemd, such as Debian and Ubuntu. The examples intentionally use placeholders so the procedure can be reused in different environments.

---

## Table of Contents

- [1. Architecture](#1-architecture)
- [2. Example Variables](#2-example-variables)
- [3. Requirements](#3-requirements)
  - [3.1 Operating System](#31-operating-system)
  - [3.2 Network Requirements](#32-network-requirements)
  - [3.3 DNS Requirements](#33-dns-requirements)
- [4. Install Technitium and Required Packages](#4-install-technitium-and-required-packages)
  - [4.1 Install Base Packages on Debian-Family Systems](#41-install-base-packages-on-debian-family-systems)
  - [4.2 Install Technitium DNS Server](#42-install-technitium-dns-server)
  - [4.3 Verify DNS Port Availability](#43-verify-dns-port-availability)
  - [4.4 Optional but Recommended: Install DNS-over-QUIC Support](#44-optional-but-recommended-install-dns-over-quic-support)
  - [4.5 Configure DNS-over-QUIC as an Upstream Forwarder](#45-configure-dns-over-quic-as-an-upstream-forwarder)
  - [4.6 Optional: Serve DNS-over-QUIC to Clients](#46-optional-serve-dns-over-quic-to-clients)
  - [4.7 Initial Technitium Configuration](#47-initial-technitium-configuration)
  - [4.8 Verify Time Synchronization](#48-verify-time-synchronization)
  - [4.9 Configure Technitium Clustering](#49-configure-technitium-clustering)
- [5. Verify the Physical Interfaces](#5-verify-the-physical-interfaces)
- [6. Configure the Technitium Health Check](#6-configure-the-technitium-health-check)
- [7. Configure UFW](#7-configure-ufw)
- [8. Configure VRRP Firewall Rules](#8-configure-vrrp-firewall-rules)
- [9. Remove an Incorrect UFW Rule](#9-remove-an-incorrect-ufw-rule)
- [10. Configure Keepalived](#10-configure-keepalived)
- [11. Keepalived Configuration Notes](#11-keepalived-configuration-notes)
- [12. Optional: Legacy VRRP Authentication](#12-optional-legacy-vrrp-authentication)
- [13. Optional: Unicast VRRP](#13-optional-unicast-vrrp)
- [14. Validate the Keepalived Configuration](#14-validate-the-keepalived-configuration)
- [15. Verify VIP Ownership](#15-verify-vip-ownership)
- [16. Verify VRRP Traffic](#16-verify-vrrp-traffic)
- [17. Why tcpdump Can See VRRP While Keepalived Cannot](#17-why-tcpdump-can-see-vrrp-while-keepalived-cannot)
- [18. Test DNS Through the VIP](#18-test-dns-through-the-vip)
- [19. Perform a Real DNS-Service Failover Test](#19-perform-a-real-dns-service-failover-test)
- [20. Expected Steady-State Behavior](#20-expected-steady-state-behavior)
- [21. Troubleshooting](#21-troubleshooting)
- [22. Useful Operational Commands](#22-useful-operational-commands)
- [23. Firewall Summary](#23-firewall-summary)
- [24. Final Validation Checklist](#24-final-validation-checklist)
- [25. Reference Behavior](#25-reference-behavior)
- [26. References](#26-references)

---

## 1. Architecture

Example logical layout:

```text
                         DNS Clients
                             |
                             |
                      <DNS_VIRTUAL_IP>
                             |
                  +----------+----------+
                  |                     |
             DNS Node A             DNS Node B
          <DNS_NODE_A_IP>         <DNS_NODE_B_IP>
            priority 150           priority 100
          preferred MASTER            BACKUP
                  |                     |
                  +-------- VRRP -------+
                     IP protocol 112
                       224.0.0.18
```

Technitium clustering and Keepalived solve two different problems:

- **Technitium clustering** synchronizes supported DNS configuration between Technitium nodes.
- **Keepalived/VRRP** controls which operating-system host owns the shared DNS VIP.

The VIP does **not** need to be configured as a Technitium cluster node address.

Clients should normally be configured to use only the VIP as their DNS server:

```text
DNS Server: <DNS_VIRTUAL_IP>
```

---

## 2. Example Variables

Substitute values appropriate for the environment.

```text
DNS Node A hostname:       <dns-node-a>
DNS Node A IP:             <DNS_NODE_A_IP>
DNS Node A interface:      <DNS_NODE_A_INTERFACE>

DNS Node B hostname:       <dns-node-b>
DNS Node B IP:             <DNS_NODE_B_IP>
DNS Node B interface:      <DNS_NODE_B_INTERFACE>

DNS VIP:                   <DNS_VIRTUAL_IP>
Subnet prefix:             <PREFIX_LENGTH>
Health-check DNS zone:     <INTERNAL_DNS_ZONE>

VRRP Virtual Router ID:    53
Node A priority:           150
Node B priority:           100
```

Example VIP declaration:

```text
<DNS_VIRTUAL_IP>/<PREFIX_LENGTH>
```

The interface names do not need to match between the two servers.

For example, one node can use `eth0` while the other uses `enp1s0`.

---

# 3. Requirements

## 3.1 Operating System

The HA design itself is not tied to a specific Linux distribution. Keepalived/VRRP, Technitium, and the networking concepts apply broadly to Linux.

The command examples in this runbook assume a **Debian-family operating system** with:

- APT
- systemd
- UFW/netfilter
- Keepalived
- Technitium DNS Server

Examples include Debian and Ubuntu.

The commands assume `sudo` access.

Check the OS release:

```bash
cat /etc/os-release
```

Check the CPU architecture:

```bash
dpkg --print-architecture
```

Common architectures include:

```text
amd64
arm64
```

If the selected platform uses a different package manager, service manager, or firewall, translate the package-install, service-management, and firewall steps accordingly while keeping the same network and VRRP requirements.

---

## 3.2 Network Requirements

The two Keepalived nodes should have:

- Static or reserved IP addresses.
- A VIP that is unused by any other host.
- The VIP in the same IPv4 subnet as the Keepalived interfaces.
- Layer-2 connectivity between the two nodes when using multicast VRRP.
- IPv4 multicast permitted between the nodes.
- VRRP IP protocol `112` permitted between the nodes.
- Access to multicast address `224.0.0.18`.


### Why multicast is the default in this runbook

This runbook uses standard multicast VRRP by default because it is the native VRRP operating model, requires less peer-specific configuration, and works cleanly when both DNS nodes share the same Layer-2 network. It also keeps the configuration easy to expand if another VRRP participant is ever added.

Use unicast VRRP only when multicast is unavailable, filtered, unreliable, or undesirable in the network. An optional unicast configuration is included later in this guide.

The normal multicast VRRP flow is:

```text
<DNS_NODE_A_IP> ----\
                     >---- 224.0.0.18 / IP protocol 112
<DNS_NODE_B_IP> ----/
```

VRRP is **not TCP or UDP**.

Do not create rules for:

```text
TCP/112
UDP/112
```

VRRP itself is IP protocol:

```text
112
```

### VIP requirements

The VIP must:

- Not be assigned permanently to either DNS host.
- Not be present in DHCP pools.
- Not be assigned to another device.
- Be reachable by all client networks that will use it for DNS.

Routing/firewall infrastructure between client VLANs and the DNS subnet must allow DNS traffic to the VIP.

---

## 3.3 DNS Requirements

Both Technitium nodes should:

- Be installed and running.
- Be capable of answering the same internal DNS health-check query.
- Be configured consistently, preferably through Technitium clustering.
- Listen on UDP/53 and TCP/53.
- Be independently capable of performing recursive/forwarded lookups.

A useful health check is an SOA query against a private authoritative zone hosted by Technitium.

Example:

```bash
dig @127.0.0.1 <INTERNAL_DNS_ZONE> SOA +short
```

The query should return a valid SOA response on both nodes.

---

# 4. Install Technitium and Required Packages

Perform the following steps on **both DNS nodes**.

## 4.1 Install Base Packages on Debian-Family Systems

Install the base Debian packages used by this runbook:

```bash
sudo apt update
sudo apt install -y \
  curl \
  wget \
  ca-certificates \
  dnsutils \
  keepalived \
  ufw \
  tcpdump
```

Package purposes:

| Package | Purpose |
| --- | --- |
| `curl` | Runs the official Technitium installer |
| `wget` | Downloads Microsoft repository configuration when enabling QUIC |
| `ca-certificates` | TLS certificate trust for HTTPS package/download operations |
| `dnsutils` | Provides `dig` for health checks and troubleshooting |
| `keepalived` | VRRP and VIP ownership |
| `ufw` | Host firewall management |
| `tcpdump` | VRRP, DNS, and DoQ packet validation |

---

## 4.2 Install Technitium DNS Server

Technitium provides an automated Linux installer/updater.

Run on each DNS node:

```bash
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```

The installer installs Technitium and its required runtime components, configures the DNS server as a systemd service, and creates the standard service:

```text
dns.service
```

### Installer download troubleshooting

The official automated installer is retrieved from:

```text
https://download.technitium.com/dns/install.sh
```

If the installer fails to download, test the URL before troubleshooting Technitium itself:

```bash
curl -I https://download.technitium.com/dns/install.sh
```

and:

```bash
curl -v https://download.technitium.com/dns/install.sh -o /tmp/technitium-install.sh
```

In environments that use country-based egress filtering or GeoIP blocking, be careful with broad geographic deny rules. The Technitium download endpoint may be delivered through cloud/CDN infrastructure whose resolved address is geolocated outside the region you expect. In particular, **blocking traffic geolocated to India can prevent the installer or related Technitium downloads from working in some environments**.

If this occurs:

- allow outbound HTTPS to `download.technitium.com`;
- allow the cloud/CDN addresses that hostname resolves to at installation time;
- review firewall/proxy/GeoIP logs for denied HTTPS sessions;
- avoid assuming that the web site's apparent geographic location matches the actual download infrastructure.

The official Technitium documentation confirms `download.technitium.com` as the automated installer endpoint, but does not document a fixed underlying cloud provider. For that reason, this runbook intentionally does not hard-code a provider or IP range.

Check current resolution if needed:

```bash
getent ahosts download.technitium.com
```

or:

```bash
dig download.technitium.com A +short
dig download.technitium.com AAAA +short
```

Once access is confirmed, rerun:

```bash
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```

Verify:

```bash
sudo systemctl status dns
```

Enable the service at boot if needed:

```bash
sudo systemctl enable dns
```

Follow the service logs:

```bash
sudo journalctl -fu dns
```

The default Technitium web console is available at:

```text
http://<DNS_NODE_IP>:5380/
```

Open the console on each node and complete the initial administrator setup.

> **Important:** Port `5380/tcp` is the default HTTP management console. If HTTPS management is configured later, Technitium commonly uses `53443/tcp`.

---

## 4.3 Verify DNS Port Availability

Technitium must be able to bind TCP and UDP port 53.

Check whether another resolver is already listening:

```bash
sudo ss -lntup | grep ':53 '
```

Technitium's installer normally handles the local resolver configuration required for its supported installation flow. If Technitium fails to bind port 53, check for competing services such as:

```text
systemd-resolved
dnsmasq
unbound
bind9
```

Inspect listeners:

```bash
sudo ss -lntup
```

Do not disable an existing resolver blindly; first confirm which service owns port 53 and how the host currently manages `/etc/resolv.conf`.

After installation, verify local Technitium resolution:

```bash
dig @127.0.0.1 example.com A
```

---

## 4.4 Optional but Recommended: Install DNS-over-QUIC Support

Technitium uses Microsoft's **MsQuic** library for DNS-over-QUIC (DoQ) and HTTP/3 support on Linux.

The required package is:

```text
libmsquic
```

This package is **not required** if the deployment will only use traditional DNS, DoT, or other protocols that do not depend on QUIC. Install it when Technitium will:

- use a DNS-over-QUIC upstream forwarder,
- provide a DNS-over-QUIC listener to clients,
- or use HTTP/3 support.

### Debian 13

Add Microsoft's Debian 13 package repository:

```bash
wget https://packages.microsoft.com/config/debian/13/packages-microsoft-prod.deb \
  -O packages-microsoft-prod.deb

sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

sudo apt update
```

Then install MsQuic:

```bash
sudo apt install -y libmsquic
```

### Debian 12

For Debian 12:

```bash
wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb \
  -O packages-microsoft-prod.deb

sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

sudo apt update
sudo apt install -y libmsquic
```

### Other Debian-derived distributions

For another Debian-derived distribution, first inspect:

```bash
cat /etc/os-release
dpkg --print-architecture
```

Use the Microsoft repository matching the Debian base release when appropriate.

Do not assume the derived distribution's own `VERSION_ID` is necessarily a valid Microsoft repository path.

### Verify MsQuic

Confirm the package:

```bash
dpkg -l | grep libmsquic
```

or:

```bash
apt policy libmsquic
```

Restart Technitium after installing the library:

```bash
sudo systemctl restart dns
```

Verify Technitium came back normally:

```bash
sudo systemctl status dns
```

---

## 4.5 Configure DNS-over-QUIC as an Upstream Forwarder

Installing `libmsquic` only gives Technitium the ability to use QUIC. To actually encrypt upstream DNS queries with DNS-over-QUIC, configure one or more **DoQ-capable forwarders** in Technitium.

Conceptually:

```text
Client
   |
   | UDP/TCP 53 on the internal network
   v
Technitium
   |
   | DNS-over-QUIC
   | QUIC over UDP/853
   v
Encrypted upstream DNS provider
```

DoQ uses:

```text
UDP/853
```

The fact that packet captures show UDP is expected: QUIC runs over UDP while encrypting the DNS payload.

### Configure the forwarder in Technitium

From the Technitium web console:

1. Go to:

   ```text
   Settings -> Proxy & Forwarders
   ```

2. Under **Forwarders**, add the desired upstream resolver address or addresses.
3. Set the **Forwarder Protocol** to:

   ```text
   QUIC
   ```

4. Enable DNSSEC validation if it is desired for forwarded responses.
5. Save the settings.

Technitium supports a name-server address format that can include both the TLS hostname and a fixed IP address. This is useful because it avoids needing to resolve the forwarder's hostname before connecting while still retaining the hostname for TLS certificate validation.

Example format:

```text
dns.example.net (192.0.2.53:853)
```

When the protocol is set to `QUIC`, port `853` is the normal DoQ port.

### Example: Quad9 over DoQ

Quad9 provides DNS-over-QUIC on its resolver hostnames over port 853.

For the standard secure/blocking service, configure:

```text
dns.quad9.net (9.9.9.9:853)
dns.quad9.net (149.112.112.112:853)
```

and set:

```text
Forwarder Protocol: QUIC
```

For IPv6-capable environments, Quad9 also publishes IPv6 resolver addresses. Only configure IPv6 forwarders when the DNS host has working IPv6 connectivity.

The important part is that the hostname remains:

```text
dns.quad9.net
```

so Technitium can validate the TLS identity while connecting directly to the configured resolver IP.

### Example: AdGuard DNS over DoQ

AdGuard's standard public DNS service supports DNS-over-QUIC using:

```text
dns.adguard-dns.com
```

with standard resolver addresses:

```text
94.140.14.14
94.140.15.15
```

Example Technitium entries:

```text
dns.adguard-dns.com (94.140.14.14:853)
dns.adguard-dns.com (94.140.15.15:853)
```

with:

```text
Forwarder Protocol: QUIC
```

AdGuard also provides other resolver profiles such as family-filtering and unfiltered services. Use the hostname and addresses documented for the specific service profile being selected.

### Do not mix up inbound and outbound DoQ

These are separate features:

```text
Technitium -> Public Resolver
```

uses the **Forwarders** configuration.

```text
Client -> Technitium
```

uses the **Optional Protocols** configuration described in the next section.

An administrator who only wants encrypted upstream DNS does **not** need to expose UDP/853 inbound to clients.

### Firewall requirements for upstream DoQ

If UFW uses its normal default outbound policy of `allow`, no additional outbound rule is usually required.

Check:

```bash
sudo ufw status verbose
```

If outbound traffic is restricted, allow UDP/853 to the selected resolver addresses.

Example for Quad9:

```bash
sudo ufw allow out to 9.9.9.9 port 853 proto udp
sudo ufw allow out to 149.112.112.112 port 853 proto udp
```

Example for AdGuard:

```bash
sudo ufw allow out to 94.140.14.14 port 853 proto udp
sudo ufw allow out to 94.140.15.15 port 853 proto udp
```

### Verify that DoQ is actually being used

Packet capture is the clearest transport-level check:

```bash
sudo tcpdump -ni <INTERFACE> 'udp port 853'
```

Then generate a fresh DNS request through Technitium:

```bash
dig @127.0.0.1 example.com A
```

or from a LAN client:

```bash
dig @<DNS_VIRTUAL_IP> example.com A
```

You should see traffic similar to:

```text
<DNS_NODE_IP>.<EPHEMERAL_PORT> > <UPSTREAM_DNS_IP>.853: UDP
<UPSTREAM_DNS_IP>.853 > <DNS_NODE_IP>.<EPHEMERAL_PORT>: UDP
```

That is expected for DoQ. The DNS payload itself is carried inside encrypted QUIC traffic, so `tcpdump` sees UDP packets but should not decode the upstream DNS query name as plaintext.

For Quad9 specifically, a fresh query can also be used to confirm that Quad9 is being reached:

```bash
dig @<DNS_VIRTUAL_IP> proto.on.quad9.net TXT
```

Provider-side protocol-reporting behavior can change over time, so packet capture of UDP/853 remains the most direct local verification that Technitium is using QUIC.

### Failure behavior

If Technitium is configured with:

```text
Forwarder Protocol: QUIC
```

it should be treated as a deliberate protocol choice. Do not assume a failed DoQ connection will automatically downgrade to plaintext UDP/53.

If DoQ is unreliable on a network, troubleshoot:

- outbound UDP/853 filtering;
- NAT/firewall handling of QUIC;
- `libmsquic` installation;
- system time and TLS validation;
- upstream-provider availability.

If QUIC cannot be made reliable, DNS-over-TLS is a reasonable encrypted fallback:

```text
Forwarder Protocol: TLS
TCP/853
```

---

## 4.6 Optional: Serve DNS-over-QUIC to Clients

Installing `libmsquic` enables the Linux QUIC dependency, but Technitium does not automatically force clients to use DoQ.

To provide DoQ directly to clients, enable it in the Technitium web console under the optional protocol settings and configure the required TLS certificate/name.

Client-facing DoQ requires:

```text
UDP/853 inbound
```

Add the UFW rule only if clients will connect directly to Technitium using DoQ:

```bash
sudo ufw allow 853/udp
```

This is separate from using DoQ only as an **outbound forwarder**.

---

## 4.7 Initial Technitium Configuration

Before configuring Keepalived, complete the basic DNS configuration on both nodes.

At minimum:

1. Configure the administrator credentials.
2. Configure the desired authoritative/internal DNS zones.
3. Configure upstream forwarding/recursion behavior.
4. Install `libmsquic` if DoQ will be used.
5. Configure Technitium clustering if the deployment will use the cluster feature.
6. Confirm both physical node IPs can independently answer DNS.
7. Confirm the internal zone selected for the Keepalived health check exists and returns an SOA record on both nodes.

Example checks:

```bash
dig @<DNS_NODE_A_IP> <INTERNAL_DNS_ZONE> SOA +short
dig @<DNS_NODE_B_IP> <INTERNAL_DNS_ZONE> SOA +short
```

Do not proceed to the VIP health check until both commands return successfully.

> Technitium clustering synchronizes supported DNS configuration. Keepalived remains responsible for the shared operating-system VIP and is configured separately.

---


## 4.8 Verify Time Synchronization

Accurate time is important for more than log timestamps. A significantly incorrect clock can break or destabilize:

- DNSSEC validation
- TSIG validation and zone transfers
- TLS certificate validation
- DANE/TLSA validation
- Technitium cluster communication

Before building the cluster or enabling DNSSEC validation, verify that each DNS node has a working time source.

### If using `systemd-timesyncd`

Check status:

```bash
timedatectl
timedatectl timesync-status
```

Useful indicators:

```text
System clock synchronized: yes
NTP service: active
```

If network time synchronization is disabled:

```bash
sudo timedatectl set-ntp true
```

Example sources in `/etc/systemd/timesyncd.conf`:

```ini
[Time]
NTP=time.cloudflare.com time.google.com
FallbackNTP=ntp.ubuntu.com
```

Restart and verify:

```bash
sudo systemctl restart systemd-timesyncd
timedatectl timesync-status
```

### If using `chrony`

Check synchronization:

```bash
chronyc tracking
chronyc sources -v
chronyc activity
```

Healthy output should show:

- a real reference source;
- a non-zero stratum;
- `Leap status : Normal`;
- at least one usable source, commonly marked `^*`.

Example `/etc/chrony/chrony.conf` sources:

```conf
pool ntp.ubuntu.com iburst
server time.cloudflare.com iburst
server time.google.com iburst
```

NIST is another valid option:

```conf
pool time.nist.gov iburst
```

Using more than one independent provider is preferable to depending on a single time source.

Restart and verify:

```bash
sudo systemctl restart chrony
chronyc tracking
chronyc sources -v
```

> **Bootstrap note:** If a DNS appliance boots with a badly incorrect clock and its NTP sources are configured only by hostname, it can create a circular dependency where DNSSEC fails because time is wrong and NTP cannot resolve its source because DNS is unhealthy. Consider retaining at least one reliable numeric NTP source or otherwise ensuring a dependable bootstrap path.

---

## 4.9 Configure Technitium Clustering

Technitium clustering synchronizes supported configuration between multiple Technitium DNS Server instances. It does **not** replace Keepalived/VRRP; Keepalived still controls the shared DNS VIP.

The cluster uses:

- one **Primary** cluster node;
- one or more **Secondary** cluster nodes;
- a cluster domain;
- a special cluster catalog zone;
- TSIG-secured zone transfers;
- DANE/TLSA records for authenticated node-to-node HTTPS communication.

### Before clustering

Verify on every node:

- Technitium is running normally.
- The node has a static IP address.
- DNS works independently on the physical node IP.
- System time is synchronized.
- TCP/53443 is allowed between cluster nodes if using the default Technitium HTTPS management port.
- The intended cluster domain and node naming scheme are understood before initialization.

### Initialize the Primary node

1. Log in to the Technitium web console on the node that should become the **Primary**.
2. Go to `Administration -> Cluster`.
3. Select `Initialize -> New Cluster`.
4. Enter the **Cluster Domain**.

   Example:

   ```text
   cluster.example.internal
   ```

   A private internal domain is fine for an internal DNS deployment.

5. Enter the Primary node's static IP address or addresses.
6. Complete cluster initialization.

Technitium creates and manages the cluster zone and a special cluster catalog zone. It also creates the TSIG key used to secure cluster-related zone transfers.

> Choose the cluster domain carefully. It cannot simply be renamed later without deleting and recreating the cluster.

### Join a Secondary node

1. Log in to the Technitium web console on the Secondary node.
2. Go to `Administration -> Cluster`.
3. Select `Initialize -> Join Cluster`.
4. Enter the Secondary node's static IP address or addresses.
5. Enter the **Primary Node URL** shown on the Primary node's Cluster page.
6. If the cluster domain is private and cannot yet resolve normally, provide the **Primary Node IP Address** explicitly.
7. Choose certificate validation behavior appropriate for the environment.
   - If the Primary uses a self-signed certificate during the initial join, the join may require ignoring certificate validation errors.
   - After joining, cluster nodes use DANE-EE/TLSA records for node authentication.
8. Enter an administrator username and password from the Primary node.
9. Join the cluster and allow the initial configuration synchronization to complete.

> Joining a cluster synchronizes configuration from the Primary and can overwrite settings/configuration files on the joining node.

### Verify the cluster

From `Administration -> Cluster`, confirm:

- Primary and Secondary nodes are listed;
- node state is healthy;
- the cluster secondary zone is synchronized;
- the special cluster catalog zone is synchronized.

Verify node resolution:

```bash
dig @127.0.0.1 <PRIMARY_NODE_FQDN> A +short
```

Verify the DANE TLSA record used by the cluster web service:

```bash
dig @127.0.0.1 _53443._tcp.<PRIMARY_NODE_FQDN> TLSA
```

### Add zones to the cluster catalog

Zones remain independently hosted objects in Technitium. To have a supported zone represented across the cluster, add it to the special **cluster catalog zone**.

Common examples include:

- Primary zones
- Conditional Forwarder zones
- Stub zones

Technitium then manages the corresponding member/secondary behavior on the other cluster nodes.

### Force a resync

If a Secondary falls out of sync:

1. Go to `Administration -> Cluster`.
2. Select the Secondary node with the node selector.
3. Click `Resync`.

This forces a complete cluster configuration resynchronization from the Primary.

### Cluster troubleshooting checks

If the Secondary reports TSIG, DANE, TLSA, or certificate errors:

- confirm time synchronization first;
- confirm the cluster TSIG key name, algorithm, and shared secret match;
- verify the cluster secondary zone is current;
- verify A/AAAA records for each node;
- verify TLSA records exist for the node HTTPS service;
- verify TCP/53443 connectivity between nodes.

Useful checks:

```bash
chronyc tracking
```

or:

```bash
timedatectl
```

Then:

```bash
dig @127.0.0.1 <PRIMARY_NODE_FQDN> A +short
dig @127.0.0.1 _53443._tcp.<PRIMARY_NODE_FQDN> TLSA
```

and:

```bash
sudo journalctl -fu dns
```

---

# 5. Verify the Physical Interfaces

Determine the interface carrying each node's DNS subnet address.

```bash
ip -br addr
```

Example:

```text
eth0     UP     <DNS_NODE_A_IP>/<PREFIX_LENGTH>
```

or:

```text
enp1s0   UP     <DNS_NODE_B_IP>/<PREFIX_LENGTH>
```

Record the correct interface name for each host.

The names can be different between nodes.

---

# 6. Configure the Technitium Health Check

Create the health-check script on **both nodes**:

```bash
sudo nano /usr/local/bin/check-technitium-dns.sh
```

Contents:

```bash
#!/bin/bash

/usr/bin/dig @127.0.0.1 <INTERNAL_DNS_ZONE> SOA +time=1 +tries=1 +short | /usr/bin/grep -q .
```

Set ownership and permissions:

```bash
sudo chown root:root /usr/local/bin/check-technitium-dns.sh
sudo chmod 755 /usr/local/bin/check-technitium-dns.sh
```

Test it:

```bash
/usr/local/bin/check-technitium-dns.sh
echo $?
```

Expected result while Technitium is healthy:

```text
0
```

Also test the underlying query directly:

```bash
dig @127.0.0.1 <INTERNAL_DNS_ZONE> SOA +time=1 +tries=1 +short
```

A valid SOA record should be returned.

---

# 7. Configure UFW

## 7.1 Important UFW Status Check

Do not rely only on:

```bash
systemctl status ufw
```

The systemd service may display:

```text
active (exited)
```

even when UFW itself is disabled.

Use:

```bash
sudo ufw status
```

The desired state is:

```text
Status: active
```

---

## 7.2 Required DNS Rules

At minimum, clients need TCP and UDP DNS access.

On both nodes:

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
```

SSH administration, if required:

```bash
sudo ufw allow 22/tcp
```

Technitium web console, if directly exposed to the management network:

```bash
sudo ufw allow 5380/tcp
sudo ufw allow 53443/tcp
```

Prefer restricting management ports to a trusted management subnet rather than opening them globally.

Example:

```bash
sudo ufw allow from <MANAGEMENT_SUBNET> to any port 53443 proto tcp
```

---

## 7.3 Optional Technitium Protocols

Only open these if the corresponding inbound Technitium service is enabled.

### DNS-over-TLS

```bash
sudo ufw allow 853/tcp
```

### DNS-over-QUIC

```bash
sudo ufw allow 853/udp
```

### DNS-over-HTTPS

```bash
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
```

These are **client-to-Technitium** rules.

If Technitium only uses DoQ as an outbound upstream forwarder, an inbound UDP/853 rule is not required for that purpose.

---

# 8. Configure VRRP Firewall Rules

Each node must accept VRRP advertisements from the **other node**.

## Node A

Allow VRRP from Node B:

```bash
sudo ufw allow from <DNS_NODE_B_IP> to 224.0.0.18 proto vrrp
```

## Node B

Allow VRRP from Node A:

```bash
sudo ufw allow from <DNS_NODE_A_IP> to 224.0.0.18 proto vrrp
```

Reload UFW:

```bash
sudo ufw reload
```

Verify:

```bash
sudo ufw status numbered
```

Expected conceptually:

### Node A

```text
224.0.0.18/vrrp    ALLOW IN    <DNS_NODE_B_IP>
```

### Node B

```text
224.0.0.18/vrrp    ALLOW IN    <DNS_NODE_A_IP>
```

If UFW is not yet enabled, make sure SSH access is allowed first, then enable it:

```bash
sudo ufw allow 22/tcp
sudo ufw enable
```

---

# 9. Remove an Incorrect UFW Rule

If a VRRP rule is accidentally added to the wrong host, list numbered rules:

```bash
sudo ufw status numbered
```

Delete the incorrect rule by number:

```bash
sudo ufw delete <RULE_NUMBER>
```

Alternatively, delete the exact rule:

```bash
sudo ufw delete allow from <PEER_IP> to 224.0.0.18 proto vrrp
```

Using the numbered method is generally safest when there is any ambiguity.

---

# 10. Configure Keepalived

Both nodes start in `BACKUP` state.

The higher priority determines which healthy node becomes MASTER.

This avoids hard-coding MASTER as an initial state and lets VRRP perform the election.

The health-check script intentionally has **no weight configured**.

With an unweighted tracked script:

```text
health check succeeds -> node is eligible for VRRP
health check fails    -> VRRP instance enters FAULT
```

This is preferable for DNS because a node with a failed DNS service should not continue owning the DNS VIP at a reduced priority.

---

## 10.1 Node A Keepalived Configuration

Create:

```bash
sudo nano /etc/keepalived/keepalived.conf
```

Use:

```conf
global_defs {
    router_id <DNS_NODE_A_ROUTER_ID>
    script_user root
    enable_script_security
}

vrrp_script check_technitium_dns {
    script "/usr/local/bin/check-technitium-dns.sh"
    interval 2
    timeout 2
    fall 3
    rise 2
}

vrrp_instance DNS_VIP {
    state BACKUP
    interface <DNS_NODE_A_INTERFACE>
    virtual_router_id 53
    priority 150
    advert_int 1

    virtual_ipaddress {
        <DNS_VIRTUAL_IP>/<PREFIX_LENGTH>
    }

    track_script {
        check_technitium_dns
    }
}
```

Example router ID format:

```text
DNS_NODE_A
```

---

## 10.2 Node B Keepalived Configuration

Create:

```bash
sudo nano /etc/keepalived/keepalived.conf
```

Use:

```conf
global_defs {
    router_id <DNS_NODE_B_ROUTER_ID>
    script_user root
    enable_script_security
}

vrrp_script check_technitium_dns {
    script "/usr/local/bin/check-technitium-dns.sh"
    interval 2
    timeout 2
    fall 3
    rise 2
}

vrrp_instance DNS_VIP {
    state BACKUP
    interface <DNS_NODE_B_INTERFACE>
    virtual_router_id 53
    priority 100
    advert_int 1

    virtual_ipaddress {
        <DNS_VIRTUAL_IP>/<PREFIX_LENGTH>
    }

    track_script {
        check_technitium_dns
    }
}
```

---

# 11. Keepalived Configuration Notes

The following settings must match on both nodes:

```text
virtual_router_id
advert_int
VIP
subnet prefix
health-check behavior
```

The following settings are expected to differ:

```text
router_id
interface
priority
```

Recommended priorities:

```text
Preferred Node A: 150
Backup Node B:    100
```

The exact values are not special. The important requirement is:

```text
Node A priority > Node B priority
```

Do not configure `nopreempt` if the desired behavior is for the preferred higher-priority node to reclaim the VIP after it recovers.

---

# 12. Optional: Legacy VRRP Authentication

Keepalived supports the legacy VRRP `authentication` block.

This is **optional** and is not required for the two-node design in this runbook.

A major caveat is that `auth_type PASS` does **not encrypt the password**. The value is transmitted in plaintext inside the VRRP advertisements and should not be treated as a strong security control.

For legacy VRRP password authentication, both nodes must use the same authentication configuration:

```conf
authentication {
    auth_type PASS
    auth_pass <VRRP_SHARED_PASSWORD>
}
```

Add the block inside `vrrp_instance DNS_VIP` on **both** nodes.

Example:

```conf
vrrp_instance DNS_VIP {
    state BACKUP
    interface <INTERFACE>
    virtual_router_id 53
    priority <PRIORITY>
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass <VRRP_SHARED_PASSWORD>
    }

    virtual_ipaddress {
        <DNS_VIRTUAL_IP>/<PREFIX_LENGTH>
    }

    track_script {
        check_technitium_dns
    }
}
```

Important considerations:

- The password must match on every VRRP peer.
- The legacy `PASS` mechanism sends the password in plaintext over the network.
- It helps prevent accidental participation by peers using a different password, but it should **not** be considered cryptographically secure authentication.
- Legacy VRRPv2 password authentication effectively uses only the first 8 characters of the password, so using an intentional 8-character value avoids misleading configuration.
- Do not reuse an important password or credential.
- VLAN isolation, host firewalls, and limiting VRRP traffic to expected peers remain important.

Newer Keepalived builds may provide stronger HMAC-based authentication. That is separate from the legacy `authentication { auth_type PASS ... }` mechanism and should be evaluated against the exact Keepalived version deployed.

---

# 13. Optional: Unicast VRRP

The primary configuration in this runbook uses **multicast VRRP**, and multicast should remain the default when both DNS nodes share a normal Layer-2 network and multicast works correctly.

Unicast VRRP is useful when:

- multicast is blocked by the network,
- the environment does not support multicast,
- multicast behavior is unreliable,
- the nodes communicate through an overlay or other network where multicast is undesirable,
- or explicit peer-to-peer VRRP traffic is operationally preferred.

With multicast, advertisements are sent to:

```text
224.0.0.18
```

With unicast, each Keepalived node sends VRRP advertisements directly to the configured peer IP.

Conceptually:

```text
Multicast:

<DNS_NODE_A_IP> ----\
                     >---- 224.0.0.18
<DNS_NODE_B_IP> ----/


Unicast:

<DNS_NODE_A_IP> <--------> <DNS_NODE_B_IP>
```

## 13.1 Node A Unicast Configuration

```conf
global_defs {
    router_id <DNS_NODE_A_ROUTER_ID>
    script_user root
    enable_script_security
    vrrp_check_unicast_src
}

vrrp_script check_technitium_dns {
    script "/usr/local/bin/check-technitium-dns.sh"
    interval 2
    timeout 2
    fall 3
    rise 2
}

vrrp_instance DNS_VIP {
    state BACKUP
    interface <DNS_NODE_A_INTERFACE>

    unicast_src_ip <DNS_NODE_A_IP>
    unicast_peer {
        <DNS_NODE_B_IP>
    }

    virtual_router_id 53
    priority 150
    advert_int 1

    virtual_ipaddress {
        <DNS_VIRTUAL_IP>/<PREFIX_LENGTH>
    }

    track_script {
        check_technitium_dns
    }
}
```

## 13.2 Node B Unicast Configuration

```conf
global_defs {
    router_id <DNS_NODE_B_ROUTER_ID>
    script_user root
    enable_script_security
    vrrp_check_unicast_src
}

vrrp_script check_technitium_dns {
    script "/usr/local/bin/check-technitium-dns.sh"
    interval 2
    timeout 2
    fall 3
    rise 2
}

vrrp_instance DNS_VIP {
    state BACKUP
    interface <DNS_NODE_B_INTERFACE>

    unicast_src_ip <DNS_NODE_B_IP>
    unicast_peer {
        <DNS_NODE_A_IP>
    }

    virtual_router_id 53
    priority 100
    advert_int 1

    virtual_ipaddress {
        <DNS_VIRTUAL_IP>/<PREFIX_LENGTH>
    }

    track_script {
        check_technitium_dns
    }
}
```

## 13.3 Unicast Firewall Rules

Unicast VRRP still uses **IP protocol 112**. It does not become TCP or UDP.

When using UFW, allow VRRP directly between the peer addresses.

### Node A

```bash
sudo ufw allow from <DNS_NODE_B_IP> to <DNS_NODE_A_IP> proto vrrp
```

### Node B

```bash
sudo ufw allow from <DNS_NODE_A_IP> to <DNS_NODE_B_IP> proto vrrp
```

The multicast-specific destination `224.0.0.18` is no longer required for Keepalived advertisements when operating entirely in unicast mode.

Verify the traffic with:

```bash
sudo tcpdump -ni <INTERFACE> 'ip proto 112'
```

You should see VRRP packets directly between the node addresses rather than packets addressed to `224.0.0.18`.

### Security note for unicast

Unicast changes some of the assumptions normally provided by on-link multicast VRRP. Keep peer-specific firewall rules in place and do not expose protocol 112 broadly.

If the deployed Keepalived version supports modern HMAC authentication, it is worth evaluating for unicast deployments where stronger peer authentication is desired.

---

# 14. Validate the Keepalived Configuration

On each node:

```bash
sudo keepalived --config-test
```

Then enable and restart Keepalived:

```bash
sudo systemctl enable keepalived
sudo systemctl restart keepalived
```

Watch logs:

```bash
sudo journalctl -fu keepalived
```

Expected normal state:

```text
Node A:
VRRP_Script(check_technitium_dns) succeeded
Entering BACKUP STATE
Entering MASTER STATE
```

Node B should normally show:

```text
VRRP_Script(check_technitium_dns) succeeded
Entering BACKUP STATE
```

Node B should remain BACKUP while Node A is healthy.

---

# 15. Verify VIP Ownership

## Node A

```bash
ip -4 addr show dev <DNS_NODE_A_INTERFACE> | grep <DNS_VIRTUAL_IP>
```

Expected:

```text
<DNS_VIRTUAL_IP>
```

## Node B

```bash
ip -4 addr show dev <DNS_NODE_B_INTERFACE> | grep <DNS_VIRTUAL_IP>
```

Expected:

```text
no output
```

Only one node should own the VIP at a time.

---

# 16. Verify VRRP Traffic

VRRP uses IP protocol 112.

On either node:

```bash
sudo tcpdump -ni <INTERFACE> 'ip proto 112'
```

A healthy MASTER should generate advertisements similar to:

```text
<DNS_NODE_A_IP> > 224.0.0.18:
VRRPv2, Advertisement, vrid 53, prio 150
```

During normal steady state, the BACKUP should primarily receive the MASTER advertisements rather than continuously advertise itself as MASTER.

If **both nodes advertise continuously**, investigate:

- UFW/firewall rules
- multicast delivery
- wrong VRRP ID
- mismatched configuration
- incorrect interface
- duplicate VIP/VRRP deployment

---

# 17. Why tcpdump Can See VRRP While Keepalived Cannot

A useful troubleshooting detail:

```text
tcpdump sees packet
        |
        v
host firewall processes packet
        |
        +---- ACCEPT ---> Keepalived receives it
        |
        +---- DROP -----> Keepalived never receives it
```

Therefore:

```bash
sudo tcpdump -ni <INTERFACE> 'ip proto 112'
```

showing advertisements does **not** by itself prove that the host firewall permits those advertisements to reach Keepalived.

If one node repeatedly promotes itself to MASTER despite receiving the peer's packets in tcpdump, check UFW first:

```bash
sudo ufw status
sudo ufw status numbered
```

Also inspect the underlying rules if required:

```bash
sudo nft list ruleset
sudo iptables -S
```

---

# 18. Test DNS Through the VIP

From a third client:

```bash
dig @<DNS_VIRTUAL_IP> <INTERNAL_DNS_ZONE> SOA
```

Test an internal record:

```bash
dig @<DNS_VIRTUAL_IP> <INTERNAL_HOSTNAME> A +short
```

Test recursive/public resolution:

```bash
dig @<DNS_VIRTUAL_IP> example.com A
```

Test DNS over TCP:

```bash
dig @<DNS_VIRTUAL_IP> example.com A +tcp
```

Compare both physical nodes and the VIP:

```bash
dig @<DNS_NODE_A_IP> <INTERNAL_DNS_ZONE> SOA +short
dig @<DNS_NODE_B_IP> <INTERNAL_DNS_ZONE> SOA +short
dig @<DNS_VIRTUAL_IP> <INTERNAL_DNS_ZONE> SOA +short
```

---

# 19. Perform a Real DNS-Service Failover Test

The most important test is to stop **Technitium**, not Keepalived.

Start a continuous query from a third client:

```bash
while true; do
    dig @<DNS_VIRTUAL_IP> example.com +time=1 +tries=1 +short | head -1
    sleep 1
done
```

On the preferred MASTER node:

```bash
sudo systemctl stop dns
```

Watch Keepalived:

```bash
sudo journalctl -fu keepalived
```

After the configured health-check failures, Node A should show:

```text
VRRP_Script(check_technitium_dns) failed
(DNS_VIP) Entering FAULT STATE
```

With:

```conf
interval 2
fall 3
```

the failed health check normally requires approximately three consecutive failures before the node becomes FAULT.

Node B should then show:

```text
(DNS_VIP) Entering MASTER STATE
```

Verify that Node B owns the VIP:

```bash
ip -4 addr show dev <DNS_NODE_B_INTERFACE> | grep <DNS_VIRTUAL_IP>
```

Restart Technitium on Node A:

```bash
sudo systemctl start dns
```

With:

```conf
rise 2
```

the health check must succeed twice before the node becomes healthy again.

The preferred node should then:

```text
health check succeeds
        |
        v
enters BACKUP
        |
        v
higher priority wins election
        |
        v
becomes MASTER
```

Node B should return to:

```text
BACKUP
```

---

# 20. Expected Steady-State Behavior

Normal operation:

```text
Node A
  Technitium: Healthy
  Keepalived: MASTER
  Priority:   150
  VIP:        Present

Node B
  Technitium: Healthy
  Keepalived: BACKUP
  Priority:   100
  VIP:        Absent
```

Node A DNS failure:

```text
Node A
  Technitium: Failed
  Keepalived: FAULT
  VIP:        Removed

Node B
  Technitium: Healthy
  Keepalived: MASTER
  VIP:        Present
```

Node A recovery:

```text
Node A
  Technitium: Healthy
  Priority:   150
  Keepalived: MASTER
  VIP:        Present

Node B
  Technitium: Healthy
  Priority:   100
  Keepalived: BACKUP
  VIP:        Absent
```

---

# 21. Troubleshooting

## Both Nodes Become MASTER

Check VRRP traffic:

```bash
sudo tcpdump -ni <INTERFACE> 'ip proto 112'
```

Check UFW:

```bash
sudo ufw status
sudo ufw status numbered
```

Each host must allow VRRP from its peer.

Node A:

```bash
sudo ufw allow from <DNS_NODE_B_IP> to 224.0.0.18 proto vrrp
```

Node B:

```bash
sudo ufw allow from <DNS_NODE_A_IP> to 224.0.0.18 proto vrrp
```

---

## Keepalived Health Check Fails

Test manually:

```bash
/usr/local/bin/check-technitium-dns.sh
echo $?
```

Then test DNS itself:

```bash
dig @127.0.0.1 <INTERNAL_DNS_ZONE> SOA +time=1 +tries=1 +short
```

Check Technitium:

```bash
sudo systemctl status dns
sudo journalctl -u dns -n 100 --no-pager
```

---

## VIP Exists on Both Nodes

Check:

```bash
ip -4 addr show
```

Verify:

- both nodes use the same `virtual_router_id`
- priorities differ
- VRRP is allowed through UFW
- both nodes can receive `224.0.0.18`
- only one Keepalived deployment is running per node

---

## UFW systemd Service Says Active but `ufw status` Says Inactive

This is possible.

Use:

```bash
sudo ufw status
```

as the operational check.

If needed:

```bash
sudo ufw enable
```

Make sure SSH and other required access rules are present **before** enabling UFW remotely.

---

# 22. Useful Operational Commands

Check Technitium:

```bash
sudo systemctl status dns
```

Check Keepalived:

```bash
sudo systemctl status keepalived
```

Follow Keepalived logs:

```bash
sudo journalctl -fu keepalived
```

Check UFW:

```bash
sudo ufw status numbered
```

Check VIPs:

```bash
ip -4 addr show
```

Capture VRRP:

```bash
sudo tcpdump -ni <INTERFACE> 'ip proto 112'
```

Test local health:

```bash
/usr/local/bin/check-technitium-dns.sh
echo $?
```

Test the VIP:

```bash
dig @<DNS_VIRTUAL_IP> example.com
```

---

# 23. Firewall Summary

Minimum inbound services typically required:

| Purpose | Protocol | Port / IP Protocol | Source |
| --- | --- | --- | --- |
| DNS | UDP | 53 | Client networks |
| DNS | TCP | 53 | Client networks |
| VRRP | IP | 112 | Peer DNS node |
| SSH | TCP | 22 | Management network |
| Technitium HTTP UI | TCP | 5380 | Management network, if used |
| Technitium HTTPS UI | TCP | 53443 | Management network, if used |

Optional inbound services:

| Purpose | Protocol | Port |
| --- | --- | --- |
| DNS-over-TLS | TCP | 853 |
| DNS-over-QUIC | UDP | 853 |
| DNS-over-HTTPS | TCP | 443 |
| DNS-over-HTTPS/HTTP3 | UDP | 443 |

VRRP-specific rules:

### Node A

```bash
sudo ufw allow from <DNS_NODE_B_IP> to 224.0.0.18 proto vrrp
```

### Node B

```bash
sudo ufw allow from <DNS_NODE_A_IP> to 224.0.0.18 proto vrrp
```

If outbound firewall policy is restrictive and Technitium uses DNS-over-QUIC to an upstream resolver, allow the required outbound UDP/853 traffic separately.

---

# 24. Final Validation Checklist

- [ ] Both Technitium instances are healthy.
- [ ] Technitium configuration is synchronized as intended.
- [ ] Both nodes have static/reserved addresses.
- [ ] The VIP is reserved and unused elsewhere.
- [ ] Keepalived is installed on both nodes.
- [ ] `dig` is installed on both nodes.
- [ ] Health-check script returns `0` on both healthy nodes.
- [ ] UFW reports `Status: active` if host firewalling is intended.
- [ ] UDP/53 and TCP/53 are allowed from client networks.
- [ ] VRRP protocol 112 is allowed from each peer.
- [ ] Node A remains MASTER during normal operation.
- [ ] Node B remains BACKUP during normal operation.
- [ ] Only Node A owns the VIP during normal operation.
- [ ] Stopping Technitium on Node A places it into Keepalived FAULT state.
- [ ] Node B acquires the VIP after Node A DNS failure.
- [ ] Clients continue resolving through the VIP during failover.
- [ ] Restarting Technitium on Node A allows it to reclaim MASTER due to its higher priority.
- [ ] Node B returns to BACKUP after Node A recovery.

---

# 25. Reference Behavior

The design intentionally uses:

```conf
state BACKUP
```

on both nodes and relies on priority for MASTER election.

The design also intentionally omits a `weight` from the health check so that a failed Technitium DNS service makes that VRRP instance enter `FAULT` rather than simply reducing its election priority.

For a two-node DNS service, this produces straightforward semantics:

```text
DNS healthy   = eligible to own VIP
DNS unhealthy = not eligible to own VIP
```

This is generally preferable to allowing a DNS node with a failed resolver service to retain a reduced but nonzero VRRP priority.

---

# 26. References

Abbreviated references used by this runbook:

- **Technitium DoQ** — Technitium Blog, *Configuring DNS-over-QUIC and HTTPS/3 For Technitium DNS Server*.
- **Technitium Forwarder Address Format** — Technitium DNS Server API documentation, *Name Server Address Format*.
- **Quad9 DoQ** — Quad9, *Quad9 Enables DNS Over HTTP/3 and DNS Over QUIC*.
- **AdGuard DoQ** — AdGuard DNS Knowledge Base, *Known DNS Providers*.
- **Technitium Clustering** — Technitium Blog, *Understanding Clustering And How To Configure It*.
- **Technitium v14** — Technitium Blog, *Technitium DNS Server v14 Released!*.
- **Technitium Catalog Zones** — Technitium Blog, *Technitium DNS Server v13 Released!*.
- **systemd time sync** — Debian `timedatectl(1)`, `systemd-timesyncd(8)`, and `timesyncd.conf(5)`.
- **chrony** — chrony documentation for `chronyc`, source status, tracking, and `chrony.conf`.
- **Keepalived/VRRP** — Keepalived documentation for VRRP, tracked scripts, multicast/unicast, and IPVS.
