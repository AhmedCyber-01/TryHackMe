# PHP2Shell Writeup

**Lab Link: https://tryhackme.com/room/cvephp2shell**

## What is PHP2Shell?

On 21 October 2026, security researchers found a big problem in PHP itself — not in any website built with PHP, but in PHP's own engine. This means almost any website running PHP could be hacked, even if the website's code was written perfectly.

No login needed. No file upload needed. Just a normal web request was enough to take over the server.

## How does it work? 

**Step 1: A helpful feature gets abused**

PHP has a feature that shows upload progress (like a loading bar) when you upload a file. To turn this on, you just add a special field to your upload request. The catch? This progress-tracking code runs *before* the website's own code even starts. So an attacker doesn't need to find a weakness in the website — they just need to send a request, and PHP starts working before any security checks happen.

**Step 2: PHP makes a mistake when it runs out of memory**

PHP has a memory limit. If a request uses too much memory, PHP panics and jumps out immediately — like running out of a burning building without turning off the stove first. Normally, PHP cleans up properly, but during this emergency exit, it sometimes forgets to clear a "pointer" (basically an address pointing to a piece of memory).

This leftover pointer is like a sticky note pointing to a room that's already been cleared out. The next visitor (next web request) might walk into that room not knowing it's now storing something completely different.

**Step 3: Leaking secrets**

Because the server handles one request after another without restarting, this mix-up carries over. An attacker can trick the server into handing back pieces of its own private memory — like passwords, session data, or secret keys — sent back in a cookie.

This memory leak also tells the attacker exactly where things are located inside the server's memory, breaking a protection called ASLR (which normally hides memory locations from attackers).

**Step 4: Taking control**

Once the attacker knows where things are in memory, they build a fake piece of data and plant it in that same leftover memory spot. When PHP goes to "clean up" at the end of the request, it accidentally runs the attacker's command instead of doing normal cleanup.

Result: the attacker can now run any command they want on the server, as if they were sitting at the keyboard.

## Why is this so serious?

- It affects PHP going back to 2012 — 14 years of websites.
- It doesn't matter what website software you use (WordPress, Laravel, anything)  the problem is below all of that, in PHP itself.
- Firewalls that scan website traffic usually can't stop it, because the attack finishes before the website code (and the firewall checks tied to it) even starts.

## How do you fix it?

- **Best fix:** Update PHP itself to the patched version (8.5.12, 8.4.27, 8.3.36, or 8.2.35) and restart the server.
- **Temporary fixes** if you can't update right away:
  - Turn off the upload-progress feature.
  - Turn off PHP's opcode cache.
  - Block uploads at your firewall (though this breaks real file uploads too).

## The big lesson

Even if your website's code is flawless, the tools running underneath it (like PHP) can still have hidden flaws. Keeping those updated is just as important as securing your own code.


