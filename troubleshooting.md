# Troubleshooting — Cyberdefense Homelab

Last updated: **2026-11-21** (Europe/Paris).

This document retains the original Samba AD/Pi-hole incident and adds the Wazuh installation and recovery record. Commands target the named Linux host, generally as root. They are documented procedures, not commands executed on the homelab during this update.

**Evidence labels:** **Confirmed** means supported by recorded terminal output or an explicit user success report; **reported resolved** means described as fixed in the project handoff without the detailed successful output; **pending** means the result has not been captured. A user confirmation of recovery does not independently validate every suggested step.

Sources for the lab history are the existing repository, the earlier troubleshooting document dated 2026-10-11, the accessible 2026-11-21 conversation, and the current project handoff. Intermediate installation exchanges were not available. Upstream references validate procedures, not the state of this particular server.

## Current Incident Status

| Incident | Outcome | Remaining evidence |
|---|---|---|
| Pi-hole / Samba AD port 53 | Immediate conflict resolved; Samba and LDAP/Kerberos SRV lookups later confirmed working | Persistent separation and reboot behavior |
| Debian agent stale mount / dpkg failure | Agent 4.14.8 service confirmed active on 2026-10-11 | Exact final recovery sequence, Manager connection, and test alert |
| Indexer certificates, data path, single-node settings | Earlier blockers captured; later stack operation reported | Final configuration, certificate validation, authenticated cluster health |
| GitHub template URL | Reported resolved | Original failing URL and final successful download output unavailable |
| Wazuh Filebeat module 0.5 | Reported resolved | Archive installation and current module readback |
| Filebeat certificates and HTTP 401 / keystore | Reported resolved | Current TLS and authenticated output test |
| Dashboard `kibanaserver` account | Reported resolved; earlier recap states Dashboard access succeeded | Current service credential and authenticated connection check |
| Manager missing `/var/ossec` bind mount after reboot | Operation restored, explicitly confirmed by the user | `/etc/fstab` readback and a subsequent reboot test |
| Dashboard `AxiosError 3000` during Manager failure | Recovery reported after restoring the Manager mount | Separate API/token and `run_as` validation not captured |

The Tiny Windows VM is still **being installed**. Windows agent enrollment, Sysmon, domain join, and detection scenarios are not marked complete. See the [project overview and next steps](README.md).

## 1. Issue Overview

During the deployment of a Samba Active Directory Domain Controller on Debian 12, I encountered a DNS port conflict with an existing Pi-hole installation.

The Samba AD service failed to start because Pi-hole was already listening on port 53, which is required by Samba's internal DNS server.

**Environment:**

- Operating System: Debian 12
- Hostname: `srvads.luhoop.lan`
- Domain: `luhoop.lan`
- Server IP: `192.168.0.49`
- Remote access: Tailscale VPN
- DNS services: Pi-hole FTL and Samba AD

## 2. Problem Identification

After configuring Samba AD, I checked the service status:

```bash
systemctl status samba-ad-dc
```

The service was in a failed state:

```text
Active: failed (Result: exit-code)
```

The logs showed the following errors:

```text
Failed to listen on 0.0.0.0:53
NT_STATUS_ADDRESS_ALREADY_ASSOCIATED

Failed to bind to :::53 TCP
NT_STATUS_ADDRESS_ALREADY_ASSOCIATED

dns failed to setup interfaces
```

These errors indicated that another process was already using the DNS port.

## 3. Investigation

To identify which service was using port 53, I executed:

```bash
ss -lntup | grep ':53 '
```

The output showed:

```text
0.0.0.0:53  users:(("pihole-FTL",pid=1138))
[::]:53     users:(("pihole-FTL",pid=1138))
```

Pi-hole FTL was listening on port 53 across all IPv4 and IPv6 interfaces.

## 4. Root Cause

Both Samba AD and Pi-hole require access to DNS port 53.

Pi-hole was already bound to all network interfaces using:

- `0.0.0.0:53` — IPv4
- `[::]:53` — IPv6

As a result, Samba AD could not initialize its internal DNS service.

This prevented the domain controller from starting correctly.

## 5. Temporary Resolution

To release port 53, I stopped the Pi-hole service:

