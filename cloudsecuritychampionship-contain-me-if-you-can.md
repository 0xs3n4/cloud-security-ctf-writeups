# Wiz Cloud Security Championship — Challenge 2 (Container Escape)  
## CTF Vlog Write-up (No Flag / No Credentials)

> **Spoiler policy:** This write-up intentionally **does not** include the flag value or any real credentials.  
> **Goal:** document my learning journey, mistakes, and the investigative path I followed.

---

## 1) Starting Point: “I Don’t Know Containers (Yet)”

I started Challenge 2 feeling genuinely lost. The prompt clearly pointed toward **privilege escalation in a container**, but I didn’t have solid fundamentals in container security or common escape patterns.

My first instinct was to search for a ready-made walkthrough (HackTricks-style). I quickly realized that if I asked for help the wrong way, I would basically get a “full solution” handed to me—so I stopped and reset.

### What I changed
Instead of asking for a full write-up, I focused on:
- learning **container security basics**
- building a repeatable **enumeration mindset**
- collecting **evidence first**, not assumptions

---

## 2) Study Phase: Building a “Container Security Mental Model”

Before going deeper, I spent time reviewing container security concepts and hardening best practices:
- namespaces (PID, mount, network)
- Linux capabilities
- seccomp/AppArmor
- mounts / volumes / bind mounts
- common “escape by misconfiguration” cases

This changed how I approached the challenge:
- not every escape is a kernel exploit
- many escapes are **misconfigurations** (mounts/sockets/devices)
- sometimes the “escape” comes from **lateral movement + secrets + a service pivot**

---

## 3) Enumeration: Evidence First

I began with basic host/container reconnaissance:
- identity and privilege context (`id`)
- kernel and OS hints (`uname -a`, `/etc/os-release`)
- containerization signals (`/proc/1/cgroup`, overlay mounts)
- mounts and storage (`/proc/mounts`, `/proc/self/mountinfo`)
- network context (bridge IPs, gateway, internal DNS)

At first, I was looking for classic “easy wins”:
- a runtime socket (`/var/run/docker.sock`, containerd sockets)
- privileged container flags
- obvious host filesystem mounts (like `/host`)

None of the “obvious” shortcuts appeared.

---

## 4) Breakthrough: Lateral Movement via Internal PostgreSQL

During enumeration, I discovered an active TCP connection to an internal **PostgreSQL service** (`:5432`) on the Docker bridge network.

This was the first big pivot:
- instead of forcing a direct “container escape”
- I followed the evidence: **a service relationship existed**
- my container was already talking to Postgres

I tried a straightforward `psql` connection and quickly hit authentication failures. That meant:
- the DB wasn’t “trust auth”
- I needed credentials, not brute force

---

## 5) The Rabbit Hole: Inode → PID → `/proc/<pid>/environ`

My next plan was the classic “steal secrets from the process that owns the connection”:
1. find the TCP connection inode in `/proc/net/tcp`
2. map `socket:[inode]` to a PID via `/proc/<pid>/fd/`
3. read `/proc/<pid>/environ` for `DATABASE_URL`, `PGPASSWORD`, etc.

This took **a lot** of time, and I learned an important lesson:
- the inode kept changing (timing/race issues)
- I couldn’t consistently map it to a visible process
- the process model and namespaces didn’t behave the way I expected

In hindsight, I spent too long trying to “force” this method to work.

---

## 6) The Hints Changed Everything: “Plain-text Connection”

After struggling, I used **two official hints**.

The second hint was the turning point:

> **“This network connection is plain-text. Can you think of a way to take advantage of it?”**

That instantly reframed the strategy:
- if the connection is truly plain-text
- then the correct move is **sniffing the network**
- and capturing authentication material (ideally at login time)

This was the moment where the challenge stopped being “guessy” and became evidence-driven again.

---

## 7) Sniffing Reality: Capturing the Right Moment (Handshake)

I captured traffic and quickly confirmed:
- I could see SQL queries like periodic health checks (`SELECT now();`)
- but **credentials won’t show up** in the middle of an already-established session

So the real problem became:
- **how do I capture the authentication phase** (startup + password exchange)?

I needed a way to observe a reconnect / login event.

A helpful suggestion came from a friend in the **OwlSec Discord** community: using a technique to **reset the TCP session** and force a reconnect so the handshake could be captured.

Once I captured the right moment, I was able to extract:
- the DB user
- the database name
- the password (appearing in cleartext in this challenge’s setup)

> Again: I’m not including real credentials here.

---

## 8) Database Access: PostgreSQL as a Pivot

With working DB access, the next question was:
- how do I turn database access into a path to the host (where `/flag` exists)?

At this point, I re-read the Wiz research post about isolation issues in managed PostgreSQL environments and tried to apply the *mindset*:
- isolation problems often come from **roles, permissions, extensions, or service architecture**
- the DB service can become a stepping stone into the surrounding environment

