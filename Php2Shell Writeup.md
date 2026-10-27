# PHP2Shell — TryHackMe Writeup

**Lab:** https://tryhackme.com/room/cvephp2shell

This writeup follows the room task by task. Each task has a short, simple explanation of what is happening, followed by the question(s) asked in that task and *why* the answer is what it is.

---

## Task 1 — Introduction & Connecting to the Lab

**What's going on:**
On 21 October 2026, the PHP team announced that PHP itself (not any website built with it) had a serious bug. Two flaws, found together, let a stranger run commands on a PHP server just by sending normal web requests — no login, no file upload accepted by the app, no bug in the website's own code. The bug had been sitting inside PHP since 2012 (PHP 5.4.0), so basically every PHP site running an old enough version was affected.

This bug was named **PHP2Shell**. It's tracked as two CVEs:
- `CVE-2026-109618` — leaks server memory
- `CVE-2026-109619` — lets you run commands (RCE)

Both are rated 9.2 Critical. Fixed versions: PHP 8.5.12, 8.4.27, 8.3.36, 8.2.35.

In this task you just start the vulnerable machine. It runs PHP 8.4.26 behind Apache in a Docker container, listening on `127.0.0.1:8088`, and you connect to it over SSH.

---

## Task 2 — The Upload-Progress Feature (the "foothold")

**What's going on:**
Before we get to the bug, we need to understand the *feature* the bug lives inside.

When you upload a big file to a website, the browser sometimes needs to show a progress bar. PHP has a built-in way to support this: `session.upload_progress`. While PHP is still reading the uploaded file (before the website's actual code has even started running), PHP writes "how much of this upload has arrived" into the session data.

To turn this on, the attacker just needs to include one specific field in their upload request. That field's default name is what the question is asking about.

**The important detail:** this code runs *before* the page's own PHP script executes. That means it doesn't matter what the page actually does — even a page that does nothing but print "OK" is a valid target, because PHP processes the upload-progress field before it even looks at what script was requested. No login, no app logic, nothing  just a POST request is enough to reach this code. This is why it's called the "foothold": it's the door the entire attack walks through.

### Question: What is the default name of the form field that switches on PHP's upload-progress tracking?

**Why this is the answer:** PHP looks for a specific field name in the multipart form data to decide whether to start tracking upload progress. This name is controlled by the `session.upload_progress.name` setting in php.ini, and its default value is literally spelled out in the room text as the field you must include in your POST request to trigger the feature.

**Answer:** `PHP_SESSION_UPLOAD_PROGRESS`

---

## Task 3 — The Bailout Bug (where the dangling pointer comes from)

**What's going on:**
Now we get to the actual bug.

PHP has a memory limit per request (`memory_limit`). If a request tries to use more memory than that, PHP can't just continue — it has to abort immediately. The way it aborts is called a **bailout**: PHP jumps straight back to a safe recovery point using a low-level `longjmp()`, abandoning whatever it was doing mid-task. This is the same mechanism behind the classic "Allowed memory size exhausted" error you may have seen before.

The problem is that a bailout is an *emergency exit*, not a clean shutdown. Normally, when PHP frees a block of memory, it also clears the pointer (the "address label") that pointed to it, so nothing tries to use that memory again. But if the out-of-memory error hits at exactly the wrong moment — after the memory is freed, but before the pointer is cleared — the bailout skips the cleanup step. The pointer is left "dangling," still pointing at memory that's no longer valid.

This exact problem  a pointer that still points to memory after that memory has been freed/invalidated has an official weakness classification used across the security industry.

Because Apache (in the "prefork" mode this lab uses) reuses the same worker process for request after request without restarting, this dangling pointer doesn't disappear with the failed request. It sticks around and gets reused by the *next* request that happens to land on that same worker and now the attacker can influence what ends up in that leftover memory spot.

### Question 1: What one-word name does the PHP engine give to the longjmp it performs to abort a request that hits a fatal error such as running out of memory?

**Why this is the answer:** The room text explicitly names this mechanism it's the term PHP's own internals and changelog use for this kind of abrupt, emergency-exit recovery jump triggered by things like running out of memory.

**Answer:** `bailout`

### Question 2: What is the CWE identifier for the weakness class behind PHP2Shell? (format: CWE-NNN)

**Why this is the answer:** A pointer that still refers to a memory location after that memory should no longer be trusted/valid is a well-known, catalogued category of software weakness. The room explicitly classifies this exact scenario the dangling pointer left behind by the bailout under this standard CWE number.

**Answer:** `CWE-824`

---

## Task 4 — Confirming the Target

**What's going on:**
Before attacking anything, you check what you're dealing with. You use Docker to see which PHP version the container is running, and you look at the actual web page you're about to attack.

This step matters because it proves a point: the target page (`index.php`) is a single line of code that just prints a word to the screen. There's no login form, no database, no file upload form literally nothing an attacker could normally "hack" in the usual sense. If the exploit still works against this page, it proves the vulnerability lives in PHP's runtime itself, not in any mistake made by whoever wrote the website.

### Question 1: What version of PHP is the target server running?

**Why this is the answer:** Running `php -v` inside the container shows the exact version string. It's one patch release behind the fixed version (8.4.27), confirming this server is vulnerable.

**Answer:** `8.4.26`

### Question 2: What single word does the target page at /index.php print in response?

**Why this is the answer:** Looking at the actual source of `index.php`, the whole script is `<?php echo "OK\n"; ?>` so requesting the page with curl returns exactly that one word, nothing else.

**Answer:** `OK`

---

## Task 5 — Leaking Memory to Defeat ASLR

**What's going on:**
Modern systems scramble where code and data sit in memory, a protection called **ASLR** (Address Space Layout Randomisation). It's meant to stop attackers from knowing exactly where to point an exploit. To defeat it, the attacker needs to somehow find out the real memory addresses being used and this is exactly what `CVE-2026-109618` provides.

Here's how: the bailout bug from Task 3 is triggered again, but this time it leaves behind a dangling pointer to the *session ID* itself. A follow-up request reoccupies that freed memory with a string whose reported length is far bigger than it should be. When PHP sends the session cookie back to the browser, it reads out that oversized "session ID" — which now spills over into whatever memory happens to sit right after it. That spilled memory gets sent back to the attacker inside one specific response header, since that's the header PHP uses to deliver the session cookie back to the client.

By doing this twice, the attacker learns:
1. The base address of PHP's memory heap
2. The base address of the PHP library itself (libphp)

With both addresses known, ASLR no longer protects anything the attacker now knows exactly where things live in that worker's memory.

### Question: Which HTTP response header does CVE-2026-109618 abuse to return leaked process memory to the attacker?

**Why this is the answer:** Session IDs are normally sent to the browser so the browser can send them back on the next request, and PHP delivers that session ID using one specific, well-known response header built for exactly that purpose. Since the leak happens *through* the oversized session ID being returned to the client, it rides out inside that same header.

**Answer:** `Set-Cookie`

---

## Task 6 — Turning the Leak into Code Execution

**What's going on:**
Now the attacker has everything they need: they know where the heap and the PHP library sit in memory. `CVE-2026-109619` is what actually pulls the trigger.

PHP internally uses "destructors" functions that run automatically to clean up a value once it's no longer needed. The exploit plants a fake internal data structure into that same dangling-memory spot from before, but this time, instead of planting data to *read*, it plants a forged structure whose "destructor pointer" has been rewritten to point at the address of `system()` the function that runs OS commands inside the PHP library (an address the attacker now knows thanks to Task 5's leak).

