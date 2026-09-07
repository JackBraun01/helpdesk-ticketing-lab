# Helpdesk Ticketing Lab: TICKET01

A self-hosted GLPI ticketing system built in VirtualBox and integrated with the existing `lab.local` Active Directory environment, simulating a small company's IT service desk — deployment, LDAP integration (upgraded to encrypted LDAPS with a real internal CA), and a realistic ticket set drawn from genuine troubleshooting encountered while building the environment.

**Status: complete**, including the LDAPS stretch goal — see Section 10 for the full write-up and Issues 7-11 in the Troubleshooting Log.

This project extends [active-directory-lab](../active-directory-lab) — TICKET01 joins DC01 and CLIENT01 on the same isolated network, authenticating against the same domain.

## Project Goal

To add a service-desk layer on top of an existing enterprise IT environment: deploy an open-source ITSM tool, integrate it with Active Directory for real user authentication, and populate it with a realistic ticket history — including tickets drawn directly from genuine issues hit while building the infrastructure itself.

## Environment Overview

| Component | Details |
|---|---|
| Hypervisor | Oracle VirtualBox 7.2.16 |
| Ticketing Server | `TICKET01`, Ubuntu Server 26.04.1 LTS (Resolute Raccoon), minimized install |
| Ticketing Software | GLPI (official `glpi/glpi` Docker image) + MySQL, via Docker Compose |
| Domain | Joins existing `lab.local` forest (see active-directory-lab) |
| Networking | Dual-homed: NAT (management/internet access) + isolated `LabNetwork` (same as DC01/CLIENT01) |
| TICKET01 LabNetwork IP | `192.168.1.30` (static, via netplan) |
| DNS | Points to DC01 (`192.168.1.10`) |
| Remote access | SSH from Windows PowerShell, via NAT port-forward (host `2222` → guest `22`) |
| GLPI web access | Browser via NAT port-forward (host `8080` → guest `80`) |
| LDAP Source | `lab.local`, authenticated against DC01 |

TICKET01 sits on the same isolated internal network as the existing domain controller and client, reflecting how a real internal service-desk server would sit inside a company's LAN rather than being a bolt-on SaaS tool. It is **dual-homed** — one network adapter for host management (NAT), a second dedicated purely to talking to the domain (LabNetwork) — the same pattern real servers use to separate management traffic from production traffic.

## Design Rationale

GLPI was chosen over other open-source options (osTicket, Zammad) specifically because it combines ticket management with native LDAP/Active Directory authentication — allowing genuine integration with the existing domain rather than a standalone tool with its own separate user base. This mirrors how commercial ITSM platforms commonly referenced in UK IT support job listings (ServiceNow, Freshservice) integrate with corporate directory services.

## What Was Built

### 1. TICKET01 VM provisioning
Ubuntu Server 26.04.1 LTS installed via VirtualBox unattended installation (minimized install), initially on NAT networking to allow package/image downloads.

![VM creation settings](Screenshots/doc-01_ticket01-vm-creation-settings.png)
![Booting past the fixed disk issue](Screenshots/doc-02_ticket01-vm-booting-grub-menu.png)
![Installer progressing after the CPU fix](Screenshots/doc-03_ticket01-installer-progressing-post-cpu-fix.png)
![Guided storage / LVM configuration](Screenshots/doc-04_ticket01-storage-configuration-summary.png)
![First login and hostname confirmed](Screenshots/doc-05_ticket01-first-login-hostname-confirmed.png)
![Confirming the NAT IP address](Screenshots/doc-06_ticket01-ip-a-nat-address.png)

### 2. Remote administration via SSH
OpenSSH server installed during setup; administered remotely from a Windows PowerShell client via a NAT port-forwarding rule (host `2222` → guest `22`), rather than the local VirtualBox console — reflecting standard real-world headless server management.

![SSH connection from PowerShell](Screenshots/doc-07_ticket01-ssh-connection-from-powershell.png)

### 3. Docker & GLPI deployment
Docker Engine and Docker Compose installed via Ubuntu's official package repository. GLPI deployed using the official `glpi/glpi` Docker image maintained by the GLPI project, paired with a MySQL database container, defined via the project's official `docker-compose.example.yml` and `.env.example` templates.

