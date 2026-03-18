# TAK Server 5.6 + WinTAK on Windows with Docker Desktop — Issues & Fixes

---

## License & Disclaimer

Maintained by **SPUD OPS**

Public domain. No restrictions on use, modification, or redistribution. These solutions were discovered through hands-on debugging — none of this is covered in the TAK Server Configuration Guide, WinTAK Quick Start Guide, myTeckNet, CivTAK, or any community resource as of March 2026.

**Tested environment:** TAK Server 5.6-RELEASE-22, WinTAK 5.6.0.140, Docker Desktop on Windows 11, March 2026. Results may differ with other versions.

This guide assumes you're using the **TAK Server Configurator** package from tak.gov with Docker Desktop on Windows 11.

**Test in a lab before deploying to operations.**

---

## Network Architecture Overview

The setup described in this guide runs everything on a single Windows 11 machine:

```
                        INTERNET
                           |
                      [Router/NAT]
                           |
                    LAN (e.g. 192.168.x.0/24)
                           |
              [Windows 11 Laptop — Single Host]
                    |              |
            Docker Network     Native Windows
            (172.16.16.0/24)   Processes
                    |              |
           +--------+--------+    +------------------+
           |                 |    |                  |
     [takserver]      [tak-database]  [WinTAK]   [Python scripts]
     container         container      (binds      (mesh radio CoT
     :8089 CoT         :5432 DB        :8443!)     fixer, etc.)
     :8443 API
     :8446 Admin
     :8444 Fed
```

**The core conflict:** WinTAK and TAK Server both want port 8443. WinTAK binds it on the host for its own internal HTTPS (Hypertext Transfer Protocol Secure — encrypted web traffic) listener. Docker tries to map it for TAK Server's Marti API (Application Programming Interface — the REST interface TAK Server exposes for file transfers, data packages, and administration). They can't both have it.

**External clients** (ATAK on phones/tablets over cellular) connect through the router via port forwarding to the remapped Docker port.

**Internal clients** (WinTAK on the same machine) get silently redirected via Windows `netsh portproxy` from the port they expect (8443) to the port Docker is actually listening on (8553).

---

## Table of Contents