When PHP later runs its normal cleanup at the end of the request, it thinks it's calling a harmless internal cleanup function. Instead, it's actually calling `system()` with a command the attacker supplied. That command runs as whatever user the web server process runs as.

Running the exploit script performs the whole chain automatically: leak the heap address, leak the library address, forge the destructor, and execute a command proven by running `id` on the target.

### Question 1: When the exploit runs, which system user do the commands execute as?

**Why this is the answer:** The command executes inside the PHP worker process, and that worker process runs under the web server's own system account not root, and not any special attacker-controlled user. The proof output from running the exploit (`id`) shows exactly which account that is.

**Answer:** `www-data`

### Question 2: What is the flag returned by the exploit?

**Why this is the answer:** Once code execution works, you can run any command you like including reading the flag file that sits on the server at a short path (`/f`). Running the exploit with `-c 'cat /f>/out/p1'` makes the forged destructor execute that exact command as `www-data`, which reads the flag file and writes it where the tool can print it back to you.


**Answer:** ~~`THM{b4ilout_l3aves_th3_h3ap_dang1ing}`~~

---

## Task 7 — Fixing It and Detecting It

**What's going on:**
Because the bug is inside PHP itself, updating your website's framework (WordPress, Laravel, Drupal, whatever) does absolutely nothing to fix it. The only real fix is patching the PHP runtime/interpreter itself and restarting every worker process so none of them keep running the old, vulnerable code in memory.

If you can't patch immediately, there are stop-gap measures:
- Turn off the upload-progress feature entirely (removes the delivery mechanism for most of the attack)
- Disable the opcode cache
- On PHP 8.5, turn off fatal error backtraces
- Block multipart form uploads at the network edge (breaks real uploads, but stops the attack)
- Run PHP with least privilege

One thing that does **not** help: a web application firewall that runs as a PHP module. That's because the whole attack finishes *before* the website's own application code  and therefore before any in-app firewall check ever runs. Protection has to happen either in the patched PHP runtime itself, or at a layer sitting in front of PHP (like a reverse proxy or network firewall).

For detecting this attack after the fact: look for abnormally huge `PHPSESSID` cookie values, large multipart uploads carrying the `PHP_SESSION_UPLOAD_PROGRESS` field, and PHP worker crashes / out-of-memory errors in the logs.

### Question 1: Which PHP 8.4 release fixes PHP2Shell?

**Why this is the answer:** The room states the exact patched versions across each supported branch. For the 8.4.x branch specifically, the fix landed in the very next release after the vulnerable 8.4.26 the lab target is running.

**Answer:** `8.4.27`

### Question 2: Which php.ini directive, set to 0, disables the feature the attack relies on for delivery?

**Why this is the answer:** The entire attack chain depends on being able to trigger the upload-progress tracking feature from Task 2. That feature is controlled by a specific php.ini setting that turns it on or off flipping it to `0` and restarting workers removes the attacker's way of triggering the code that leads to the dangling pointer in the first place.

**Answer:** `session.upload_progress.enabled`

---

## Overall Lesson

PHP2Shell is a good reminder that security doesn't stop at your application code. A single timing mistake in how PHP recovered from an out-of-memory error sitting unnoticed since 2012 gave attackers a way to run commands on PHP servers with nothing but ordinary HTTP requests, regardless of what framework or CMS was installed on top. Patching the runtime itself has to be treated with the same urgency as patching your application.
