# Burp Suite — Notes

## What it actually is

Burp Suite is a **proxy that sits between your browser and whatever website you're
visiting**. Instead of your browser talking directly to a server, every request goes:

```
Browser → Burp (127.0.0.1:8080) → Target server
Browser ← Burp                  ← Target server
```

Because your traffic routes through it, Burp can *see*, *pause*, *edit*, and *replay*
every request before it leaves your machine, and every response before it hits your
browser. That's the entire premise of the tool — everything else (Repeater, Intruder,
Scanner, etc.) is built on top of that one capability.

It's used for **web application security testing** — checking whether a site's login,
forms, API endpoints, cookies, etc. can be manipulated in ways the developers didn't
intend. Relevant to you specifically for testing your own self-hosted stuff (Immich,
anything behind your Cloudflare Tunnel) — poking at your own services to see what an
attacker could see or change.

**Editions:**
- **Community (free)** — Proxy, Repeater, Intruder (rate-limited), Decoder, Comparer,
  Sequencer, Logger. Everything you need for manual testing.
- **Professional (paid)** — adds Burp Scanner, the automated vulnerability crawler.
  Not needed unless you're doing this professionally or want automated scans.

---

## Install (recap)

1. Download the `.sh` installer from portswigger.net (official site only).
2. `chmod +x burpsuite_linux_v*.sh && ./burpsuite_linux_v*.sh`
3. Installer wizard lets you pick install dir (yours landed at `~/BurpSuite`).
4. Binary: `~/BurpSuite/BurpSuite`. Jar: `~/BurpSuite/burpsuite.jar`.
5. JVM options go in `~/BurpSuite/user.vmoptions` (NOT `BurpSuite.vmoptions`, which
   gets overwritten on update).

---

## Why turn off Intercept (and how)

By default, **Proxy → Intercept is on**. This means every single request your browser
makes — every page load, every image, every background API call — gets frozen in
Burp waiting for you to manually click "Forward." Browsing becomes unusable almost
instantly; pages hang, nothing loads until you babysit each request.

Intercept is meant for the moment you actually want to catch and edit *one specific*
request (e.g. a login POST) before it goes out. It is not meant to be on while
casually browsing.

**Turn it off:**
- Proxy tab → click the **"Intercept is on"** button so it reads **"Intercept is off."**
- Traffic now flows through Burp and gets logged (visible in Proxy → HTTP History)
  without pausing anything.
- Turn it back **on** only right before the specific action you want to catch —
  e.g. flip it on, submit a login form, catch that one POST request, edit it, forward
  it, then flip it back off.

---

## Installing Burp's CA certificate (for HTTPS interception)

Browsers refuse to trust Burp's proxy for HTTPS sites unless you install Burp's own
certificate authority. You already have `cacert.der` downloaded — here's what to do
with it.

**How it was generated:** with Burp running and your browser proxied through it,
visiting `http://burp` (not https) inside that browser gives you a "CA Certificate"
download link — that's your `cacert.der`.

**Firefox:**
1. `about:preferences#privacy` → scroll to **Certificates** → **View Certificates**.
2. **Authorities** tab → **Import** → select `cacert.der`.
3. Check **"Trust this CA to identify websites"** → OK.

**Chrome/Chromium (uses the system NSS trust store on Linux):**
```bash
certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "PortSwigger CA" -i ~/Downloads/cacert.der
```
(needs `libnss3-tools` — `sudo dnf install nss-tools` on Fedora if `certutil` isn't found)
Then restart Chrome.

Once imported, HTTPS sites will show as trusted while proxied through Burp — no more
certificate warnings.

---

## Browser proxy setup (FoxyProxy)

Rather than manually editing your browser's network settings every time:
1. Install the **FoxyProxy** extension (Firefox or Chrome).
2. Add a proxy: `127.0.0.1`, port `8080`, type HTTP.
3. Toggle it on/off from the toolbar icon instead of digging through settings.

---

## The tabs, briefly

| Tab | What it's for |
|---|---|
| **Dashboard** | Overview — running tasks, issues found, event log. |
| **Target** | Site map (tree of everything Burp has seen) + Scope config (what Burp is allowed to touch). |
| **Proxy** | Live traffic. Intercept (pause/edit in real time) + HTTP History (searchable log of everything). |
| **Repeater** | Take one captured request, tweak it, resend it, see the response — repeat as many times as you want. This is where most manual testing actually happens. |
| **Intruder** | Automated fuzzing — swap a parameter value for a list of payloads and send hundreds of variations. Community edition rate-limits this. |
| **Decoder** | Encode/decode Base64, URL encoding, hex, etc. |
| **Comparer** | Byte-for-byte diff between two requests or responses. |
| **Sequencer** | Analyzes randomness of session tokens — checks if a "random" session ID is actually predictable. |
| **Logger** | Broader traffic log with filtering, beyond just Proxy history. |

---

## Basic example workflow

Say you want to check whether your Immich instance's login is doing anything you
don't expect:

1. Intercept **on**, log into Immich normally in your proxied browser.
2. Burp catches the login POST request before it's sent.
3. Right-click it → **"Send to Repeater."**
4. Forward the original request (so you're not blocking your actual login) — Intercept
   can go back **off** now.
5. In Repeater, you now have a saved copy of that exact login request. Edit fields
   directly in the request pane — change the password field, tamper with a hidden
   field, alter a header — and hit **Send**.
6. Response shows up right next to it. Compare what changed vs. the original login
   response. This is how you'd notice things like: does the server leak whether a
   username exists based on a subtly different error message, does it accept
   malformed input it shouldn't, etc.

Second common pattern — just watching:
1. Intercept off, browse your app normally for a few minutes.
2. Proxy → HTTP History now has a full log of every request the app made in the
   background — every API call, every asset load, every cookie set.
3. Sort/filter by host, status code, or MIME type to spot anything unexpected talking
   to endpoints you didn't know existed.

---

## Useful bits worth knowing

- **Scope** (Target tab → Scope) — define which hosts you actually care about, then
  filter Proxy History / Site map to "in scope only." Cuts out noise from ads,
  analytics, CDNs, etc.
- **Temporary vs. disk project** — Temporary loses everything on close (fine for
  quick checks). "New project on disk" persists your history/site map across
  sessions — worth it if you're doing a longer test on the same target over days.
- **Only test things you own or have explicit permission to test.** Running Burp
  against a third-party site without authorization is illegal in most places, even
  if you're "just looking."