![Docker installed and versions confirmed](Screenshots/doc-08_ticket01-docker-installed-versions.png)
![.env configured with a real database password](Screenshots/doc-09_ticket01-nano-installed-env-configured.png)
![Both containers running](Screenshots/doc-10_ticket01-docker-compose-containers-running.png)

### 4. First GLPI access
A second NAT port-forwarding rule (host `8080` → guest `80`) exposed GLPI's web interface to the host browser. GLPI auto-installed its own database schema on first run (detected via the `.env` database variables), skipping straight to a working login screen.

![GLPI login page, first access](Screenshots/doc-11_ticket01-glpi-login-page-first-access.png)
![GLPI dashboard after first login](Screenshots/doc-12_ticket01-glpi-dashboard-first-login.png)

### 5. Dual-homed networking for LabNetwork integration
A second network adapter (`enp0s8`) was added to TICKET01, attached to the same isolated `LabNetwork` used by DC01 and CLIENT01, and given a static IP (`192.168.1.30/24`) via netplan — continuing the addressing scheme already established in the AD lab (DC01 `.10`, CLIENT01 `.20`). The original NAT adapter (`enp0s3`) was left untouched, so remote management access was never lost.

![Second network adapter detected](Screenshots/doc-14_ticket01-second-nic-detected.png)
![Static IP applied on LabNetwork](Screenshots/doc-15_ticket01-static-ip-applied.png)
![Successful ping to DC01 — confirming TICKET01 can now reach the domain controller](Screenshots/doc-16_ticket01-ping-dc01-success.png)

### 6. LDAP integration with lab.local
A dedicated least-privilege service account (`svc-glpi-ldap`) was created in a new `Service-Accounts` OU on DC01 specifically for this integration, rather than reusing an existing personal account — limiting what an attacker could do if the integration itself were ever compromised. GLPI's LDAP directory was configured to bind to DC01 using this account. The connection initially failed due to DC01's default LDAP signing enforcement (see Troubleshooting Log, Issue 6); once resolved, all five test stages passed, confirming genuine authentication against the domain.

![Service-Accounts OU created on DC01](Screenshots/doc-19_dc01-service-accounts-ou-created.png)
![svc-glpi-ldap account created](Screenshots/doc-20_dc01-svc-glpi-ldap-account-created.png)
![LDAP directories list before configuration](Screenshots/doc-17_glpi-ldap-directories-empty-list.png)
![Successful LDAP test — all five stages passing](Screenshots/doc-25_glpi-ldap-test-success-after-signing-fix.png)

Next: importing jsmith, Sarah Jones, and Mike Brown from their respective OUs as real GLPI users.

