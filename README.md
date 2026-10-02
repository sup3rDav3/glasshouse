# GLASSHOUSE: 41 Days Watching the Internet Try to Break In

A few months back I spun up a small DigitalOcean droplet, pointed an SSH
honeypot (Cowrie) at it, and basically let it sit there as bait. No real
service, no DNS record anyone would ever find, nothing pointing to it.
Just an open port on the internet. I let it run for a little over six
weeks before shutting it down, and I wanted to actually look at what
landed in the logs instead of just letting them pile up.

The short version: it's busier out there than I expected, and a decent
chunk of the traffic wasn't just scanning. It was a working, self-cleaning
worm chain doing exactly what it was built to do.

## The numbers

The box was live from August 22nd through October 1st, 40 days and
change. In that window it logged 246,695 sessions from 7,883 distinct
IPs, 2.85 million login attempts, and 1.86 million commands typed (or
rather, sent, more on that below). That works out to something like
6,000 connection attempts a day, on a server that had never been
mentioned anywhere.

Of those sessions, 95.3% ended in Cowrie reporting a "successful" login.
That number looks terrifying out of context but it isn't real. Cowrie is
configured on purpose to accept whatever credentials it's handed, so it
can keep watching what happens next instead of just logging a rejected
password. The actual takeaway from that stat is the opposite of alarming:
almost nobody paused to check whether they were actually in before
immediately running commands, which tells you plainly that this is
automated traffic, not a person at a keyboard.

## What they tried

`root` showed up in 1,169,617 of the 2.85 million login attempts, 41% of
everything thrown at the box. The password side was exactly what you'd
guess: `password`, `123456`, `12345`, `admin`, `1234`, `111111`, in
roughly that order of popularity. `admin`/`admin` alone was tried 14,174
times.

What I didn't expect was how specialized some of the username lists have
gotten. `wallet` showed up 60,414 times, `crypto` 21,466, `bitcoin`
20,830, `blockchain` 15,409, `exchange` 12,562, all in the top 15 most
attempted usernames overall. These aren't generic "try everything"
dictionaries anymore. Someone built credential lists specifically aimed
at finding cryptocurrency infrastructure, and they're running them
constantly.

## The part that wasn't just noise

Scrolling through the command log, most of it is exactly what you'd
expect from scanning bots: `uname -a` over and over (1.3 million times,
more than any other single command), a few `whoami`s, a `hostname` here
and there. Boring, automated fingerprinting.

But one command chain showed up 368 times with the exact same structure
every time, and it's a real worm, not a scanner. It fingerprints the
host, sends back an obfuscated beacon that decodes to `auth_ok` (almost
certainly a signal to the operator's infrastructure that a live shell was
obtained), moves into a writable throwaway directory like `/tmp` or
`/dev/shm`, drops a hardcoded SSH key to disk, uses that key to pull down
a second payload from a remote host (falling back to `wget`/`curl` with
certificate checking turned off if the first method fails), runs it, and
then deletes the key, the config, and the payload so there's nothing left
for a casual look at the filesystem afterward.

Two other commands from what looks like the same family showed up even
more often: one that strips the immutable flag off `~/.ssh` (`chattr
-ia`), and one that wipes out `authorized_keys` and replaces it with the
attacker's own key. Put together, this isn't just "infect and move on."
It's actively locking out whatever other malware might already be
sitting on the box. Botnets compete with each other for the same
compromised hosts more than people realize.

A separate, much longer script went further than basic fingerprinting,
checking CPU core count, CPU model (with detailed lookup tables for ARM
and RISC-V chip variants), GPU presence, and system uptime. That's the
kind of profiling you do when you're deciding which architecture-specific
binary to send, or sizing up whether a box is worth turning into a
cryptominer. I'm not posting that script or the dropper's key material
here. The pattern is the interesting part, not a working copy of the
tool.

One small detail that made me smile: `/ip cloud print`, a MikroTik
RouterOS command, made the top 25 list. Even general-purpose SSH-scanning
tooling now ships with router-specific probes baked in.

## Who was doing most of it

One block of IPs (I'll show it as `109.160.32.X` to keep the exact host
out of it) was responsible for 74,043 sessions on its own. That's 30% of
every session across the entire six weeks, from a single source. I looked
up where that block actually lives: it's registered to **TechTies Inc.
(operating as GBTCloud)**, a hosting provider based in **Hilversum,
Netherlands** (ASN 197170). Not a residential ISP, not a compromised home
router. Rented cloud/VPS infrastructure, which is exactly what you'd
expect an operator to use to run scanning and C2 at scale cheaply and
disposably.

The next four busiest sources combined still didn't add up to that one
block.

As for what was actually connecting: 93.7% of all sessions identified
themselves with a generic `SSH-2.0-Go` client string, the fingerprint of
mass-scanning tooling written in Go, built for speed and portability
rather than interactive use. A handful of sessions even sent an RDP
handshake or a raw HTTP `GET /` request at the SSH port, which tells you
some of this scanning infrastructure just blasts every protocol it knows
at every open port, regardless of what's actually listening there.

Traffic wasn't constant, either. It came in bursts. September 2nd
(18,261 sessions) and the very last day before shutdown, October 1st
(18,006), were the two busiest days of the whole run, with September 30th
right behind them. Botnets seem to cycle through IP ranges on their own
schedule rather than hammering continuously.

## What I took away from this

Credential stuffing against `root` with a weak password is still, by a
wide margin, the most common thing that happens to an open SSH port on
the internet. Not a rare event, a constant one. The lists being used are
getting more targeted, not less, with crypto-specific usernames now
competing with the old standbys. And a meaningful slice of what hits an
exposed server isn't opportunistic scanning at all. It's finished,
production-quality malware running a full infect, persist, fetch,
execute, clean-up-after-itself cycle, with the operators actively
fighting other malware for the same real estate.

If you're running anything with SSH exposed and default or weak
credentials, none of this is hypothetical. It gets found and
automatically exploited on a timescale measured in hours, not months.

---

*GLASSHOUSE ran from August 22 to October 1, 2026, on a single small
cloud instance using Cowrie. Every session, login attempt, and command
was logged to a local database for analysis; this writeup summarizes the
aggregate findings. Happy to share masked IOCs (attacking ranges,
payload-staging hosts) with anyone doing defensive work who wants them
for blocklists.*