I created an `exploit.sql` and ran it using `psql -f`.

---

## 9) The “Annoying but Real” Part: Quoting and Execution Friction

Once I started executing system-level commands through SQL, I hit a different class of issues:
- commands failing because they weren’t executed via a shell
- missing binaries and incorrect paths
- difficulty capturing stderr
- output tables holding “old” results because I forgot to clear them
- struggling to do “background” workflows with limited terminal setup

This was a very practical lesson:
- exploitation often fails because of **execution environment details**, not because the idea is wrong
- careful iteration beats rushing

I fixed this by standardizing how I executed commands and by always:
- capturing stderr (`2>&1`)
- validating small commands first (`id`, `pwd`, `ls`)
- clearing result tables before each run

---

## 10) Reverse Shell: Comfort, but Also New Problems

Eventually I got a remote shell into the PostgreSQL container environment. That made enumeration much faster, but I still had to deal with:
- limited tools
- single-terminal constraints
- privilege boundaries inside the container

This phase felt like a real “ops” exercise: execution reliability and workflow mattered as much as technical ideas.

---

## 11) The Reminder: “Always Enumerate Completely”

A key moment happened when I realized I had skipped an obvious baseline check:
- the `postgres` user’s privilege boundaries (including `sudo` rules)

After another pass of structured enumeration, I found a path to become root **inside the PostgreSQL container**.

This reinforced a core habit:
> Even when you think you’re deep into the challenge, go back and run full enumeration again.

---

## 12) The Final Step: Logical Escape via Devices/Mounts

Once I was root in the container, I focused on “host adjacency” evidence:
- mounts (`/proc/mounts`)
- volumes backed by block devices
- what appeared under `/dev` (including disk devices)
- what storage was mounted where

Eventually, this led to a critical discovery:
- a device exposed within the container mapped to host storage
- and the host filesystem contained the flag location described in the challenge

This wasn’t a flashy kernel exploit. It was a lesson in:
- **misconfiguration and isolation failure**
- using mounts/devices as evidence
- following what the environment is *actually* exposing

I located the flag file on the host as instructed by the challenge—without publishing it.

---

# Mistakes I Made (and What They Taught Me)

### 1) Asking for “too much help” too early
- I nearly got a full write-up, which would’ve killed the learning.
- I changed my prompts to focus on **reasoning and evidence**.

### 2) Over-investing in inode→PID tricks
- It’s a great technique, but it’s not universal.
- Namespace constraints can make it impossible or unreliable.

### 3) Sniffing the wrong part of the session
- I initially only saw keepalive queries.
- Credentials appear during **authentication**, so timing matters.

### 4) Losing time to quoting/ops friction
- Shell wrappers, stderr capture, and clean output workflows matter.
- “Exploit logic” is useless if your execution is brittle.

### 5) Skipping baseline checks
- `sudo -l` and privilege checks should be early and repeated.
- Enumeration is not “one and done.”

---

# What I Learned (Big Picture)

- Container escape isn’t always “kernel exploit.”  
  Often it’s **architecture + isolation gaps + exposed resources**.
- Service pivots (like PostgreSQL) are powerful when:
  - the network is observable
  - authentication is weak or leakable
  - the service has risky permissions or extensions
- Evidence-driven thinking wins:
  - mounts and devices don’t lie
  - `/proc` and network artifacts tell a story
- Tooling constraints are part of the challenge:
  - workflow and iteration matter as much as technique

---

# References / Sources

These are the resources that influenced my approach and helped me understand the underlying patterns:

- **Wiz Research (Primary)**  
  - *The cloud has an isolation problem: PostgreSQL vulnerabilities*  
    https://www.wiz.io/blog/the-cloud-has-an-isolation-problem-postgresql-vulnerabilities

- **Challenge page**  
  - Wiz Cloud Security Championship — Challenge 2  
    https://www.cloudsecuritychampionship.com/challenge/2

- **HackTricks (General container / privilege escalation reference)**  
  - HackTricks main knowledge base (used to look up concepts and patterns)  
    https://book.hacktricks.xyz/

- **Community help (Operational hint)**  
  - OwlSec Discord — suggestion that helped me think about forcing a reconnect to capture authentication traffic  
  *(No public link included; community discussion.)*

---

## Appendix: Personal Workflow Notes (Why this is a “vlog”)
This challenge felt like a real learning arc:
- confusion → study → structured enumeration  
- dead-end methods → hints → better hypotheses  
- ops friction → repeatable execution patterns  
- privilege escalation → logical host-adjacent evidence  
- and finally: successful retrieval of the flag

If you’re reading this because you’re learning too:  
don’t underestimate how much progress comes from **fixing your process**, not just learning one new trick.
