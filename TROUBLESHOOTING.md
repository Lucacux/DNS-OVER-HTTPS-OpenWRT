# Troubleshooting — DoH DNS outage

Post-mortem of a full LAN DNS outage on this DoH setup after upgrading to
**OpenWrt 25.12.5** (`r33051-f5dae5ece4`). Internet worked by IP, but **no name
resolved** on any client. Two independent faults were stacked.

## Symptom

- `ping 1.1.1.1` → OK (Internet/routing fine).
- `nslookup <name> 192.168.1.1` → `connection timed out; no servers could be reached`.
- DHCP still worked (leases handed out normally), which hid the fact that DNS was down.

## Diagnosis flow

Separate *connectivity* from *resolution*, then isolate each link of the chain
`client → dnsmasq → https-dns-proxy (DoH) → upstream`.

```sh
# 1. Connectivity vs DNS
ping -c3 1.1.1.1                      # OK  -> not a routing problem
nslookup openwrt.org 192.168.1.1     # timeout -> DNS problem

# 2. Is the DoH proxy alive and does upstream work?
ps w | grep '[h]ttps-dns-proxy'                 # process present, listening :5053
nslookup dns.quad9.net 1.1.1.1                  # bootstrap over plain 53 -> OK
wget -qO- --no-check-certificate https://9.9.9.9/dns-query   # 443 to DoH -> OK

# 3. Is dnsmasq even listening / did its DNS get disabled?
netstat -lnup | grep ':53 '          # dnsmasq listening on :53
uci show dhcp.@dnsmasq[0]            # noresolv=1, server=...127.0.0.1#5053

# 4. Did dnsmasq actually start? (the decisive clue)
logread | grep -i dnsmasq | grep -iE 'fail|cannot'
```

> Note: with `noresolv=1` dnsmasq has **no fallback** — if the DoH proxy does not
> answer, every lookup on the LAN dies even though the Internet is up. That is the
> intended privacy trade-off, but it makes the DoH proxy a single point of failure.
>
> Also: a `DNS-LOCK` firewall rule (REJECT port 53 to WAN) makes `dig @8.8.8.8`
> from a LAN client fail **by design** — don't read that as the bug.

## Root cause #1 — `adblock-fast` crashes a jailed dnsmasq on 25.12

`logread` showed, on every dnsmasq (re)start:

```
daemon.crit dnsmasq[1]: cannot access /tmp/dnsmasq.cfg01411c.d/adblock-fast: No such file or directory
daemon.crit dnsmasq[1]: FAILED to start up
```

On 25.12 dnsmasq runs inside a **ujail**. `adblock-fast` (v1.2.4, mode
`dnsmasq.conf`, ~40k domains) injects its blocklist as a `conf-dir` symlink
(`/tmp/dnsmasq.cfg01411c.d/adblock-fast → /var/run/adblock-fast/adblock-fast.dnsmasq`)
and relies on `option addnmount` to bind-mount it into the jail. When that target
file is not present at the moment dnsmasq starts (boot-time race, or after the
proxy/adblock reorder their restarts), dnsmasq **fails to start entirely** and the
whole LAN loses DNS. A plain reboot or a package upgrade is enough to trigger it.

## Root cause #2 — `https-dns-proxy` wedged (alive but not answering)

With dnsmasq finally starting cleanly, lookups still timed out. The proxy process
was alive, listening on `127.0.0.1:5053`, bootstrap resolved and `:443` to the
resolver was reachable — but it returned nothing. A plain
`/etc/init.d/https-dns-proxy restart` did **not** clear it; a full **stop + start**
(new PID) did. Likely a stale upstream connection left over from the chaotic
upgrade boot.

## Fix applied

```sh
# --- Fault #1: take adblock-fast out of the dnsmasq path (chosen: disable it) ---
/etc/init.d/adblock-fast stop
/etc/init.d/adblock-fast disable                 # stop it crashing dnsmasq on boot
rm -f /tmp/dnsmasq.cfg01411c.d/adblock-fast      # drop the dangling conf symlink
uci -q delete dhcp.@dnsmasq[0].addnmount         # drop the stale jail bind-mount
uci commit dhcp
/etc/init.d/dnsmasq restart                      # now starts clean

# --- Fault #2: fully recycle the DoH proxy (restart is NOT enough) ---
/etc/init.d/https-dns-proxy stop
/etc/init.d/https-dns-proxy start

# --- Verify end to end ---
nslookup openwrt.org 127.0.0.1                   # resolves via DoH -> rc=0
for d in google.com github.com cloudflare.com; do nslookup "$d" 127.0.0.1; done
```

## Prevention

- **adblock-fast on a jailed dnsmasq is fragile.** If you keep it, make sure it
  generates its blocklist *before* dnsmasq starts and that `addnmount` points to an
  existing file; otherwise disable it (done here) and rely on DoH alone.
- Because `noresolv=1` leaves no fallback, consider a lightweight watchdog that
  does a test lookup and runs `stop`/`start` on `https-dns-proxy` if it stops
  answering — a cron check avoids a wedged proxy becoming a silent LAN-wide outage.
- Remember that upgrading from 24.10 to 25.12 reset the resolver to the package
  default (**Quad9**); re-apply `openwrt-config.txt` if you want Cloudflare back.

---
_Context: observed on OpenWrt 25.12.5. Diagnosed and fixed over SSH, 2026-10-01._
