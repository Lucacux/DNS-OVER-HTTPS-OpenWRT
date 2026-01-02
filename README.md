# OpenWrt DNS-over-HTTPS Configuration

This repository contains a simple and reproducible configuration file for enabling **DNS-over-HTTPS (DoH)** on an OpenWrt router.

The goal of this setup is to improve **privacy and security** by encrypting DNS queries, preventing local network observers or ISPs from inspecting or manipulating DNS traffic.

## What this file does

The `openwrt-config.txt` file includes:

- Installation of the `https-dns-proxy` package
- Configuration of a DoH resolver (Cloudflare)
- Redirection of DNS queries through the local DoH proxy
- Cleanup of conflicting DNS settings in `dnsmasq`

This configuration is intended to be:
- Executed manually over SSH **or**
- Copied directly into an OpenWrt system for quick setup

## Use case

This setup is useful if you want to:
- Protect DNS queries from passive monitoring
- Avoid DNS tampering or ISP-level filtering
- Harden a home or lab OpenWrt router with minimal complexity

## Notes

- This configuration does **not** include a VPN.
- It focuses only on DNS encryption.
- Tested on **OpenWrt 24.10.5 (r29087-d9c5716d1d)**
  LuCI: **openwrt-24.10 branch (25.340.26705~d88390b)**
## File structure

- `openwrt-config.txt` — DNS-over-HTTPS configuration commands
- `README.md` — Project documentation

## Screenshot / Reference
![OpenWrt DNS-over-HTTPS](Screenshot_20260101_230552.png)