```bash
systemctl stop pihole-FTL.service
```

I then verified its status:

```bash
systemctl status pihole-FTL.service
```

Result:

```text
Active: inactive (dead)
```

This confirmed that Pi-hole had stopped successfully.

At this initial point, Samba AD had not yet been verified. The later 2026-10-11 record confirmed `samba-ad-dc` active and successful LDAP/Kerberos SRV resolution from the Debian client. The immediate conflict was therefore resolved in that session.

The original recovery checks were:

```bash
ss -lntup | grep ':53 '
systemctl reset-failed samba-ad-dc
systemctl start samba-ad-dc
systemctl status samba-ad-dc --no-pager
```

## 6. DNS Verification

Use these commands to repeat DNS verification from the Debian client:

```bash
dig @192.168.0.49 srvads.luhoop.lan A
```

```bash
dig @192.168.0.49 _ldap._tcp.luhoop.lan SRV
dig @192.168.0.49 _kerberos._tcp.luhoop.lan SRV
```

Earlier successful SRV results identified `srvads.luhoop.lan` on LDAP port 389 and Kerberos port 88. An A-record lookup also returned Docker bridge addresses `172.17.0.1` and `172.18.0.1`, alongside `192.168.0.49`. Their cleanup is unconfirmed; verify the addresses clients actually use.

## 7. Planned Permanent Solution

The long-term objective is to run Samba AD and Pi-hole without DNS port conflicts.

The planned architecture will separate their DNS listening addresses.

Samba AD will provide authoritative DNS services for the `luhoop.lan` domain, while Pi-hole will continue filtering DNS requests for other network devices.

The final configuration will be documented after successful deployment and testing.

The captured Pi-hole unit remained enabled. Stopping it alone does not prevent a conflict after reboot. Check `systemctl is-enabled pihole-FTL` and listener ownership before marking this permanent solution complete.

## 8. Lessons Learned

This issue highlighted several important aspects of Linux system administration:

- Two DNS services cannot bind to the same IP address and port simultaneously.
- Existing services should be identified before deploying new infrastructure components.
- The `ss` command is useful for identifying port conflicts.
- `systemctl` helps diagnose service failures and manage system services.
- DNS availability is essential for Active Directory and Kerberos.
- Temporary troubleshooting actions should be distinguished from permanent configuration changes.

## 9. Debian Agent: Stale Mount and Incomplete Package Installation

**Host: Debian client, not srvads. Status: local agent startup confirmed.**

Installation failed while creating directories under `/var/ossec`. The decisive mount output was:

```text
/var/ossec /dev/sda1[/home/wazuh-manager//deleted] ext4
```

The mount referenced a deleted directory. The record shows it was unmounted, `/var/ossec` recreated as a normal directory, and `dpkg --configure -a` attempted. This client-side stale mount is a different incident from the Manager's missing mount in section 15; do not apply the client's unmount procedure to the server.

The next agent installation still failed with pre-installation exit code **127** and package state `iH`. The upgrade branch invoked unavailable control binaries from an incomplete installation. A successful manual trace with the `install` argument did not validate that failing `upgrade` branch.

Following recovery/reinstallation, terminal output confirmed:

```text
wazuh-agent.service: enabled
Active: active (running) since Sun 2026-10-11 20:28:43 CEST
Starting Wazuh v4.14.8...
Completed.
```

The exact final package-recovery sequence was not captured and is not reconstructed here. Do not delete package metadata or bypass maintainer scripts based on this record.

```bash
# Debian client
dpkg-query -W -f='${Package}\t${Version}\n' wazuh-agent
systemctl status wazuh-agent --no-pager -l
tail -n 80 /var/ossec/logs/ossec.log
grep -nE 'ossec|wazuh-manager' /etc/fstab

# srvads (Manager only)
/var/ossec/bin/agent_control -l
```

