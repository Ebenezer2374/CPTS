# The Concept of Attacks: Study Notes

## Core idea

A **concept** is an outlined plan you reuse across projects (like a house: floor, walls, roof; the materials and design vary). For attacks, we want one general pattern that fits **any service** (SSH, FTP, SMB, HTTP), so we can spot attack points without memorizing each service separately.

The pattern is a **process that grows over time**, not a finished product. You keep refining it as you see more cases.

## The four categories

Every vulnerability involves these four things, in a deliberately **linear** flow:

**Source → Process → Privileges → Destination**

The flow is linear because a Destination isn't always a Source for a new task. For any task to exist, you need information (Source), a planned process (Process), a goal (Destination), and Privileges to control how the processing is allowed to happen.

---

### 1. Source: where the information comes from

The input handed to a process. **The protocol doesn't matter**: HTTP header injection and buffer overflows are both just manipulated sources.

| Source | Meaning |
| --- | --- |
| Code | Results of already-executed program code / functions |
| Libraries | Collections of reusable resources (prebuilt code, subroutines, classes, config, docs) |
| Config | Static or prescribed values that dictate how a process handles info |
| APIs | Interfaces for retrieving or providing information |
| User Input | Values manually entered by a person |

### 2. Process: how the information is handled

The developer decides how input is processed (classes, functions, calculations, loops). **Most vulnerabilities live in the code the process executes.**

| Component | Meaning |
| --- | --- |
| PID | Process ID; running processes already have privileges assigned |
| Input | Information supplied by a user or by a programmed function |
| Data processing | Hard-coded functions that dictate how input is handled |
| Variables | Placeholders for info passed between functions |
| Logging | Events recorded to a file/register; info stays on the system |

### 3. Privileges: what the process is allowed to do

Permissions that decide which actions a process can perform (think of a bus ticket valid only for certain routes). The system checks these when a process needs to do its job.

| Privilege | Meaning |
| --- | --- |
| System | Highest level: **SYSTEM** on Windows, **root** on Linux |
| User | Permissions assigned to a specific user (services often get their own users) |
| Groups | Users grouped together with shared permissions |
| Policies | Govern execution of application-specific commands, for users or groups |
| Rules | Permissions handled inside the applications themselves |

### 4. Destination: where the result goes

Every task has a goal: data is stored or forwarded somewhere. Completing that closes the cycle.

| Destination | Meaning |
| --- | --- |
| Local | The same system: files or records modified, or data passed to other local services |
| Network | Results forwarded to a remote interface: an IP/service, or whole networks (can even influence routing) |

---

## Worked example: Log4j (CVE-2021-44228)

Log4j is a Java logging **library**, widely embedded in open-source and commercial software. Logs are sensitive, so apps using it often ran with **administrator privileges**.

**The bug:** the attacker sets the HTTP **User-Agent** header to a **JNDI lookup** instead of something like `Mozilla/5.0`. Log4j misinterprets the string and *executes a request* instead of just logging it.

### Stage 1: Initiation

| # | What happens | Category |
| --- | --- | --- |
| 1 | Attacker manipulates the User-Agent with a JNDI lookup | **Source** |
| 2 | Process misinterprets the string and runs the command | **Process** |
| 3 | Lookup executes with admin privileges (due to logging permissions) | **Privileges** |
| 4 | Lookup points to the attacker's server hosting a malicious Java class | **Destination** |

### Stage 2: Trigger RCE (the cycle restarts)

| # | What happens | Category |
| --- | --- | --- |
| 5 | Malicious Java class fetched from attacker's server becomes the new input | **Source** |
| 6 | The class's malicious code is read in | **Process** |
| 7 | Code runs with admin privileges | **Privileges** |
| 8 | Code connects back to the attacker, giving remote control | **Destination** |

**Takeaway:** one attack can be *two loops* of the same pattern. The Destination of loop 1 (attacker's server) becomes the Source of loop 2.

Reference diagram: GovCERT.ch Log4j write-up ([https://www.govcert.ch/blog/zero-day-exploit-targeting-popular-java-library-log4j/](https://www.govcert.ch/blog/zero-day-exploit-targeting-popular-java-library-log4j/))

---

## Why this matters (practical uses)

- **Analyze exploits** by breaking them into the four categories
- **Debug your own exploits** during development and testing
- **Source code review**: walk through functionality step by step
- **Threat-think per task**: ask what the danger is at each of the four stages

## Quick-recall questions

When looking at any service or vuln, ask:

1. **Source:** what input can I control, and where does it come from?
2. **Process:** how does the code handle that input? Can it be misinterpreted?
3. **Privileges:** what rights does this process run with?
4. **Destination:** where does the output go (local files, remote host)? Can it feed a new Source?

## One-line summary

> Every attack = controlled **Source** → mishandled by a **Process** → running with certain **Privileges** → sending results to a **Destination**, which may start the cycle again.