1. [WinTAK Steals Port 8443 and Breaks File Transfers](#1-wintak-steals-port-8443-and-breaks-file-transfers)
2. [TAK Server 5.6 Forces TLS on All Connectors](#2-tak-server-56-forces-tls-on-all-connectors)
3. [TAK Server Configurator Generates Broken Default CoreConfig.xml](#3-tak-server-configurator-generates-broken-default-coreconfigxml)
4. [docker compose down Destroys All Certificates](#4-docker-compose-down-destroys-all-certificates)
5. [Certificate Rebuild Gotchas](#5-certificate-rebuild-gotchas)
6. [WinTAK Certificate Caching](#6-wintak-certificate-caching)
7. [External Data Feeds — General Pattern](#7-external-data-feeds--general-pattern)
8. [AIS Feed (aiscot) Setup with PyTAK](#8-ais-feed-aiscot-setup-with-pytak)
9. [ADS-B Aircraft Feed (adsbcot) Setup with PyTAK](#9-ads-b-aircraft-feed-adsbcot-setup-with-pytak)
10. [Mesh Radio CoT Integration](#10-mesh-radio-cot-integration)
11. [Networking Gotchas](#11-networking-gotchas)
12. [Working Configuration Files](#12-working-configuration-files)
13. [Quick Diagnostic Commands](#13-quick-diagnostic-commands)
14. [WAN IP Changes — What Breaks and What to Update](#14-wan-ip-changes--what-breaks-and-what-to-update)
15. [Windows Firewall Rules for TAK](#15-windows-firewall-rules-for-tak)
16. [Additional Lessons Learned](#16-additional-lessons-learned)
17. [Resource Links](#17-resource-links)

---

## 1. WinTAK Steals Port 8443 and Breaks File Transfers

### Context

When TAK Server runs inside Docker on the same Windows machine as WinTAK, both applications try to use TCP (Transmission Control Protocol) port 8443. This creates a silent conflict that specifically breaks file transfers (data package uploads and downloads), even though CoT (Cursor on Target — the XML-based data format TAK uses for real-time position and messaging data) streaming on port 8089 works fine.

### The Problem

**What happens:** WinTAK 5.6 silently binds port 8443 on startup for its own internal HTTPS listener. This listener serves a self-signed certificate with `CN=unknown`. It happens every time WinTAK starts, regardless of Docker or TAK Server state.

**How it affects TAK:** When WinTAK attempts a file transfer (data package upload/download), it hardcodes port 8443 for the Marti API request — it ignores `webBaseUrl` in CoreConfig.xml. Because WinTAK owns 8443 on the host, it connects to its own listener instead of TAK Server and fails with:

```
Failed to upload Data Package
Unable to query TAK server to see if package exists
```

Or in Wireshark/logs: `Unknown CA` TLS (Transport Layer Security — the encryption protocol used for secure connections) fatal alert (WinTAK is seeing its own `CN=unknown` cert instead of the TAK Server cert).

You can verify this with:

```powershell
netstat -ano | findstr ":8443.*LISTENING"
# Returns WinTAK's PID, not Docker's
Get-Process -Id <PID>
# Shows "WinTAK"
```

### The Fix

**Three components work together:**

**Step 1 — Remap Docker's port mapping in docker-compose.yaml:**

Change `8443:8443` to `8553:8443`. This puts Docker on a port WinTAK doesn't own:

```yaml
ports:
  - 8553:8443   # NOT 8443:8443 — WinTAK steals that port
  - 8446:8446
  - 8089:8089
  - 8444:8444
```

**Step 2 — Update CoreConfig.xml:**

Set `webBaseUrl` and `urladd` to use port 8553 so WAN (Wide Area Network — your internet-facing network) clients (ATAK on cellular) can reach the API:

```xml
<federation-server webBaseUrl="https://<YOUR_WAN_IP>:8553/Marti">
```

```xml
<urladd host="https://<YOUR_WAN_IP>:8553"/>
```

**Step 3 — Add Windows netsh portproxy rules:**

WinTAK hardcodes 8443 and can't be configured to use a different port. The portproxy silently redirects WinTAK's 8443 traffic to Docker's 8553:

```powershell
# Redirect WAN IP:8443 -> LAN IP:8553
netsh interface portproxy add v4tov4 listenport=8443 listenaddress=<YOUR_WAN_IP> connectport=8553 connectaddress=<YOUR_LAN_IP>

# Redirect LAN IP:8443 -> LAN IP:8553
netsh interface portproxy add v4tov4 listenport=8443 listenaddress=<YOUR_LAN_IP> connectport=8553 connectaddress=<YOUR_LAN_IP>
```

These rules are persistent across reboots. Verify with:

```powershell
netsh interface portproxy show all
```

**Step 4 — Add router port forward:**

Forward `<WAN_IP>:8553` to `<LAN_IP>:8553` so external ATAK clients can reach the Marti API.

### How It Works

```
WinTAK file transfer -> hardcoded :8443
    |
netsh portproxy intercepts -> redirects to :8553
    |
Docker on :8553 -> container :8443 -> TAK Server Tomcat
    |
File transfer succeeds

ATAK on cellular -> WAN_IP:8553 (via router port forward)
    |
Docker on :8553 -> container :8443 -> TAK Server Tomcat
    |
File transfer succeeds
```

### Why Other Approaches Don't Work

| Approach | Why It Fails |
|----------|-------------|
| Change `webBaseUrl` to port 8553 | WinTAK ignores `webBaseUrl` for file transfer port — always uses 8443 |
| Use port 8080 HTTP for file transfers | TAK Server 5.6 forces TLS on ALL connectors (see Section 2) |
| Add `tls="false"` to 8080 connector | Crashes the API process with infinite "Waiting for API process" loop |
| Clear WinTAK SSL cert cache | Doesn't change port behavior — WinTAK still goes to 8443 |
| Stop WinTAK's 8443 listener | Cannot be disabled — it's built into WinTAK with no configuration option |
| Windows hosts file redirect | Hosts file only works for hostnames, not raw IP addresses |

---

## 2. TAK Server 5.6 Forces TLS on All Connectors

### Context

TAK Server has historically supported both plain HTTP and HTTPS connectors. In earlier versions, you could serve the Marti API over HTTP on port 8080 for simple setups or debugging. TAK Server 5.6 changed this behavior.

### The Problem

**What happens:** TAK Server 5.6 applies the global `<security><tls>` configuration from CoreConfig.xml to every connector, including port 8080 which is traditionally the HTTP connector.

**How it affects TAK:** There is no way to serve plain HTTP. Attempting to reach port 8080 via plain HTTP returns:

```
Bad Request
This combination of host and port requires TLS.
```

Attempting to force HTTP by adding `tls="false"`:

```xml
<connector port="8080" tls="false" _name="http_plaintext"/>
```

**Crashes the API process entirely.** TAK Server enters an infinite "Waiting for API process" loop and never starts. The messaging service starts (CoT flows on 8089), but the Marti API (file transfers, admin UI, REST endpoints) never comes up.

### The Fix

Accept that TAK Server 5.6 cannot serve plain HTTP. All file transfers must go through HTTPS. Use the port remap solution from Section 1 instead of trying to use HTTP on port 8080.

Keep the 8080 connector in CoreConfig.xml for potential future use, but don't rely on it for file transfers:

```xml
<connector port="8080" _name="http"/>
```

---

## 3. TAK Server Configurator Generates Broken Default CoreConfig.xml

### Context

The TAK Server Configurator is the official installation package from tak.gov that automates the setup of TAK Server in Docker. It generates a `docker-compose.yaml` and `CoreConfig.xml` as part of the setup. On a standard deployment where TAK Server runs on its own dedicated machine, these defaults may work. On a co-hosted setup (TAK Server + WinTAK on the same Windows machine with Docker), the defaults break several things.

### The Problem

**What happens:** A fresh install from the TAK Server Configurator generates a CoreConfig.xml with Docker internal IPs in client-facing URLs and missing keystore attributes on HTTPS connectors.

**How it affects TAK:** The Docker internal IPs (e.g., `172.16.16.x`) are unreachable from any client outside the Docker network. The missing keystore attributes cause Tomcat (the Java web server inside TAK Server) to fall back to a self-signed `CN=unknown` certificate, which makes HTTPS connections fail with certificate errors even when your PKI (Public Key Infrastructure — the system of certificates and certificate authorities) is correctly configured.

| Default Value | Problem |
|--------------|---------|
| `webBaseUrl="https://172.16.16.x:8443/Marti"` | Uses Docker internal IP — unreachable from any client |
| `urladd host="http://172.16.16.x:8080"` | Same Docker internal IP problem |
| `<connector port="8443" _name="https"/>` | Missing `keystoreFile`, `keystorePass`, and `keyAlias` attributes |
| `<connector port="8446" ... />` | Same missing keystore attributes |
| No 8080 connector | Not present by default |

### The Fix

Always add explicit keystore attributes to 8443 and 8446 connectors:

```xml
<connector port="8443" clientAuth="NONE" _name="https"
           keystore="JKS" keystoreFile="certs/files/takserver.jks"
           keystorePass="<YOUR_CERT_PASSWORD>" keyAlias="takserver"/>

<connector port="8446" clientAuth="false" _name="Cert_Enrollment"
           keystore="JKS" keystoreFile="certs/files/takserver.jks"
           keystorePass="<YOUR_CERT_PASSWORD>" keyAlias="takserver"/>
```

Update `webBaseUrl` and `urladd` to use your actual WAN IP and the remapped port:

```xml
<federation-server webBaseUrl="https://<YOUR_WAN_IP>:8553/Marti">
<urladd host="https://<YOUR_WAN_IP>:8553"/>
```

**Keep a backup of your working CoreConfig.xml and docker-compose.yaml.** A fresh Configurator install overwrites both files with broken defaults.

---

## 4. docker compose down Destroys All Certificates

### Context

TAK Server's PKI — the Root CA (Certificate Authority — the trusted entity that signs certificates), Intermediate CA, server certificate, and every client certificate — is generated and stored inside the Docker container at `/opt/tak/certs/files/`. If these files are lost, every client (WinTAK, ATAK, feed containers) must be re-issued new certificates and reconfigured.

### The Problem

**What happens:** The TAK Server Configurator's `docker-compose.yaml` does not bind-mount the certificate directory (`/opt/tak/certs/files`). Certificates exist only in the Docker container's writable layer — a temporary storage layer that is deleted when the container is removed.

**How it affects TAK:** Running `docker compose down` removes the container and **all certificates are permanently destroyed** — Root CA, Intermediate CA, server cert, every client cert, everything. The default `docker-compose.yaml` only mounts:

```yaml
volumes:
  - ./shared:/opt/tak/certs/shared
  - ./plugins:/opt/tak/webcontent/webtak-plugins/plugins/
```

Note that `certs/files` is NOT in this list. Also be aware that `docker compose down -v` destroys named volumes as well — never use the `-v` flag unless you intend a full rebuild.

### The Fix

**Always back up certificates before any compose operation:**

```powershell
mkdir "C:\Users\<USER>\Desktop\tak-cert-backup" -ErrorAction SilentlyContinue
docker cp takserver:/opt/tak/certs/files "C:\Users\<USER>\Desktop\tak-cert-backup"
```

**Restore after `docker compose up -d`:**

```powershell
docker cp "C:\Users\<USER>\Desktop\tak-cert-backup\files" takserver:/opt/tak/certs/files
docker cp "C:\Users\<USER>\Desktop\CoreConfig.xml" takserver:/opt/tak/CoreConfig.xml
docker restart tak-database; Start-Sleep 15; docker restart takserver
```

### Optional: Add a Volume Mount

You could add a certs volume mount to `docker-compose.yaml` to prevent this entirely:

```yaml
volumes:
  - ./tak-certs:/opt/tak/certs/files
  - ./shared:/opt/tak/certs/shared
  - ./plugins:/opt/tak/webcontent/webtak-plugins/plugins/
```

This has not been extensively tested with the Configurator workflow but would prevent cert loss on compose operations.

---

## 5. Certificate Rebuild Gotchas

### Context

TAK Server 5.6 uses a two-tier PKI: a Root CA signs an Intermediate CA, which in turn signs all server and client certificates. The `makeCert.sh` script (included in the TAK Server container) handles certificate generation, but the output requires post-processing to work with WinTAK 5.6 on modern Windows. There are roughly a dozen silent failure modes — missing files cause infinite startup hangs, wrong keystore aliases break authentication, legacy encryption formats are rejected by modern Windows, and incomplete truststores cause TLS handshake failures.

### fed-truststore.jks Must Exist

**What happens:** The federation truststore file (`fed-truststore.jks`) is referenced in CoreConfig.xml under the `<federation>` section. If the file is missing from `/opt/tak/certs/files/`, TAK Server hangs at startup.

**How it affects TAK:** TAK Server logs repeat the following line indefinitely and never fully starts:

```
Waiting for the Retention Query process...
```

**The fix:** Create it after every PKI rebuild:

```bash
keytool -import -trustcacerts -alias root-ca \
  -file /opt/tak/certs/files/root-ca.pem \
  -keystore /opt/tak/certs/files/fed-truststore.jks \
  -storepass <CERT_PASSWORD> -noprompt
```

### Server Cert Must Have IP SANs

**What happens:** SAN (Subject Alternative Name) is a field inside an SSL/TLS certificate that lists which hostnames and/or IP addresses the certificate is valid for. An IP SAN specifically means an IP address entry in that field. The default server cert generated by `makeCert.sh server takserver` only includes `DNSName: takserver` — no IP SANs.

**How it affects TAK:** When WinTAK connects to TAK Server using an IP address (e.g., `192.168.1.50:8089`), it checks the server certificate's SANs for that IP. Without the correct IP SANs, WinTAK fires a TLS `Unknown CA` fatal alert even with correct truststores. This applies to every IP a client might use to connect — your WAN IP, your LAN IP, and Docker's internal subnet IP.

**The fix:** After generating the server cert, add your IPs to the SAN configuration file and re-sign:

```bash
echo 'IP.1 = <YOUR_WAN_IP>
IP.2 = <YOUR_LAN_IP>
IP.3 = <YOUR_DOCKER_SUBNET_IP>' >> /opt/tak/certs/files/config-takserver.cfg
```

Then re-sign the server cert with the updated config. The full certificate chain (server cert + intermediate CA cert + root CA cert) must be included in the JKS (Java KeyStore — the keystore format used by TAK Server's Tomcat web server).

### Server Cert Keystore Alias Must Be "takserver"

**What happens:** TAK Server's JWT (JSON Web Token — used for API authentication) engine looks for the exact alias `takserver` in the JKS keystore.

**How it affects TAK:** If the alias is anything else (e.g., `takserver-core`, `1`, or the default import name), API authentication fails silently. The admin UI and file transfers stop working with no clear error message.

**The fix:** When importing with keytool, always specify:

```bash
keytool -importkeystore ... -alias takserver -noprompt
```

### p12 Files Need Re-encryption for WinTAK 5.6

**What happens:** TAK's `makeCert.sh` generates p12 (PKCS#12 — a binary certificate bundle format) files using legacy RC2-40-CBC encryption.

**How it affects TAK:** WinTAK 5.6 on modern Windows (which uses updated OpenSSL/SChannel libraries) cannot import these legacy-encrypted p12 files. The import either fails silently or produces certificate errors.

**The fix:**

```bash
# Extract everything from the legacy p12
openssl pkcs12 -in cert.p12 -passin pass:<PASSWORD> -passout pass:<PASSWORD> -legacy -out temp.pem

# Repackage with modern encryption
openssl pkcs12 -export -in temp.pem -passin pass:<PASSWORD> -passout pass:<PASSWORD> -out cert-new.p12

# Clean up
rm temp.pem
```

### Truststore Must Contain Both CAs

**What happens:** `truststore-intermediate.p12` is used as the trust store imported into WinTAK and ATAK. TAK's default `makeCert.sh` may only include one of the two CAs.

**How it affects TAK:** If only the intermediate CA is present (without the root CA), or vice versa, WinTAK fails to validate the server certificate chain and returns `peer not verified` errors. Using `truststore-root.p12` (root CA only) also causes this error.

**The fix:**

```bash
# Add intermediate CA
keytool -import -trustcacerts -alias intermediate \
  -file intermediate.pem \
  -keystore truststore-intermediate.p12 \
  -storepass <PASSWORD> -storetype PKCS12 -noprompt

# Add root CA
keytool -import -trustcacerts -alias root-ca \
  -file root-ca.pem \
  -keystore truststore-intermediate.p12 \
  -storepass <PASSWORD> -storetype PKCS12 -noprompt
```

### UserManager.jar Requires a Fully Started Server

**What happens:** `UserManager.jar` is the command-line tool for managing certificate authorizations in TAK Server. It needs the TAK Server's internal services to be fully initialized before it can run.

**How it affects TAK:** Running `UserManager.jar certmod -A` (the `-A` flag grants admin privileges — without it, clients can authenticate but receive no CoT data) before TAK Server has fully started produces:

```
Failed to find deployed service: distributed-user-file-manager
```

**The fix:** Wait until `docker logs takserver` shows `Retention Application started` before running UserManager commands.

### makeCert.sh Permission Errors After Cert Restore

**What happens:** After restoring certs from backup via `docker cp`, the `/opt/tak/certs/files/` directory may be owned by root instead of the `tak` user (the non-root user the TAK Server process runs as inside the container).

**How it affects TAK:** Subsequent `makeCert.sh` calls fail with `Permission denied`.

**The fix:** Run makeCert with root:

```powershell
docker exec -u root takserver bash -c "cd /opt/tak/certs && ./makeCert.sh client <CERTNAME>"
```

### OpenSSL 3.x Requires -legacy Flag

**What happens:** The TAK Server container ships with OpenSSL 3.x, which changed how it handles PKCS#12 files.

**How it affects TAK:** All `openssl pkcs12` commands inside the TAK Server container fail to read TAK-generated p12 files without the `-legacy` flag. You get cryptic errors about unsupported algorithms.

**The fix:** Add `-legacy` to every `openssl pkcs12` command:

```bash
openssl pkcs12 -in cert.p12 -passin pass:<PASSWORD> -legacy ...
```

### auth Section Must Not Be Empty

**What happens:** CoreConfig.xml has an `<auth>` section that defines how users are authenticated.

**How it affects TAK:** An empty `<auth></auth>` section causes client certificate validation failures even when certs are correct. Clients connect on 8089 but get authentication errors.

**The fix:** Always include:

```xml
<auth>
    <File location="UserAuthenticationFile.xml"/>
</auth>
```

### CA Private Key Is Encrypted

**What happens:** The intermediate CA's private key (`ca-do-not-share.key`) is encrypted with the certificate password.

**How it affects TAK:** Any OpenSSL command that uses this key (e.g., re-signing the server cert) will fail without providing the password via `-passin pass:<PASSWORD>`.

**The fix:** Always include `-passin pass:<YOUR_CERT_PASSWORD>` when referencing the CA key in OpenSSL commands.

---

## 6. WinTAK Certificate Caching

### Context

WinTAK stores its TLS certificates and trust decisions in a local folder on the Windows machine. When you replace certificates (e.g., after a PKI rebuild), WinTAK does not automatically pick up the new files. This is a common source of confusion after cert rebuilds — everything looks correct, but WinTAK keeps failing with the old certificate errors.

### The Problem

**What happens:** WinTAK creates `.dat` sidecar files next to each `.p12` certificate file in:

```
C:\Users\<USER>\AppData\Roaming\WinTAK\SslCerts\
```

These `.dat` files cache TLS trust decisions (WinTAK's internal SQLite databases do not store cert data — it's all in these `.dat` files).

**How it affects TAK:** Replacing a `.p12` file without deleting the corresponding `.dat` file has no effect. WinTAK continues using the cached trust decision from the old certificate.

### The Fix

Full wipe and reimport:

```powershell
Stop-Process -Name WinTAK -Force
Remove-Item "C:\Users\<USER>\AppData\Roaming\WinTAK\SslCerts\*" -Force
# Reopen WinTAK and reimport certs through the UI
```

### Note: httpscert.p12

WinTAK always has an `httpscert.p12` in its SslCerts folder. **This is NOT a TAK Server cert** — it's WinTAK's own internal certificate for its built-in HTTPS listener (the one that binds port 8443). `peer not verified` errors referencing this cert are normal background noise and can be ignored.

---

<!-- NEW SECTION -->
<div style="border-left: 4px solid #2196F3; padding-left: 16px; margin-bottom: 24px;">

## 7. External Data Feeds — General Pattern

### Context

TAK's real power comes from fusing multiple data sources onto a single map — ship positions, aircraft tracks, mesh radio units, weather stations, sensor platforms. The PyTAK framework (a Python library maintained by the SNSTAC team) provides a standard way to pull data from external sources, convert it to CoT (Cursor on Target — TAK's XML data format), and push it into TAK Server over TLS (Transport Layer Security).

Several community tools are built on PyTAK, each handling a specific data type. The two most common for situational awareness are `aiscot` (ship tracking via AIS) and `adsbcot` (aircraft tracking via ADS-B). Both follow the same general pattern for connecting to TAK Server, and both hit the same undocumented issues.

### The General Pattern for Any PyTAK Feed

If you want to add a new external data feed to your TAK Server, the process is the same regardless of the data type:

**1. Generate a client certificate for the feed:**

```powershell
docker exec -u root takserver bash -c "cd /opt/tak/certs && ./makeCert.sh client <FEEDNAME>"
docker exec takserver bash -c "java -jar /opt/tak/utils/UserManager.jar certmod -A /opt/tak/certs/files/<FEEDNAME>.pem"
```

**2. Extract PEM-format certs from the p12 bundle** (PyTAK cannot use p12 files directly):

```bash
# Extract client certificate
openssl pkcs12 -in <FEEDNAME>.p12 -passin pass:<PASSWORD> -nokeys -out <FEEDNAME>.crt.pem -legacy

# Extract and decrypt private key
openssl pkcs12 -in <FEEDNAME>.p12 -passin pass:<PASSWORD> -nocerts -out <FEEDNAME>.key.pem -legacy -passout pass:<PASSWORD>
openssl rsa -in <FEEDNAME>.key.pem -out <FEEDNAME>.key.pem -passin pass:<PASSWORD>

# Build CA bundle (intermediate + root concatenated)
cat intermediate.pem root-ca.pem > <FEEDNAME>.ca.bundle.pem
```

**3. Build a custom Docker image** that includes the `cryptography` Python package (not included by default in any PyTAK-based image):

```dockerfile
FROM python:3.11-slim
RUN pip install --no-cache-dir <feedtool> pytak cryptography
CMD ["<feedtool>"]
```

**4. Run the container** on the `takserver` Docker network with certs mounted and all `PYTAK_TLS_*` environment variables set explicitly:

```powershell
docker run -d --name <FEEDNAME> --restart unless-stopped `
  --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 `
  --network takserver `
  -v "/path/to/<FEEDNAME>.crt.pem:/certs/client.crt.pem:ro" `
  -v "/path/to/<FEEDNAME>.key.pem:/certs/client.key.pem:ro" `
  -v "/path/to/<FEEDNAME>.ca.bundle.pem:/certs/ca.bundle.pem:ro" `
  -e "COT_URL=tls://takserver:8089" `
  -e "PYTAK_TLS_CLIENT_CERT=/certs/client.crt.pem" `
  -e "PYTAK_TLS_CLIENT_KEY=/certs/client.key.pem" `
  -e "PYTAK_TLS_CA_CERTS=/certs/ca.bundle.pem" `
  -e PYTAK_TLS_DONT_CHECK_HOSTNAME=1 `
  -e PYTAK_TLS_DONT_VERIFY=1 `
  -e MAX_IN_QUEUE=50000 `
  -e MAX_OUT_QUEUE=50000 `
  <your-image>
```

### Common Gotchas (Apply to All PyTAK Feeds)

These issues apply to every PyTAK-based feed — `aiscot`, `adsbcot`, `adsbxcot`, `stratuxcot`, or any custom feed:

| Issue | Symptom | Fix |
|-------|---------|-----|
| Missing `cryptography` package | `NameError: name 'rsa' is not defined` | Add `cryptography` to Dockerfile |
| Encrypted private key | TLS handshake failure, no clear error | Decrypt key with `openssl rsa` |
| Missing `PYTAK_TLS_CLIENT_CERT` | `SyntaxError: Missing value: PYTAK_TLS_CLIENT_CERT` | Set all three `PYTAK_TLS_*` env vars even with `DONT_VERIFY=1` |
| Default queue size (10) | `QueueFull` crash when data source returns many items | Set `MAX_IN_QUEUE=50000` and `MAX_OUT_QUEUE=50000` |
| Wrong CA bundle | `peer not verified` or `certificate verify failed` | Concatenate intermediate.pem + root-ca.pem (both required) |
| Container not on `takserver` network | Connection refused to `takserver:8089` | Add `--network takserver` to `docker run` |
| No log rotation | Log files grow to gigabytes over days | Add `--log-driver json-file --log-opt max-size=10m --log-opt max-file=3` |

The sections below cover feed-specific setup for AIS (Section 8) and ADS-B (Section 9). Our operational setup uses both — AIS for maritime vessel tracking and ADS-B for aircraft tracking.

</div>

---

## 8. AIS Feed (aiscot) Setup with PyTAK

### Context

AIS (Automatic Identification System) is a tracking system used by ships and vessel traffic services. Vessels broadcast their position, speed, course, and identification over radio frequencies, and this data is also aggregated by online services like SeaVision (operated by the U.S. DOT Volpe Center).

`aiscot` is a Python tool built on the PyTAK framework that pulls AIS vessel data from online sources and converts it into CoT (Cursor on Target) messages — the standard data format used by TAK. This lets you see real-time ship positions on the TAK map.

Running `aiscot` as a Docker container alongside your TAK Server seems straightforward, but getting it to authenticate over TLS (Transport Layer Security — encrypted connections) to TAK Server involves multiple undocumented requirements. The default `aiscot` Docker image is missing a required Python package, the certificate format requirements are specific and poorly documented, and several environment variables must be set even when you'd expect them to be optional.

### The Problem

**What happens:** The `aiscot` container needs to connect to TAK Server via mutual TLS (mTLS — where both the client and server present certificates to each other) on port 8089. This requires PEM-format certs (Privacy Enhanced Mail — a text-based certificate format) with a decrypted private key, explicit environment variables for cert paths (even when verification is disabled), a large queue size to avoid crashes, and the `cryptography` Python package which isn't installed by default.

**How it affects TAK:** Without all of these pieces in place, the aiscot container fails to start or fails to connect to TAK Server, and no AIS vessel tracks appear on the map.

### The Fix

Build a custom Docker image with `cryptography` added, extract and decrypt PEM certs from the p12 (PKCS#12 — a binary certificate bundle format that TAK Server generates by default), mount them into the container, and set all `PYTAK_TLS_*` environment variables explicitly.

### Details

**TLS Client Cert Is Required Even with DONT_VERIFY:**

Even with `PYTAK_TLS_DONT_VERIFY=1`, PyTAK still requires the `PYTAK_TLS_CLIENT_CERT` environment variable to be set for mutual TLS on port 8089. Without it:

```
SyntaxError: Missing value: PYTAK_TLS_CLIENT_CERT
```

**aiscot Uses Environment Variables Only — No Config File:**

Unlike some PyTAK-based tools, `aiscot` does not read from a configuration file. All settings are passed via `-e` environment variables in the `docker run` command.

**Cert Files Must Be PEM Format with Decrypted Key:**

PyTAK cannot use encrypted private keys or p12 files directly. TAK Server generates p12 bundles by default, so you need to extract and convert them. After generating a client cert in TAK Server:

```bash
# Extract the client certificate from the p12 bundle
openssl pkcs12 -in client.p12 -passin pass:<PASSWORD> -nokeys -out client.crt.pem -legacy

# Extract the private key (still encrypted at this point)
openssl pkcs12 -in client.p12 -passin pass:<PASSWORD> -nocerts -out client.key.pem -legacy -passout pass:<PASSWORD>

# Decrypt the private key (PyTAK cannot read encrypted keys)
openssl rsa -in client.key.pem -out client.key.pem -passin pass:<PASSWORD>

# Build CA bundle by concatenating intermediate and root certs
cat intermediate.pem root-ca.pem > ca.bundle.pem
```

**Mount Certs into Container:**

```powershell
docker run -d --name aiscot --restart unless-stopped `
  --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 `
  --network takserver `
  -v "/path/to/client.crt.pem:/certs/client.crt.pem:ro" `
  -v "/path/to/client.key.pem:/certs/client.key.pem:ro" `
  -v "/path/to/ca.bundle.pem:/certs/ca.bundle.pem:ro" `
  -e "COT_URL=tls://takserver:8089" `
  -e "PYTAK_TLS_CLIENT_CERT=/certs/client.crt.pem" `
  -e "PYTAK_TLS_CLIENT_KEY=/certs/client.key.pem" `
  -e "PYTAK_TLS_CA_CERTS=/certs/ca.bundle.pem" `
  -e PYTAK_TLS_DONT_CHECK_HOSTNAME=1 `
  -e PYTAK_TLS_DONT_VERIFY=1 `
  -e MAX_IN_QUEUE=50000 `
  -e MAX_OUT_QUEUE=50000 `
  <your-aiscot-image>
```

Note the `--log-driver` flags — these prevent the container's logs from growing unbounded.

**MAX_IN_QUEUE and MAX_OUT_QUEUE Must Be Large:**

The default PyTAK queue size (10) causes `QueueFull` crashes when the AIS data source returns dozens of vessels per poll (SeaVision can return 40-80 vessels in a single response). Set both to 50000.

**SeaVision Poll Interval Must Be At Least 65 Seconds:**

The SeaVision API has a rate limiter that enforces roughly a 60-second minimum between requests. Set `POLL_INTERVAL=65` to stay safely above this limit.

**Dockerfile Needs cryptography Package:**

The default `aiscot` image does not include the `cryptography` Python package, which PyTAK needs for TLS connections. You need to build a custom image:

```dockerfile
FROM python:3.11-slim
RUN pip install --no-cache-dir aiscot pytak cryptography
CMD ["aiscot"]
```

Without `cryptography`, you get:

```
NameError: name 'rsa' is not defined
```

---

<!-- NEW SECTION -->
<div style="border-left: 4px solid #2196F3; padding-left: 16px; margin-bottom: 24px;">

## 9. ADS-B Aircraft Feed (adsbcot) Setup with PyTAK

### Context

ADS-B (Automatic Dependent Surveillance-Broadcast) is a surveillance technology where aircraft determine their position via GPS (Global Positioning System) and periodically broadcast it. This data includes the aircraft's callsign, altitude, speed, heading, and position. Anyone with an SDR (Software Defined Radio — a USB radio receiver that can tune to various frequencies) and the right software can receive these broadcasts and decode them.

`adsbcot` is a Python tool built on the PyTAK framework that takes decoded ADS-B aircraft data and converts it into CoT messages for TAK. This lets you see real-time aircraft positions on the TAK map. It connects to a local ADS-B decoder (typically `dump1090` or `readsb`) that does the actual radio reception and decoding.

Setting this up involves two layers: getting the SDR hardware and decoder working, then getting `adsbcot` to authenticate to TAK Server over TLS. The SDR/decoder layer has its own set of issues, and the PyTAK/TLS layer has all the same undocumented problems as `aiscot` (see Section 7 for the general pattern).

### The Architecture

```
[SDR USB Dongle] → [dump1090 / readsb decoder] → [adsbcot] → [TAK Server :8089]
     Radio              Decodes ADS-B              Converts        Displays
     hardware           to JSON/SBS               to CoT           on map
```

`dump1090` or `readsb` receives raw 1090 MHz radio signals from the SDR dongle, decodes them into structured aircraft data, and serves it over a local network port (typically port 30003 for SBS format or port 30005 for Beast binary format). `adsbcot` connects to that local port, converts the aircraft data to CoT XML, and sends it to TAK Server via TLS.

### SDR and Decoder Setup (dump1090)

Before `adsbcot` can do anything, you need a working ADS-B decoder.

**Hardware required:**
- An RTL-SDR USB dongle (RTL2832U-based, ~$25-35) — this is the radio receiver
- An ADS-B antenna (1090 MHz tuned) — comes with most RTL-SDR kits

**Software required:**
- **dump1090** — the most common open-source ADS-B decoder. On Windows, `dump1090` runs as a native executable
- **RTL-SDR drivers** — the USB drivers for the SDR dongle (Zadig is commonly used on Windows to install the WinUSB driver)

**dump1090 Critical Configuration:**

```
dump1090 --net --interactive --log-file NUL
```

| Flag | Purpose |
|------|---------|
| `--net` | Enables network output (ports 30001-30005) so `adsbcot` can connect |
| `--interactive` | Shows decoded aircraft in the console (useful for verifying the SDR is working) |
| `--log-file NUL` | **CRITICAL on Windows.** Without this, dump1090 logs every received message to a file. On a busy frequency, this log file can grow to 37+ GB in days, filling the disk completely. `NUL` is the Windows null device (equivalent to `/dev/null` on Linux) |

**Verify dump1090 is receiving aircraft:**

Once dump1090 is running with `--net`, you can verify it's decoding aircraft by opening a browser to `http://localhost:8080` (dump1090's built-in web UI) or by checking port 30003:

```powershell
# Check if dump1090 is serving data on SBS port
Test-NetConnection -ComputerName localhost -Port 30003
```

### adsbcot Setup

Once dump1090 is running and receiving aircraft, set up `adsbcot` to forward the data to TAK Server.

**Dockerfile:**

```dockerfile
FROM python:3.11-slim
RUN pip install --no-cache-dir adsbcot pytak cryptography
CMD ["adsbcot"]
```

**Build:**

```powershell
cd "C:\path\to\adsbcot"
docker build -t adsbcot-local:1 .
```

**Run command:**

```powershell
docker run -d --name adsbcot --restart unless-stopped `
  --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 `
  --network takserver `
  -v "/path/to/client.crt.pem:/certs/client.crt.pem:ro" `
  -v "/path/to/client.key.pem:/certs/client.key.pem:ro" `
  -v "/path/to/ca.bundle.pem:/certs/ca.bundle.pem:ro" `
  -e "FEED_URL=tcp+beast://host.docker.internal:30005" `
  -e "COT_URL=tls://takserver:8089" `
  -e "PYTAK_TLS_CLIENT_CERT=/certs/client.crt.pem" `
  -e "PYTAK_TLS_CLIENT_KEY=/certs/client.key.pem" `
  -e "PYTAK_TLS_CA_CERTS=/certs/ca.bundle.pem" `
  -e PYTAK_TLS_DONT_CHECK_HOSTNAME=1 `
  -e PYTAK_TLS_DONT_VERIFY=1 `
  -e MAX_IN_QUEUE=50000 `
  -e MAX_OUT_QUEUE=50000 `
  -e COT_STALE=120 `
  adsbcot-local:1
```

### ADS-B-Specific Gotchas

These are in addition to the general PyTAK issues covered in Section 7.

**FEED_URL Must Use host.docker.internal:**

`dump1090` runs natively on Windows (outside Docker), but `adsbcot` runs inside a Docker container. The container can't reach `localhost` on the host machine. Use `host.docker.internal` — a special Docker Desktop DNS name that resolves to the host machine's IP from inside a container:

```
FEED_URL=tcp+beast://host.docker.internal:30005
```

Do NOT use `localhost`, `127.0.0.1`, or the Docker network gateway IP. `host.docker.internal` is the only reliable way to reach host services from a Docker container on Docker Desktop for Windows.

**Beast Format (Port 30005) vs SBS Format (Port 30003):**

`adsbcot` supports multiple input formats. Beast binary format on port 30005 is more efficient and carries more data than SBS text format on port 30003. Use Beast format unless you have a specific reason not to:

| Format | FEED_URL | Port | Notes |
|--------|----------|------|-------|
| Beast binary | `tcp+beast://host.docker.internal:30005` | 30005 | Preferred — more data, lower overhead |
| SBS/BaseStation | `tcp://host.docker.internal:30003` | 30003 | Text-based, simpler but less data |

**dump1090 Must Be Running Before adsbcot Starts:**

If `dump1090` isn't running when `adsbcot` starts, the container will crash-loop trying to connect to the feed URL. The `--restart unless-stopped` flag handles this by restarting the container until `dump1090` comes up, but expect error logs during that period.

**RTL-SDR Driver Conflicts on Windows:**

Windows may install its own generic drivers for the RTL-SDR USB dongle. These drivers don't work with dump1090. You need to use Zadig (a USB driver installer) to replace the default driver with WinUSB. This is a one-time setup per USB dongle, but if Windows Update reinstalls the generic driver, dump1090 will stop receiving data with no clear error — it just shows zero aircraft.

**COT_STALE for Aircraft Should Be Short:**

Aircraft move fast. A stale time of 120 seconds (2 minutes) is reasonable. If an aircraft stops broadcasting (landed, went out of range, or transponder off), its track disappears from the map after 2 minutes. For AIS vessels (which move slowly), the stale time can be much longer (700 seconds in our setup).

**Busy Airspace = High Queue Throughput:**

In busy airspace (near airports), dump1090 can decode hundreds of aircraft messages per second. The default PyTAK queue size of 10 will cause immediate `QueueFull` crashes. Set `MAX_IN_QUEUE=50000` and `MAX_OUT_QUEUE=50000` as with all PyTAK feeds.

**UAT 978 MHz (U.S. Only):**

In the United States, general aviation aircraft below 18,000 feet can use UAT (Universal Access Transceiver) on 978 MHz instead of standard 1090 MHz ADS-B. To receive UAT traffic, you need a second SDR dongle tuned to 978 MHz and a separate decoder (`dump978`). The `uat978cot` PyTAK tool handles UAT-to-CoT conversion using the same general pattern described in Section 7.

</div>

---

## 10. Mesh Radio CoT Integration

### Context

Tactical mesh radios (such as Persistent Systems MPU5 or Silvus radios) form self-healing wireless networks used in military and maritime operations. These radios can transmit CoT (Cursor on Target) data via UDP (User Datagram Protocol) multicast on the physical LAN (Local Area Network), allowing radio-equipped vehicles or personnel to share their positions.

The challenge is getting this multicast CoT data into TAK Server when TAK Server runs inside a Docker container. Docker's networking model creates an isolated virtual network that doesn't receive multicast traffic from the host's physical network interfaces.

### The Problem

**What happens:** Docker's bridge network (e.g., 172.16.16.0/24) is completely isolated from the host machine's physical network interfaces. UDP multicast packets arriving on the host's Ethernet NIC (Network Interface Card) never cross the bridge into the container. On top of that, the CoT XML from mesh radios is often malformed in ways that TAK Server cannot parse.

**How it affects TAK:** You might think adding a multicast `<input>` to CoreConfig.xml would work:

```xml
<!-- DO NOT DO THIS -->
<input _name="mesh" protocol="udp" group="239.x.x.x" port="xxxxx"/>
```

**Don't.** Even if the multicast traffic somehow reached the container, TAK Server would double-encode HTML entities in the CoT XML, corrupting all tracks. This is a known issue with UDP multicast inputs and certain types of mesh radio CoT.

### The Fix: Native Script with Scapy

Run a Python script natively on Windows (not in Docker) that:

1. Listens for UDP multicast on the physical NIC using scapy's packet capture
2. Parses the raw CoT XML from the multicast packets
3. Fixes common malformations (see below)
4. Forwards the cleaned CoT to TAK Server via TLS on port 8089

The script connects to TAK Server using the **host's LAN IP** (e.g., `192.168.x.x`), not Docker internal IPs (172.16.16.x) or localhost (127.0.0.1). Both of those are unreachable from native Windows processes trying to reach a containerized service.

### Common Mesh Radio CoT Malformations

Depending on your mesh radio system, the CoT XML may have some or all of these issues:

| Malformation | Description |
|-------------|-------------|
| HTML entity encoding | XML contains `&lt;` `&gt;` `&amp;` instead of `<` `>` `&` |
| Epoch timestamps | `time`, `start`, `stale` fields use Unix epoch (seconds since 1970) instead of ISO 8601 format (`YYYY-MM-DDTHH:MM:SSZ`) |
| Zero stale window | `stale` equals `time`, making tracks immediately expire and disappear from the map |
| Null Island positions | Lat/lon of `0,0` (a point in the Gulf of Guinea) when the radio has no GPS fix |
| Missing `<track>` element | Speed/course element absent, causing some TAK clients to ignore the track entirely |

A fixer script should detect and correct each of these before forwarding to TAK Server:

- Decode HTML entities back to proper XML characters
- Convert epoch timestamps to `YYYY-MM-DDTHH:MM:SSZ` format
- Add a reasonable stale window (e.g., 30 seconds after `time`)
- Filter or flag Null Island positions so they don't appear at 0,0 on the map
- Inject a default `<track speed="0" course="0"/>` if missing

### Script Requirements

- **Python 3.x** with **scapy** (`pip install scapy`) — a packet manipulation library that can capture raw network traffic
- **Npcap** — required by scapy on Windows for packet capture (WinPcap is deprecated; Npcap is the modern replacement)
- TLS client certificate (PEM format, decrypted key) for mutual TLS to TAK Server on 8089
- The script's `TAK_HOST` must be set to the host machine's **static LAN IP** — never Docker internal IPs or localhost
- The script's sniff interface must be set to the correct physical NIC name (check with `ipconfig`)

**Important:** Use the full path to the real Python executable (find it with `where.exe python`), not the Windows Store alias (`WindowsApps\python.exe`), which may not work from a scheduled task or elevated context.

### Auto-Start with Task Scheduler

Set up a Windows Scheduled Task to run the fixer script at logon:

```powershell
$action = New-ScheduledTaskAction `
  -Execute "C:\path\to\python.exe" `
  -Argument '"C:\path\to\your_cot_fixer.py"' `
  -WorkingDirectory "C:\path\to\script\directory"

$trigger = New-ScheduledTaskTrigger -AtLogon

$settings = New-ScheduledTaskSettingsSet `
  -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
  -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1) `
  -ExecutionTimeLimit (New-TimeSpan -Days 365)

$principal = New-ScheduledTaskPrincipal -UserId "<USER>" -RunLevel Highest -LogonType Interactive

Register-ScheduledTask -TaskName "MeshRadioCoTFixer" `
  -Action $action -Trigger $trigger -Settings $settings -Principal $principal -Force
```

---

## 11. Networking Gotchas

### Context

Running TAK Server in Docker on a Windows machine introduces several non-obvious networking behaviors. These are easy to miss because they don't produce clear error messages — they manifest as silent failures, infinite loops, or intermittent connectivity issues.

### Restart Order: Database Always Before TAK Server

**What happens:** TAK Server depends on a PostgreSQL database running in a separate container (`tak-database`). On startup, TAK Server immediately tries to connect to the database.

**How it affects TAK:** Restarting TAK Server before the database is ready causes HikariPool (the Java database connection pool library) connection failures and a "Waiting for API process" infinite loop. The messaging process may start, but the API never comes up.

**The fix:**

```powershell
docker restart tak-database; Start-Sleep 15; docker restart takserver
```

Always restart the database first and wait at least 15 seconds before restarting TAK Server.

### Windows hosts File Cannot Override Raw IP Addresses

**What happens:** The Windows hosts file (`C:\Windows\System32\drivers\etc\hosts`) maps hostnames to IP addresses.

**How it affects TAK:** You cannot use the hosts file to redirect one IP address to another. An entry like:

```
192.168.50.100    74.x.x.x    # THIS DOES NOT WORK
```

is silently ignored. The hosts file only works for hostname-to-IP mappings (e.g., `takserver    192.168.50.100`). If you're using raw IP addresses everywhere (as this guide recommends), the hosts file cannot help with redirection. Use `netsh portproxy` instead.

### VPN Software Hijacks TAK WAN Connections

**What happens:** Overlay network software like Tailscale, ZeroTier, or similar VPN tools modify the Windows routing table and may intercept traffic destined for WAN IP addresses.

**How it affects TAK:** WinTAK's 8443 connections (and potentially 8089 connections) may be routed through the VPN tunnel instead of to the local Docker container. Connections fail intermittently or produce unexpected certificate errors.

**The fix:** Disable any VPN/overlay services you're not actively using:

```powershell
Stop-Service Tailscale -Force
Set-Service Tailscale -StartupType Disabled
```

### Docker Compose Restart Does Not Reload Port Mappings

**What happens:** Port mappings in `docker-compose.yaml` are only read when the container is created (during `docker compose up`).

**How it affects TAK:** `docker restart takserver` does NOT re-read `docker-compose.yaml`. If you change port mappings (e.g., `8443:8443` to `8553:8443`), the change is ignored until you destroy and recreate the container.

**The fix:** To apply port mapping changes, you must:

```powershell
# WARNING: destroys certs — back up first!
docker compose down
docker compose up -d
# Then restore certs and CoreConfig from backup
```

### Hairpin NAT (Most Consumer Routers Don't Support It)

**What happens:** When a device on your LAN (Local Area Network — your home/office network) tries to reach your own public WAN (Wide Area Network — your internet-facing) IP address, the traffic goes out to your router. On most consumer and prosumer routers, the router doesn't know to send that traffic back to a device on the same local network — it just drops it. This is called a **hairpin NAT (Network Address Translation)** failure.

**How it affects TAK:** If TAK Server is running on your machine and you configure WinTAK to connect via your public IP (e.g., `WAN_IP:8443`), the connection fails. The packets leave your machine, hit the router, and go nowhere. You'll see connection timeouts in WinTAK with no useful error message.

**The fix:** Using `netsh portproxy` creates a local forwarding rule on the Windows machine itself. Traffic destined for the public IP gets intercepted and redirected *before it ever leaves the machine*, bypassing the router entirely. This means WinTAK can connect to your public IP and have it silently forwarded to TAK Server's Docker container — no router cooperation needed. Setup steps are in Section 1.

### WSL Ghost Directory Bug

**What happens:** WSL2 (Windows Subsystem for Linux — the backend Docker Desktop uses) has a known issue where bind-mounted files that are deleted and recreated on the Windows side may appear as empty directories inside the container.

**How it affects TAK:** If you delete and recreate files (like CoreConfig.xml) that are bind-mounted into the container, the container may see a directory instead of a file, causing startup failures.

**The fix:** Use `docker cp` to copy files into the container instead of relying on bind mounts for frequently-edited files.

---

## 12. Working Configuration Files

### docker-compose.yaml

```yaml
version: '3.7'
services:
  takserver-db:
    build:
      dockerfile: Dockerfile-takserver-db
    container_name: tak-database
    hostname: tak-database
    init: true
    networks:
      - net
    ports:
      - 5432:5432
    restart: unless-stopped
    tty: true

  takserver-core:
    build:
      dockerfile: Dockerfile-takserver
    container_name: takserver
    hostname: takserver
    init: true
    networks:
      - net
    ports:
      - 8080:8080
      - 8553:8443    # CRITICAL: Not 8443:8443 — WinTAK owns that port
      - 8446:8446
      - 8089:8089
      - 8444:8444
    restart: unless-stopped
    tty: true
    volumes:
      - ./shared:/opt/tak/certs/shared
      - ./plugins:/opt/tak/webcontent/webtak-plugins/plugins/

networks:
  net:
    name: 'takserver'
    ipam:
      driver: default
      config:
        - subnet: 172.16.16.0/24
```

### CoreConfig.xml (key sections — sanitized)

```xml
<network multicastTTL="5" serverId="<UUID>" version="5.6-RELEASE-22-HEAD">
    <input auth="x509" _name="stdssl" protocol="tls" port="8089"/>
    <connector port="8080" _name="http"/>
    <connector port="8443" clientAuth="NONE" _name="https"
               keystore="JKS" keystoreFile="certs/files/takserver.jks"
               keystorePass="<PASSWORD>" keyAlias="takserver"/>
    <connector port="8444" useFederationTruststore="true" _name="fed_https"/>
    <connector port="8446" clientAuth="false" _name="Cert_Enrollment"
               keystore="JKS" keystoreFile="certs/files/takserver.jks"
               keystorePass="<PASSWORD>" keyAlias="takserver"/>
    <announce/>
</network>

<auth>
    <File location="UserAuthenticationFile.xml"/>
</auth>

<submission ignoreStaleMessages="false" validateXml="false"/>

<repository enable="true" numDbConnections="16"
            primaryKeyBatchSize="500" insertionBatchSize="500">
    <connection url="jdbc:postgresql://tak-database:5432/cot"
               username="martiuser" password="<DB_PASSWORD>"/>
</repository>

<filter>
    <urladd host="https://<YOUR_WAN_IP>:8553"/>
</filter>

<security>
    <tls keystore="JKS" keystoreFile="certs/files/takserver.jks"
         keystorePass="<PASSWORD>" truststore="JKS"
         truststoreFile="certs/files/truststore-intermediate.jks"
         truststorePass="<PASSWORD>" context="TLSv1.2" keymanager="SunX509">
        <crl _name="TAKServer CA" crlFile="certs/files/intermediate.crl"/>
    </tls>
</security>

<federation missionFederationDisruptionToleranceRecencySeconds="43200">
    <federation-server webBaseUrl="https://<YOUR_WAN_IP>:8553/Marti">
        <tls keystore="JKS" keystoreFile="certs/files/takserver.jks"
             keystorePass="<PASSWORD>" truststore="JKS"
             truststoreFile="certs/files/fed-truststore.jks"
             truststorePass="<PASSWORD>" keymanager="SunX509"/>
    </federation-server>
</federation>
```

### netsh portproxy Rules

```powershell
# Redirect WinTAK's hardcoded 8443 to Docker's 8553
netsh interface portproxy add v4tov4 listenport=8443 listenaddress=<YOUR_WAN_IP> connectport=8553 connectaddress=<YOUR_LAN_IP>
netsh interface portproxy add v4tov4 listenport=8443 listenaddress=<YOUR_LAN_IP> connectport=8553 connectaddress=<YOUR_LAN_IP>

# Verify
netsh interface portproxy show all
```

---

<!-- NEW SECTION - Added from operational experience -->
<div style="border-left: 4px solid #2196F3; padding-left: 16px; margin-bottom: 24px;">

## 13. Quick Diagnostic Commands

### Context

When something breaks, these commands help you quickly identify the state of each component without guessing.

### Check If TAK Server Is Fully Started

```powershell
docker logs takserver --tail 20
# Look for: "Retention Application started" = fully booted
# If you see "Waiting for API process" looping = API is broken (check CoreConfig)
# If you see "Waiting for the Retention Query process" = fed-truststore.jks is missing
```

### Check Who Owns Port 8443

```powershell
netstat -ano | findstr ":8443.*LISTENING"
# Get the PID from the last column, then:
Get-Process -Id <PID>
# If it says "WinTAK" — that's the expected co-hosted behavior
# If it says something else — investigate
```

### Check Docker Port Mappings Are Active

```powershell
docker ps --format "table {{.Names}}\t{{.Ports}}"
# Verify you see 0.0.0.0:8553->8443/tcp for takserver
```

### Check netsh Portproxy Rules Are In Place

```powershell
netsh interface portproxy show all
# Should show your WAN and LAN IP redirecting 8443 -> 8553
```

### Test Marti API From the Command Line

```powershell
# From the host machine (bypasses WinTAK's port conflict via portproxy):
curl -k https://<YOUR_LAN_IP>:8553/Marti/api/version

# From inside the container:
docker exec takserver curl -k https://localhost:8443/Marti/api/version
```

### Check Container Networking

```powershell
# Verify aiscot can resolve takserver hostname:
docker exec aiscot ping -c 1 takserver

# Check which network containers are on:
docker network inspect takserver
```

### View Recent Certificate Errors

```powershell
docker logs takserver 2>&1 | findstr /i "cert\|ssl\|tls\|unknown\|peer\|handshake" | Select-Object -Last 20
```

</div>

---

<!-- NEW SECTION - Extracted from context lessons L38 -->
<div style="border-left: 4px solid #2196F3; padding-left: 16px; margin-bottom: 24px;">

## 14. WAN IP Changes — What Breaks and What to Update

### Context

If your public WAN IP changes (e.g., Starlink lease changes, ISP reset, switching to a different internet connection), multiple components reference this IP and must be updated in sync. Missing any one of them causes partial failures that are difficult to diagnose because CoT streaming may still work while file transfers or cert validation break.

### What References the WAN IP

| Component | Where | What to Update |
|-----------|-------|---------------|
| CoreConfig.xml | `webBaseUrl` in `<federation-server>` | Change to new WAN IP |
| CoreConfig.xml | `<urladd host="...">` in `<filter>` | Change to new WAN IP |
| Server certificate | IP SAN field | Must regenerate server cert with new IP |
| netsh portproxy | `listenaddress` on the WAN rule | Delete old rule, add new one |
| Router port forwards | Inbound rules | Update if router uses WAN IP in rules |
| WinTAK | Server connection profile | Update server address if using WAN IP |
| ATAK | Server connection profile | Update server address |

### Update Procedure

```powershell
# 1. Update CoreConfig.xml with new WAN IP (webBaseUrl and urladd)
# 2. Regenerate server cert with new IP SANs (see Section 5)
# 3. Update netsh portproxy:
netsh interface portproxy delete v4tov4 listenport=8443 listenaddress=<OLD_WAN_IP>
netsh interface portproxy add v4tov4 listenport=8443 listenaddress=<NEW_WAN_IP> connectport=8553 connectaddress=<YOUR_LAN_IP>
# 4. Restart TAK Server
docker restart tak-database; Start-Sleep 15; docker restart takserver
# 5. Update client profiles in WinTAK/ATAK
```

### Tip: Stable WAN IP

If using Starlink, bypass mode provides a more stable public IP than the default CGNAT (Carrier-Grade NAT — where your ISP shares one public IP among many customers) configuration. Check your Starlink settings if your WAN IP changes frequently.

</div>

---

<!-- NEW SECTION - Extracted from context Windows system state -->
<div style="border-left: 4px solid #2196F3; padding-left: 16px; margin-bottom: 24px;">

## 15. Windows Firewall Rules for TAK

### Context

Windows Defender Firewall blocks inbound connections by default. TAK Server needs several ports open for clients to connect. If you've set up everything correctly but external clients can't reach your server, missing firewall rules are a common cause.

### Required Inbound Rules

| Port | Protocol | Purpose |
|------|----------|---------|
| 8089 | TCP | CoT streaming (mutual TLS — primary client connection) |
| 8553 | TCP | Marti API / file transfers (remapped from 8443) |
| 8446 | TCP | Cert enrollment + Admin UI |
| 8444 | TCP | Federation (only if federating with other TAK servers) |
| 8080 | TCP | HTTP connector (optional — not functional for plain HTTP in 5.6) |

### Create Firewall Rules

```powershell
New-NetFirewallRule -DisplayName "TAK CoT 8089" -Direction Inbound -Protocol TCP -LocalPort 8089 -Action Allow
New-NetFirewallRule -DisplayName "TAK API 8553" -Direction Inbound -Protocol TCP -LocalPort 8553 -Action Allow
New-NetFirewallRule -DisplayName "TAK Admin 8446" -Direction Inbound -Protocol TCP -LocalPort 8446 -Action Allow
New-NetFirewallRule -DisplayName "TAK Federation 8444" -Direction Inbound -Protocol TCP -LocalPort 8444 -Action Allow
```

**Note:** Port 8553 is easy to forget because it's not a standard TAK port — it's your custom remapped port. If external ATAK clients can connect on 8089 (CoT works) but file transfers fail, check that 8553 is open in the firewall.

</div>

---

<!-- NEW SECTION - Consolidated from context lessons not covered in main sections -->
<div style="border-left: 4px solid #2196F3; padding-left: 16px; margin-bottom: 24px;">

## 16. Additional Lessons Learned

These are smaller issues discovered during operational debugging that don't warrant their own full section but are valuable to know.

### PowerShell Scripts Must Be Pure ASCII

PowerShell management scripts (e.g., `tak-manager.ps1`) must be saved as pure ASCII or UTF-8 without BOM. Unicode characters (smart quotes, em dashes, non-ASCII symbols) cause PowerShell parsing errors. If you copy-paste from web pages or AI output, check for hidden Unicode characters.

### docker compose down -v Destroys Named Volumes

`docker compose down` removes containers. `docker compose down -v` removes containers AND named volumes. Never use the `-v` flag unless you intend a full rebuild from scratch. This is separate from the cert destruction issue in Section 4 — `-v` additionally destroys the database volume, meaning you lose all stored CoT history, missions, and data packages.

### docker cp Fails If Target Path Is Bind-Mounted

If you bind-mount CoreConfig.xml directly (e.g., `./CoreConfig.xml:/opt/tak/CoreConfig.xml`), you cannot use `docker cp` to update it. The `docker cp` command fails silently or produces errors. Instead, edit the file directly on the host side since the bind mount makes it the same file.

### validateXml Should Be False for Mesh Radio CoT

In CoreConfig.xml, the `<submission>` section has a `validateXml` attribute:

```xml
<submission ignoreStaleMessages="false" validateXml="false"/>
```

If set to `true`, TAK Server validates incoming CoT XML against the schema and rejects malformed messages. Mesh radios (MPU5, Silvus) often produce non-standard CoT that fails schema validation. Keep `validateXml="false"` if you're ingesting mesh radio data.

### WinTAK TCP Timeout Should Be Increased

WinTAK's default TCP connection timeout is short. In environments with higher latency (satellite internet, VPN, cellular), connections may time out before completing. Increase the TCP timeout to 60 seconds in WinTAK's Network Preferences.

### Container Log Size Management

Docker containers log to JSON files that grow without bound by default. For long-running containers like `aiscot`, add log rotation to the `docker run` command:

```
--log-driver json-file --log-opt max-size=10m --log-opt max-file=3
```

This caps each log file at 10 MB and keeps a maximum of 3 rotated files. Without this, log files on long-running systems can consume gigabytes of disk space.

### dump1090 Log File Must Be Null

If running dump1090 (an ADS-B decoder for SDR-based aircraft tracking), the `log-file` option must be set to `NUL` (Windows null device). Without it, dump1090 logs every received message and the log file can grow to 37+ GB, filling the disk.

### urladd and webBaseUrl Must Match

Both `<urladd host="...">` in the `<filter>` section and `webBaseUrl` in the `<federation-server>` section of CoreConfig.xml must use the same IP address and port. If they differ, file transfers break because TAK Server tells clients to download files from one URL but the API is actually serving from another.

### CRL File Must Exist

CoreConfig.xml references a CRL (Certificate Revocation List) file:

```xml
<crl _name="TAKServer CA" crlFile="certs/files/intermediate.crl"/>
```

If this file is missing, TAK Server logs warnings on every client connection. The `makeCert.sh` script generates this file during CA creation. After a cert restore from backup, verify it's present.

</div>

---

## Environment Tested

- Windows 11 Pro
- Docker Desktop with WSL2 (Windows Subsystem for Linux 2) backend
- TAK Server 5.6-RELEASE-22 (Docker, from tak.gov Configurator)
- WinTAK 5.6.0.140
- Starlink internet with bypass mode for stable public IP
- Peplink MAX BR1 router (no hairpin NAT support)

---

<!-- NEW SECTION -->
<div style="border-left: 4px solid #2196F3; padding-left: 16px; margin-bottom: 24px;">

## 17. Resource Links

### TAK Software

- **TAK Server & WinTAK downloads:** [https://tak.gov](https://tak.gov) (requires .mil or approved .gov account)
- **CivTAK (community TAK resources):** [https://www.civtak.org](https://www.civtak.org)

### Tools Referenced in This Guide

- **Docker Desktop for Windows:** [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)
- **Npcap (required for scapy on Windows):** [https://npcap.com/](https://npcap.com/)
- **Scapy (Python packet manipulation library):** [https://scapy.net/](https://scapy.net/)
- **OpenSSL (certificate management):** [https://www.openssl.org/](https://www.openssl.org/)

### PyTAK Ecosystem

- **PyTAK (Python TAK framework):** [https://github.com/snstac/pytak](https://github.com/snstac/pytak)
- **aiscot (AIS to CoT converter):** [https://github.com/snstac/aiscot](https://github.com/snstac/aiscot)
- **adsbcot (ADS-B to CoT converter):** [https://github.com/snstac/adsbcot](https://github.com/snstac/adsbcot)
- **uat978cot (UAT 978 MHz to CoT converter):** [https://github.com/snstac/uat978cot](https://github.com/snstac/uat978cot)
- **stratuxcot (Stratux ADS-B receiver to CoT):** [https://github.com/snstac/stratuxcot](https://github.com/snstac/stratuxcot)

### ADS-B Decoder Software

- **dump1090 (FlightAware fork):** [https://github.com/flightaware/dump1090](https://github.com/flightaware/dump1090)
- **readsb (alternative ADS-B decoder):** [https://github.com/wiedehopf/readsb](https://github.com/wiedehopf/readsb)
- **Zadig (USB driver installer for RTL-SDR on Windows):** [https://zadig.akeo.ie/](https://zadig.akeo.ie/)
- **RTL-SDR Blog (hardware + drivers):** [https://www.rtl-sdr.com/](https://www.rtl-sdr.com/)

### AIS Data Sources

- **SeaVision (U.S. DOT Volpe Center):** [https://seavision.volpe.dot.gov/](https://seavision.volpe.dot.gov/)

### Mesh Radio Documentation

- **Persistent Systems (MPU5):** [https://www.intrepidflyer.com/](https://www.intrepidflyer.com/)
- **Silvus Technologies:** [https://silvustechnologies.com/](https://silvustechnologies.com/)

</div>

---

## Contributing

**SPUD OPS**

If you've found additional issues running TAK Server + WinTAK co-hosted on Windows, please share — working through these issues was time intensive.

---

*SPUD OPS | March 2026*