Confirm `debian-client01` is connected and produces a received alert. `agent_control` belongs on the Manager; its absence on the agent is not a failed installation. See the [agent-control reference](https://documentation.wazuh.com/current/user-manual/reference/tools/agent-control.html).

## 10. Indexer: Certificates, Data Path, and Single-Node Discovery

**Host: srvads. Status: historical blockers recorded; later stack operation reported, detailed validation pending.**

### Certificate files were removed after archiving

`wazuh-certificates.tar` was successfully created before the source certificate directory was deleted. Copying from that deleted directory failed. The archive listed `node-1.pem`, `node-1-key.pem`, `root-ca.pem`, `admin.pem`, and `admin-key.pem`. The configuration expected `indexer.pem` and `indexer-key.pem`, creating an additional filename mismatch.

Inspect the existing archive without printing private-key contents:

```bash
cd /root/wazuh-install
tar -tf wazuh-certificates.tar
```

Recovery consists of extracting the required files to a protected directory and deploying the node pair under the exact configured names. Preserve any existing certificates before replacement; use the correct node identity and CA. The Indexer service must be able to read its own certificate/key, while private material remains restricted. Follow the [certificate deployment procedure](https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/step-by-step.html#deploying-certificates).

The earlier archive listing establishes recoverability, not successful deployment. Do not add the archive, keys, or service configuration containing credentials to Git.

### `path.data` did not match the prepared directory

The prepared directory was `/home/wazuh-indexer-data`, but the configuration used `/var/lib/wazuh-indexer-data`. The log reported an `AccessDeniedException` for the latter. A later write-permission check for `/home/wazuh-indexer-data` succeeded.

The intended setting is:

```yaml
path.data: /home/wazuh-indexer-data
```

Edit the existing key rather than appending a duplicate. Preserve existing index data before any path change. Confirm the effective path and directory access:

```bash
grep -nE '^[[:space:]]*path.data:' /etc/wazuh-indexer/opensearch.yml
namei -l /home/wazuh-indexer-data
runuser -u wazuh-indexer -- test -w /home/wazuh-indexer-data
```

A successful user-level permission check does not prove the service is using this path or that its service restrictions permit access.

### Incompatible discovery and bootstrap settings

The 2026-10-11 application log reported:

```text
setting [cluster.initial_master_nodes] is not allowed when [discovery.type] is set to [single-node]
```

For the intended single-node lab, retain `discovery.type: single-node` and remove the entire conflicting `cluster.initial_master_nodes` block, including its list items. Back up and review the YAML before restarting. Increasing a timeout does not fix this configuration conflict.

The last direct service output from that earlier session was `failed (Result: timeout)`. Later reports describe a working Dashboard/stack, but the intermediate successful restart and final Indexer settings are unavailable. Recheck them rather than retaining the old failure as a claim about the current state.

```bash
systemctl status wazuh-indexer --no-pager -l
journalctl -u wazuh-indexer -b -n 100 --no-pager
tail -n 100 /var/log/wazuh-indexer/wazuh-cluster.log
```

Use fresh timestamps. For a new installation, security initialization follows certificate deployment; do not rerun initialization on an existing secured cluster merely to investigate a login error. See the [Indexer initialization procedure](https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/step-by-step.html#cluster-initialization).

## 11. Filebeat: GitHub Template URL

**Status: reported resolved; the original failing URL and error output are unavailable.**

Filebeat needs the JSON alert template, not a GitHub HTML preview or a downloaded HTTP error page. Use the raw file pinned to this lab's Wazuh release:

[Wazuh 4.14.8 alert template](https://raw.githubusercontent.com/wazuh/wazuh/v4.14.8/extensions/elasticsearch/7.x/wazuh-template.json).

For a missing or invalid template, download to a temporary file and validate JSON before replacing `/etc/filebeat/wazuh-template.json`. Back up an existing template first. Example on srvads, with Python 3 available:

```bash
template_tmp=$(mktemp)
if curl --fail --show-error --location \
  'https://raw.githubusercontent.com/wazuh/wazuh/v4.14.8/extensions/elasticsearch/7.x/wazuh-template.json' \
  --output "$template_tmp" && python3 -m json.tool "$template_tmp" >/dev/null; then
  install -m 0644 "$template_tmp" /etc/filebeat/wazuh-template.json
fi
rm -f "$template_tmp"
```

Check that the configured template path points to this file. A successful download is not proof that the template was loaded into the Indexer. Check Filebeat startup logs and, with authorized Indexer credentials, inspect the installed template:

```bash
curl --fail --show-error --cacert /etc/wazuh-indexer/certs/root-ca.pem \
  --user admin 'https://192.168.0.49:9200/_template/wazuh?pretty'
```

## 12. Filebeat: Wazuh Module 0.5

**Status: reported resolved; exact prior error and successful installation output unavailable.**

Filebeat **7.10.2** and the Wazuh Filebeat module **0.5** are different versioned components. Installing Filebeat alone does not install the Wazuh module.

The module archive is `https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.5.tar.gz`, as documented in the [Wazuh server installation guide](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html). For a new or missing module, download to a temporary location, require a successful HTTP response, inspect the archive, then extract it under `/usr/share/filebeat/module`. Preserve an existing module before replacing it.

Check the installation and configuration:

```bash
filebeat version
test -d /usr/share/filebeat/module/wazuh && echo "Wazuh module present"
test -f /usr/share/filebeat/module/wazuh/alerts/manifest.yml
filebeat test config -e
```

The alert fileset should be enabled in `/etc/filebeat/filebeat.yml`:

```yaml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: false
```

Merge with the existing configuration; do not add duplicate top-level keys. Directory presence alone does not prove the module revision, successful ingestion, or a correct template.

## 13. Filebeat: Certificates and HTTP 401 Keystore Error

**Status: certificate and credential corrections reported resolved; current tests pending.**

### Keep TLS and authentication failures separate

For TLS errors, inspect the configured CA, certificate, and key paths. The Filebeat pair must match, use the intended identity, and be readable by the service. The trusted CA must validate the Indexer certificate, and the certificate must cover the endpoint used when hostname verification is enabled.

Illustrative output settings for this lab (merge into the existing configuration):

```yaml
output.elasticsearch:
  hosts: ["192.168.0.49:9200"]
  protocol: https
  username: ${username}
  password: ${password}
  ssl.certificate_authorities: ["/etc/filebeat/certs/root-ca.pem"]
  ssl.certificate: "/etc/filebeat/certs/filebeat.pem"
  ssl.key: "/etc/filebeat/certs/filebeat-key.pem"
```

Inspect certificate metadata and permissions without printing keys:

```bash
stat -c '%a %U:%G %n' /etc/filebeat/certs /etc/filebeat/certs/*
openssl x509 -in /etc/filebeat/certs/filebeat.pem -noout -subject -issuer -dates
openssl x509 -in /etc/wazuh-indexer/certs/indexer.pem -noout -ext subjectAltName
filebeat test output -e
```

Adapt the Indexer filename to its actual configuration. Do not treat disabling certificate validation as a permanent repair.

### Correct the credentials Filebeat actually uses

An HTTP **401 Unauthorized** response indicates that an HTTP server responded but rejected authentication. A working interactive Indexer login does not prove Filebeat's keystore contains the same current credential.

Check key names, not values:

```bash
filebeat keystore list
```

The configuration above references the exact names `username` and `password`. Use the same account and `path.data` as the Filebeat service when managing its keystore. On the packaged service, confirm the unit with `systemctl cat filebeat`; custom launch options can change which keystore is read.

If the keystore is absent, create it with `filebeat keystore create`. Do not recreate an existing keystore. To replace the two entries, enter the current Indexer credentials at the prompts:

```bash
filebeat keystore add username --force
filebeat keystore add password --force
filebeat test config -e
filebeat test output -e
# Apply a successful credential correction to the running service.
systemctl restart filebeat
systemctl status filebeat --no-pager -l
```

These commands avoid putting a literal password in shell history or documentation. See the [Filebeat 7.10 keystore reference](https://www.elastic.co/guide/en/beats/filebeat/7.10/keystore.html). A **403** response requires a permissions review, not simply another password change. If the Manager's separate Indexer connector is enabled, its own keystore must also be kept consistent; that connector is not the Filebeat alert pipeline.

## 14. Dashboard: `kibanaserver` Authentication

**Status: reported resolved; previous recap states Dashboard access succeeded after correction.**

`kibanaserver` is the Dashboard's service account for its **Indexer** connection. It is separate from the human Dashboard login and the **Manager API** account, commonly `wazuh-wui`.

Check `/etc/wazuh-dashboard/opensearch_dashboards.yml` locally for the expected `opensearch.hosts` and `opensearch.username`. Synchronize the Dashboard's stored password with the existing Indexer `kibanaserver` credential. Do not reset every Wazuh password to fix one service account.

The keystore supports interactive entry, so no literal password is needed:

```bash
/usr/share/wazuh-dashboard/bin/opensearch-dashboards-keystore --allow-root list
/usr/share/wazuh-dashboard/bin/opensearch-dashboards-keystore --allow-root add opensearch.password
systemctl restart wazuh-dashboard
systemctl status wazuh-dashboard --no-pager -l
journalctl -u wazuh-dashboard -b -n 80 --no-pager
```

Confirm replacement at the local prompt if the key exists. Use the service's configured keystore location. Consult the [Dashboard credential configuration](https://documentation.wazuh.com/current/installation-guide/wazuh-dashboard/step-by-step.html) for the distinction between Indexer credentials and the Manager API settings in `data/wazuh/config/wazuh.yml`.

The detailed original failure text and successful command output are unavailable. The November API error below must not be reclassified as another `kibanaserver` failure without evidence.

## 15. Manager Bind Mount Missing After Reboot

**Host: srvads. Status: runtime recovery confirmed by the user on 2026-11-21; persistence still to validate.**

The Manager failed with exit status **127**:

```text
ExecStart=/usr/bin/env /var/ossec/bin/wazuh-control start
/var/ossec/bin/wazuh-control: No such file or directory
```

The installation had placed Manager files on `/home`, where more space was available, and used a bind mount:

```text
/home/wazuh-manager → /var/ossec
```

After reboot, the missing mount meant systemd could not find the executable at its expected path. The service being enabled did not guarantee that the filesystem layout had been restored.

### Check before mounting

```bash
findmnt --mountpoint /var/ossec
ls -l /home/wazuh-manager/bin/wazuh-control
ls -ld /var/ossec
```

If the source binary is absent, stop and investigate the source directory/package state. If `/var/ossec` is already mounted from an unexpected source, investigate that mount. Do not mount over unexpected data, reinstall the Manager, or delete directories as a first response.

### Restore the known mount

For the diagnosed case where the source exists and the expected mount is absent:

```bash
mkdir -p /var/ossec
mountpoint -q /var/ossec || mount --bind /home/wazuh-manager /var/ossec
test -x /var/ossec/bin/wazuh-control && echo "Manager binary restored"
findmnt --mountpoint /var/ossec
```

`findmnt` may display a block device followed by a bracketed subdirectory instead of the literal bind-source path. Check that it identifies the expected storage.

### Persist the mount in `/etc/fstab`

Back up `/etc/fstab`, then edit it and add this line **once**, only if an equivalent entry is absent:

```fstab
/home/wazuh-manager /var/ossec none bind 0 0
```

Suggested validation after editing:

```bash
findmnt --fstab --target /var/ossec
findmnt --verify --verbose
systemctl daemon-reload
findmnt --mountpoint /var/ossec
systemctl reset-failed wazuh-manager
systemctl start wazuh-manager
systemctl status wazuh-manager --no-pager -l
/var/ossec/bin/agent_control -l
```

Expected service state is `active (running)`. A current `findmnt` result proves the runtime mount, not that `/etc/fstab` will restore it at boot. If `/home` is a separate filesystem, verify its availability and mount ordering too. An explicit systemd dependency such as `RequiresMountsFor=/var/ossec /home/wazuh-manager` can be considered if ordering remains a problem; no such override is confirmed installed.

The user replied that it worked after this recovery procedure. No final `/etc/fstab` contents or subsequent reboot output were supplied. During the next planned reboot, verify the mount and Manager startup **before manually remounting anything**; this is the remaining persistence test.

## 16. Dashboard `AxiosError 3000`

During the same Manager startup failure, the Dashboard displayed:

```text
AxiosError: Error getting the authorization token
3000 - Error getting the authorization token:
API host with host ID [default] could not check the ability to use the run as.
```

The observed missing Manager executable explains why its API could be unavailable even when the Dashboard web page loaded. Restoring the Manager mount was followed by the user's success confirmation. There is no separately captured token request or `run_as` check.

If the error returns, check in this order:

1. Verify `/var/ossec`, the Manager binary, and `wazuh-manager` state.
2. Check that the API is listening on TCP 55000 and inspect fresh Manager/API logs.
3. Review the `default` API host's URL, port, account, and `run_as` setting in `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml` without publishing credentials.
4. If connectivity works, test API authentication locally and review the account's permissions. Do not assume Indexer `admin` or `kibanaserver` credentials apply to this API.

```bash
systemctl status wazuh-manager --no-pager -l
ss -lntp | grep ':55000 '
journalctl -u wazuh-manager -b -n 100 --no-pager
tail -n 80 /var/ossec/logs/api.log
```

An optional authentication check below prompts for the **API** account password and suppresses the response body, which contains a token. First replace the CA path with the CA that actually signs this API's certificate and use a hostname covered by it. Do not assume it is the Indexer CA.

```bash
curl --silent --show-error --fail \
  --cacert /path/to/trusted-manager-api-ca.pem \
  --user wazuh-wui --request POST \
  --output /dev/null --write-out 'HTTP %{http_code}\n' \
  'https://srvads.luhoop.lan:55000/security/user/authenticate?raw=true'
```

Use the configured API username if different. A successful authentication response still does not independently prove the required `run_as` permission. Follow the [Dashboard troubleshooting guide](https://documentation.wazuh.com/current/user-manual/wazuh-dashboard/troubleshooting.html) and [Wazuh API authentication reference](https://documentation.wazuh.com/current/user-manual/api/getting-started.html). Error 3000 alone is not proof of a bad password or a specific root cause.

## 17. Verification and Evidence

Use the [routine checks in README.md](README.md#access-and-routine-verification), then verify each layer independently:

| Layer | Check | What it establishes |
|---|---|---|
| Packages | `dpkg-query -W` on the appropriate host | Installed version; not running state |
| Storage | Runtime mount, fstab entry, available space | Filesystem configuration; reboot test still required |
| Manager / API | Service, listener, authenticated API request | Service/API availability; permission checks are separate |
| Agent | Local service and Manager-side agent list | Endpoint connection; not necessarily alert generation |
| Filebeat | `test config`, `test output`, fresh logs | Configuration/output connectivity; not end-to-end delivery |
| Indexer | Authenticated health response and alert index inspection | Cluster availability and stored data |
| Dashboard | Login, time range, agent filter, matching alert | Searchable investigation evidence |

Additional checks on srvads:

```bash
journalctl -u filebeat -b -n 80 --no-pager
tail -n 5 /var/ossec/logs/alerts/alerts.json
curl --fail --show-error --cacert /etc/wazuh-indexer/certs/root-ca.pem \
  --user admin 'https://192.168.0.49:9200/_cat/indices/wazuh-alerts-*?v'
```

Logs and alerts can contain sensitive event data; inspect locally and sanitize evidence before publishing. For the first end-to-end test, generate a small, controlled SSH authentication failure on the Debian test endpoint, confirm the source is collected and a rule matches, then correlate the timestamp, agent name, and rule ID across Manager alerts and Dashboard search. Do not claim a completed detection from service status alone.

- [ ] Capture current package versions and central service states.
- [ ] Confirm the intended Indexer data path, certificate names, and authenticated health.
- [ ] Confirm Filebeat configuration, module, template, and authenticated output.
- [ ] Read back the bind-mount fstab entry and pass a planned reboot test.
- [ ] Confirm Debian Manager-side connection and a searchable test alert.
- [ ] Recheck Pi-hole startup policy, DNS ownership, and AD A records.
- [ ] Finish Tiny Windows installation, then enroll its agent and install/configure Sysmon.
- [ ] Record Windows telemetry and a matching alert before marking that phase complete.

## 18. Lessons from the Wazuh Installation

- File paths, mount persistence, and service startup order are part of application availability.
- TLS trust, Indexer authentication, Manager API authentication, and authorization are separate checks.
- Filebeat forwards alerts; a healthy Manager alone does not prove that the Dashboard receives new data.
- Versioned templates, modules, and packages must be documented separately.
- Preserve the original failure, the corrective action, and the evidence of recovery without publishing secrets.