![jsmith, mbrown, and Sarahjones selected for import](Screenshots/doc-27_glpi-ldap-import-users-selected.png)
![Import confirmation — all three added successfully](Screenshots/doc-28_glpi-users-import-confirmation.png)
![GLPI Users list showing the three real AD accounts alongside GLPI's defaults](Screenshots/doc-29_glpi-users-list-post-import.png)

### 7. Ticket categories & templates
Five categories created matching real IT support taxonomy, giving tickets a consistent way to be sorted and reported on: Hardware, Software, Network, Account & Access, and Active Directory (the last one specifically for domain/infrastructure-related tickets, tying back to the original AD lab's troubleshooting log).

![Empty ITIL categories list](Screenshots/doc-30_glpi-ticket-categories-before.png)
![Five categories created](Screenshots/doc-31_glpi-ticket-categories-created.png)

### 8. Realistic ticket population
Nine tickets created across all three imported AD users and all five categories, mixing genuine technical incidents drawn from this project's own troubleshooting log with ordinary day-to-day helpdesk volume. Three tickets (the RCU stall, the LDAP signing failure, and an account lockout) were resolved with real, specific solution text and closed — the remainder left in "Processing (assigned)" status, reflecting a realistic snapshot of an active service desk rather than a fully-wrapped demo.

| # | Title | Requester | Category | Status |
|---|---|---|---|---|
| 1 | New server (TICKET01) failed to boot after setup | jsmith | Active Directory | Closed |
| 2 | Can't get GLPI to authenticate against AD, login keeps failing | jsmith | Active Directory | Closed |
| 3 | New starter laptop keeps setting up Windows from scratch every time I turn it on | mbrown | Active Directory | Processing |
| 4 | My wallpaper policy isn't applying even though IT said they fixed it | Sarahjones | Active Directory | Processing |
| 5 | Locked out of my account after too many password attempts | mbrown | Account & Access | Closed |
| 6 | Printer in Sales not responding | Sarahjones | Hardware | Processing |
| 7 | Can't connect to the office Wi-Fi from my laptop | jsmith | Network | Processing |
| 8 | Need Excel installed on my new PC | mbrown | Software | Processing |
| 9 | New starter needs a Finance-Department account setup | Sarahjones | Account & Access | Processing |

![All nine tickets created](Screenshots/doc-32_glpi-all-nine-tickets-created.png)
![Ticket 1 detail view showing the solution text and closed status](Screenshots/doc-33_glpi-ticket1-solution-and-closed.png)
![Final ticket list — a realistic mix of Closed and Processing](Screenshots/doc-34_glpi-tickets-mixed-status-final.png)

### 9. Asset inventory
GLPI's asset management side was populated with the three actual machines in this lab (DC01, CLIENT01, TICKET01) rather than leaving it empty — since GLPI is an IT asset manager as much as a ticketing tool, an empty inventory would have undersold that half of the product. Ticket #3 was linked directly to its real subject asset (CLIENT01) via the ticket's Items tab, demonstrating GLPI's asset-ticket relationship rather than tickets existing as disconnected text records.

![Three lab machines added as tracked computer assets](Screenshots/doc-36_glpi-computers-assets-added.png)
![Ticket linked to its real asset via the Items tab](Screenshots/doc-37_glpi-ticket-linked-to-asset.png)

GLPI's built-in demo data was also disabled, so the dashboard reflects only genuine project data rather than the thousands of placeholder assets and tickets GLPI ships with by default.


### 10. LDAPS (stretch goal): replacing the signing workaround with real encryption

The original LDAP integration (Section 6) worked over plain, unencrypted LDAP on port 389, only after relaxing DC01's LDAP signing requirements (see Troubleshooting Log, Issue 6). That was flagged at the time as a deliberate lab simplification rather than a production-correct fix. This stretch goal replaces it with genuine encrypted LDAP (LDAPS, port 636), backed by a real internal Certificate Authority — the fix an actual production environment would require.

**Steps completed:**
- Installed and configured **Active Directory Certificate Services (AD CS)** on DC01 as an Enterprise Root CA (`lab-DC01-CA`)
- DC01 auto-enrolled itself a valid Domain Controller certificate (`CN=DC01.lab.local`)
- Confirmed port 636 listening on DC01 (`netstat -an | findstr :636`)
- Confirmed TICKET01 could reach it (`nc -zv 192.168.1.10 636` succeeded)
- In GLPI's LDAP directory settings, updated **Port** to `636` and prefixed the **Server** field with `ldaps://` to switch the connection from plain LDAP to LDAPS
- Left **Use TLS** (Advanced information tab) set to **No** — this field is for the separate StartTLS method on port 389, and is mutually exclusive with the `ldaps://` prefix method used here

This surfaced a new, genuinely instructive failure (see Troubleshooting Log, Issue 7), currently being worked through.

![Confirming the exported certificate — self-issued, valid 2026-2031](Screenshots/doc-40_dc01-lab-ca-certificate-properties-valid.png)

Getting the certificate off DC01 required VirtualBox's shared clipboard, set to Bidirectional — the first paste attempt landed directly at the bash prompt instead of inside an open editor (see Troubleshooting Log, Issue 8), corrected by opening `nano` first and pasting into it.

![Certificate correctly pasted inside nano, ready to save as lab-DC01-CA.pem](Screenshots/doc-41_ticket01-cert-pasted-correctly-in-nano.png)
![File saved and verified with cat; docker ps confirms the GLPI container name (glpi-glpi-1)](Screenshots/doc-42_ticket01-cert-verified-docker-ps-container-name.png)
![TLS_CACERT successfully added to the container's ldap.conf, confirmed with cat](Screenshots/doc-43_ticket01-tls-cacert-configured-in-container.png)

Trusting the CA alone wasn't enough — the container also couldn't resolve DC01 by hostname, and GLPI's Server field was pointing at DC01's IP address rather than the hostname on its certificate, causing a TLS hostname-verification mismatch (see Troubleshooting Log, Issues 9-10). Adding a direct `/etc/hosts` entry inside the container and switching the Server field to `ldaps://dc01.lab.local` resolved it completely.

![All five LDAPS test stages passing: TCP stream, Base DN, LDAP URI, Bind connection, and a 50-entry search](Screenshots/doc-44_glpi-ldaps-test-all-stages-passed.png)

GLPI now authenticates against `lab.local` over genuinely encrypted LDAPS (port 636), backed by a real internal CA — replacing the original plain-LDAP-with-relaxed-signing workaround from Section 6 with the production-correct fix.

**Known limitation:** the container-side changes (`TLS_CACERT` in `/etc/ldap/ldap.conf`, the `/etc/hosts` entry) were applied live via `docker exec` and are not yet persisted anywhere in `docker-compose.yml` or a mounted volume. They will be lost if the `glpi` container is ever rebuilt from scratch (`docker compose down && up`, not just `restart`). Baking these into the compose file (a config volume mount plus an `extra_hosts` entry) is a natural next step to make the fix durable rather than a one-off runtime patch.

**Update — made persistent.** The runtime-only fix above was replaced with a proper `docker-compose.yml` configuration: a bind-mounted `./ldap-config` folder (containing `lab-DC01-CA.pem` and a small `ldap.conf` with the `TLS_CACERT` directive) mounted read-only to `/etc/ldap` inside the container, plus an `extra_hosts` entry mapping `dc01.lab.local` to `192.168.1.10` so the container doesn't need to rely on Docker's own DNS resolution. This was verified against a full container rebuild — not just a restart — to prove it genuinely survives recreation rather than just surviving a soft restart:

![docker-compose.yml down/up rebuild, followed by getent hosts and cat ldap.conf both returning correct values with zero manual docker exec fixes](Screenshots/doc-46_ticket01-persistence-verified-after-full-rebuild.png)

With that confirmed, the LDAPS stretch goal is complete: GLPI authenticates against `lab.local` over genuinely encrypted LDAP on port 636, backed by a real internal CA (`lab-DC01-CA`), with the entire trust and hostname-resolution configuration defined declaratively in `docker-compose.yml` rather than living only inside a running container's memory. This replaces the original plain-LDAP-with-relaxed-signing-requirements workaround from Section 6 with the actual production-correct fix that workaround was standing in for.

![Final LDAPS test: all five stages passing, connecting via dc01.lab.local, running entirely on the persistent docker-compose configuration](Screenshots/doc-47_glpi-ldaps-final-confirmation-with-persistence.png)

## Troubleshooting Log

Real issues encountered and resolved during the build, included deliberately — diagnosing and fixing genuine problems, not just following steps that work first time, is what actually demonstrates capability.

| # | Issue | Root Cause | Resolution |
|---|---|---|---|
| 1 | TICKET01 VM registered in VirtualBox but failed to boot with `VERR_FILE_NOT_FOUND`, "Could not open the medium" | The virtual hard disk (`.vdi`) file never actually finished being created during VM creation, despite the VM's configuration files registering successfully — a silent failure in disk provisioning rather than a visible error at creation time | Removed the broken VM registration (Remove → Delete all files), recreated the VM, and explicitly inspected the "Specify virtual hard disk" panel before clicking Finish to confirm the disk path, size, and type were populated correctly before proceeding |
| 2 | TICKET01 boot hung indefinitely at "Loading essential drivers" during Ubuntu 26.04.1 install, with no further progress after 10+ minutes | Kernel-level RCU stall (`rcu: INFO: rcu_preempt detected stalls on CPUs/tasks`) caused by a timing/scheduling desync between VirtualBox 7.2.x and multi-core Linux guests — matched to a known, currently open issue on VirtualBox's public GitHub tracker affecting the same version family | Reduced the VM's allocated CPU count from 2 to 1 (Settings → System → Processor). VM booted cleanly immediately afterward. Before concluding this was the cause, ruled out: a corrupted ISO download (verified via SHA256 checksum against Canonical's official published hash — confirmed exact match), incorrect hardware virtualization/acceleration settings, and host-level CPU/disk resource contention (host CPU utilisation was only 22% at time of hang) |
| 3 | `docker compose up -d` failed immediately with `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock` | Docker's daemon is controlled by root and communicates through a restricted Unix socket. A freshly created non-root user is not automatically granted access to that socket — membership of the `docker` group has to be explicitly assigned | Added the user to the `docker` group with `sudo usermod -aG docker $USER`, then fully logged out and reconnected via SSH, since group membership changes only take effect on a new login session rather than the current one. Confirmed fixed by successfully re-running `docker compose up -d`, which pulled both images and started the `glpi` and `db` containers |
| 4 | Browser could not reach GLPI at `http://localhost` (`ERR_CONNECTION_REFUSED`), despite `curl -I http://localhost` succeeding *inside* the VM | The only NAT port-forwarding rule configured was for SSH (port 22). Port 80 (GLPI's web server) had no forwarding rule, so the host's browser had no route in, even though the container itself was working correctly | Added a second NAT port-forwarding rule (host `8080` → guest `80`) in VirtualBox's Network settings. GLPI became reachable at `http://localhost:8080` |
| 5 | Second network adapter's static IP (`192.168.1.30`) failed to apply; `netplan apply` errored with `unknown key 'enp0s8'` | While editing the netplan YAML file in `nano`, the existing file content was accidentally overwritten rather than appended to, leaving only the new block with incorrect indentation (not nested under `network: / ethernets:`) | Rebuilt `/etc/netplan/50-cloud-init.yaml` from scratch with the full, correctly-nested structure for both `enp0s3` and `enp0s8`. Verified the exact file structure with `cat -A` (showing whitespace/line-endings explicitly) before reapplying, rather than trusting the editor's on-screen display. A follow-on `chmod 600` permissions fix briefly blocked the verifying user's own read access to the file; resolved by reading it with `sudo` |
| 6 | GLPI's LDAP connection test failed with `Authentication failed: Strong(er) authentication required(8)` at the Bind connection stage | Windows Server 2025 (DC01's OS) enforces LDAP server signing requirements by default, rejecting unsigned/unencrypted bind attempts over plain LDAP (port 389) — a stricter default than older Windows Server versions used. Network connectivity (`nc -zv 192.168.1.10 389`) and the account credentials were both confirmed correct beforehand, isolating the cause specifically to this security policy | Relaxed **"Domain controller: LDAP server signing requirements"** to `None` and **"...Enforcement"** to `Disabled` via the Default Domain Controllers Policy in Group Policy Management, then forced the change with `gpupdate /force`. This is a deliberate, documented simplification appropriate for an isolated lab without a certificate authority — the production-correct fix would be LDAPS (encrypted LDAP over port 636) with a properly issued certificate, which was considered but deferred as a larger, separate task |
| 7 | After switching to LDAPS (`ldaps://192.168.1.10`, port 636), GLPI's connection test passed **TCP stream**, **Base DN**, and **LDAP URI** stages, then failed at **Bind connection** with `Authentication failed: Can't contact LDAP server(-1)` | The raw TCP socket to port 636 opens fine (which is why the earlier stages pass), but the actual TLS handshake is rejected during the bind attempt because the OpenLDAP client library used by PHP/GLPI — running inside the `glpi` Docker container, which has its own isolated filesystem and certificate trust store — has never been told to trust the newly created `lab-DC01-CA` root certificate. The container's default CA bundle only contains public, well-known root CAs, not this lab's private internal CA | Exported `lab-DC01-CA`'s root certificate from DC01 in Base64 (PEM) format, copied it into the `glpi` container's filesystem, and registered it as trusted via `TLS_CACERT` in the container's `/etc/ldap/ldap.conf` (see Issues 8-10 for the full path to a working fix) |
| 8 | Pasting the exported certificate text into TICKET01's SSH session landed the raw Base64 content directly at the bash `$` prompt instead of inside a file, after fixing VirtualBox's shared clipboard direction (DC01 → host) | The clipboard paste itself worked correctly, but `nano lab-DC01-CA.pem` had not been opened first — pasting multi-line text straight at an active shell prompt causes bash to interpret each line as a separate command rather than storing it as file content | No damage occurred — each garbled Base64 line simply returned a harmless "command not found," since none matched a real command. Fixed by pressing `Ctrl+C` to clear the prompt, opening the target file with `nano lab-DC01-CA.pem` first, then pasting the still-available clipboard content into the open editor before saving |
| 9 | `docker exec -it glpi-glpi-1 mkdir -p /etc/ldap` failed with `Permission denied`; later, running `docker cp`/`docker exec` commands from inside an already-open container shell failed with `bash: docker: command not found` | `docker exec` connects as the container image's default user, `www-data`, which lacks permission to create directories under `/etc`. Separately, `docker` is a host-level CLI tool — it isn't installed inside the container itself, so it can't be run from a shell that's already inside the container | Exited back to TICKET01's own host shell, then re-ran the container commands with `docker exec -u root glpi-glpi-1 ...` to gain root privileges inside the container for the `mkdir` and the `TLS_CACERT` configuration step |
| 10 | After successfully adding `TLS_CACERT` inside the container, GLPI's LDAPS test still failed at **Bind connection** with the identical `Authentication failed: Can't contact LDAP server(-1)` | The CA is now trusted, ruling that cause out — the remaining suspect is a hostname/certificate mismatch. GLPI's Server field connects via DC01's IP address (`ldaps://192.168.1.10`), but the certificate's subject is `CN=DC01.lab.local`. TLS hostname verification compares the connection address against the certificate's name, and an IP-vs-hostname mismatch fails that check regardless of CA trust. Confirming this, `docker exec -it glpi-glpi-1 getent hosts dc01.lab.local` returned nothing — the container couldn't resolve the hostname at all, since Docker containers use their own DNS resolution separate from the host VM's netplan/DNS settings | Added a direct entry to the container's `/etc/hosts` (`docker exec -u root glpi-glpi-1 bash -c 'echo "192.168.1.10 dc01.lab.local" >> /etc/hosts'`) so the hostname would resolve, then changed GLPI's Server field to `ldaps://dc01.lab.local` to match the certificate's subject name. All five test stages passed on retest, including a full directory search |
| 11 | While rewriting `docker-compose.yml` to persist the LDAPS fix, a `cat > file << 'EOF'` heredoc paste produced a corrupted file with the YAML duplicated and a garbled line in the middle | The heredoc's closing `EOF` was pasted with no line break before a second, duplicate copy of the same command — merging into one line (`EOFcat > docker-compose.yml << 'EOF'`). Since a heredoc terminator must appear alone on its own line, bash didn't recognize this as the end of input, so it kept treating everything afterward (the garbled line, then a second full copy of the YAML) as file content | Deleted the corrupted file (`rm docker-compose.yml`) and re-ran the exact same heredoc as a single, uninterrupted paste, confirming the result with `cat` before proceeding — one clean copy of the YAML, no duplication |

## Evidence

Screenshots are split into two numbered sequences for clarity:
- `doc-NN` — standard build steps, working as expected
- `ts-NN` — errors, warnings, or troubleshooting steps and their fixes

Full-resolution screenshots are in the `screenshots` folder; the images above and below are embedded directly from there.

### Issue 1: Missing virtual disk file

![VirtualBox boot failure — VERR_FILE_NOT_FOUND](Troubleshooting/ts-01_ticket01-boot-failure-vdi-not-found.png)
![File Explorer confirming the .vdi never existed](Troubleshooting/ts-02_ticket01-folder-missing-vdi-confirmed.png)
![Removing the broken VM registration](Troubleshooting/ts-03_ticket01-remove-context-menu.png)

### Issue 2: RCU stall on multi-core boot

Before changing the CPU count, hardware acceleration settings and host resource usage were checked and ruled out:

![Acceleration settings checked and confirmed fine](Troubleshooting/ts-04_ticket01-acceleration-settings-check.png)
![ISO file size checked](Troubleshooting/ts-05_ticket01-iso-file-properties.png)
![Host CPU/disk usage confirmed not the bottleneck](Troubleshooting/ts-06_host-performance-cpu-disk-check.png)

Confirmed the ISO itself was not corrupted by comparing its SHA256 checksum directly against Canonical's officially published hash for `ubuntu-26.04.1-live-server-amd64.iso` — an exact match, ruling out download corruption as the cause. The actual fix:

![CPU count reduced from 2 to 1](Troubleshooting/ts-07_ticket01-processor-count-reduced-to-1.png)

### Issue 3: Docker permission denied

A single screenshot captures both the original error and the successful fix in one continuous terminal scrollback:

![Docker permission denied error](Troubleshooting/ts-08_ticket01-docker-permission-denied.png)
![Fixed and containers running after group membership + reconnect](Screenshots/doc-10_ticket01-docker-compose-containers-running.png)

### Issue 4: Browser connection refused (missing port forward)

![Browser refusing to connect on port 80](Troubleshooting/ts-09_ticket01-browser-connection-refused-port80.png)

### Issue 5: netplan static IP misconfiguration

![Broken YAML structure revealed with cat -A](Troubleshooting/ts-10_ticket01-netplan-yaml-indentation-check.png)

### Issue 6: GLPI LDAP authentication failure — LDAP signing enforcement

![Generic test result showing the initial failure](Troubleshooting/ts-12_glpi-ldap-test-failed-generic-error.png)
![Port 389 connectivity confirmed, ruling out a network cause](Troubleshooting/ts-13_ticket01-ldap-port-connectivity-check.png)
![LDAP signing requirements relaxed on DC01 via Group Policy](Screenshots/doc-24_dc01-ldap-signing-requirement-relaxed.png)
![All five test stages passing after the fix](Screenshots/doc-25_glpi-ldap-test-success-after-signing-fix.png)

### Issue 7: LDAPS bind fails — container doesn't trust the new internal CA

![LDAPS test — TCP/BaseDN/URI pass, Bind fails: Can't contact LDAP server(-1)](Troubleshooting/ts-15_glpi-ldaps-bind-failed-cant-contact-server.png)

Resolved across Issues 8-10 below — see doc-44 at the end of this section for the final passing result.

### Issue 8: Certificate pasted at the shell prompt instead of into a file

![Pasted certificate landing directly at the bash prompt](Troubleshooting/ts-16_ticket01-cert-pasted-directly-at-shell-prompt.png)

Fixed by opening `nano lab-DC01-CA.pem` before pasting — see doc-41 above for the corrected result.

### Issue 9: non-root docker exec user, and docker not being available inside the container

![mkdir failing with Permission denied under the default www-data exec user](Troubleshooting/ts-17_ticket01-mkdir-permission-denied-non-root-exec.png)
![Attempting to run docker commands from inside the container shell itself](Troubleshooting/ts-18_ticket01-docker-command-not-found-inside-container.png)

Resolved by exiting back to the TICKET01 host shell and re-running with `docker exec -u root` — see doc-43 above for the successful result.

### Issue 10: bind still fails after adding TLS_CACERT — hostname vs certificate mismatch

![Retest after TLS_CACERT still fails with the same generic error](Troubleshooting/ts-19_glpi-ldaps-still-failing-after-cacert-added.png)

Retesting after Issue 9's fix produced the identical `Can't contact LDAP server(-1)` error — ruling out an untrusted CA as the remaining cause. The culprit: GLPI's Server field pointed to DC01's IP address (`ldaps://192.168.1.10`), but the certificate's subject is `CN=DC01.lab.local`, and TLS hostname verification failed the mismatch. Confirmed the `glpi` container couldn't even resolve `dc01.lab.local` (`getent hosts` returned nothing), added a direct `/etc/hosts` entry inside the container, then switched GLPI's Server field to the hostname. See doc-44 above for the resulting full pass across all five test stages.

### Issue 11: heredoc paste duplicated file content

![Merged EOF/cat line causing the heredoc to swallow a second, duplicate copy of the YAML](Troubleshooting/ts-20_ticket01-heredoc-paste-duplicated-content.png)

While rewriting `docker-compose.yml` to persist the LDAPS fix (adding a bind-mounted `ldap-config` volume and an `extra_hosts` entry for `dc01.lab.local`, so the config survives a full container rebuild instead of living only inside a running container), a `cat > file << 'EOF'` paste landed with the closing `EOF` merged directly onto the start of a second, duplicate paste with no line break between them. Since a heredoc terminator must appear alone on its own line, bash didn't recognize the merged line as the end of input and kept absorbing everything afterward — including a garbled line and a second full copy of the YAML — as file content, corrupting `docker-compose.yml`. Fixed by deleting the file and re-running the heredoc as a single, uninterrupted paste.

![Clean single copy of docker-compose.yml confirmed via cat, including the new ldap-config volume mount and extra_hosts entry](Screenshots/doc-45_ticket01-docker-compose-yml-clean-with-persistence.png)

---

## Skills Demonstrated

- Linux server administration (Ubuntu Server, minimized install) via remote SSH
- Docker & Docker Compose: multi-container deployment, volume persistence, `docker exec`/`docker cp` workflows, and rebuilding a stack from a clean state to verify configuration actually persists rather than just "working right now"
- LDAP/Active Directory integration: binding a third-party application to AD, moving from plain LDAP to encrypted LDAPS
- PKI fundamentals: standing up an Enterprise Root CA, issuing and exporting certificates, and diagnosing TLS trust and hostname-verification failures from first principles
- Cross-platform troubleshooting spanning Windows Server, a Linux VM, and a containerized application — three separate trust/config boundaries in a single integration
- Dual-homed network configuration (NAT + isolated internal network) via netplan
- ITSM/service-desk tooling: GLPI deployment, category/template design, realistic ticket and asset population
- Systematic troubleshooting: isolating root cause through elimination (network vs. trust vs. DNS vs. hostname matching), rather than treating a generic error message as a single problem with a single fix

## Possible Future Additions

- Bake the LDAPS trust/hostname config into the Dockerfile itself rather than a bind-mounted host folder, for a fully self-contained image
- Automated ticket creation via GLPI's REST API, to simulate a steadier stream of inbound requests rather than a fixed seed set
- Email-to-ticket ingestion (GLPI supports receiving tickets via a mailbox)
- A second LDAP source (e.g. a synced Entra ID tenant) to demonstrate multi-directory authentication
- Scheduled backups of the MySQL volume, verified with an actual restore test

## Tools Used

VirtualBox 7.2.16 · Ubuntu Server 26.04.1 LTS · Docker & Docker Compose · GLPI (official `glpi/glpi` image) · MySQL · Windows Server 2025 (AD CS / Enterprise Root CA) · OpenLDAP/OpenSSL (TLS trust & LDAPS)

## Repository Structure

```
helpdesk-ticketing-lab-ticket01/
├── docker-compose.yml     # GLPI + MySQL service definitions, including the persistent LDAPS volume mount and extra_hosts entry
├── ldap-config/
│   ├── ldap.conf          # TLS_CACERT directive trusting the internal lab CA
│   └── lab-DC01-CA.pem    # Exported root certificate from DC01's Enterprise CA
├── screenshots/           # doc-NN (build steps) and ts-NN (troubleshooting) images, numbered in chronological order
├── .env.example           # Template for required environment variables (actual .env is gitignored)
├── .gitignore
└── README.md              # This file
```

*Not included in this repo (excluded via `.gitignore`):* the real `.env` file (contains database credentials) and the actual `lab-DC01-CA.pem`/private key material from a live deployment — clone this repo for the pipeline and configuration pattern, not a working credential set.

## License

Distributed under the MIT License.

---

*This project is complete: GLPI deployed via Docker, integrated with Active Directory over encrypted LDAPS backed by a real internal CA, populated with realistic tickets and assets, and documented alongside every genuine issue hit along the way — from VirtualBox disk provisioning failures through to a TLS hostname mismatch inside a Docker container. The Troubleshooting Log above is as much the point of this project as the finished ticketing system itself.*
