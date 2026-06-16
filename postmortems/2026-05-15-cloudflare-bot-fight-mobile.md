---
date: 2026-05-15
severity: low
category: security
affected: immich mobile app
status: resolved
---

# Cloudflare Security Level configuration (I'm under attack mode: enabled)  Blocking Mobile App

## Summary

The Immich web UI worked in browsers, but the mobile app showed cannot be reached. Root cause was Cloudflare Bot Fight Mode challenging the mobile app's HTTP client, which could not execute JavaScript challenges.

## Impact

- Mobile photo backup non-functional
- Family members could not upload photos
- Duration: ~30 minutes

## Timeline

- **08:00** Reported mobile app not connecting
- **08:05** Browser access confirmed working
- **08:10** Cloudflare Tunnel logs showed requests arriving
- **08:15** Cloudflare Security Events showed "I'm under attack mode:enabled" blocks
- **08:25** Disabled "I'm under attack mode" in Cloudflare Dashboard.
- **08:30** Mobile app connected successfully

## Root Cause

Cloudflare "I'm under attack mode"  issues JavaScript challenges to requests it deems automated. Browser clients can execute the challenge and pass. Native mobile apps using Alamofire or OkHttp cannot execute JavaScript and are blocked.

## Resolution

Disabled "I'm under attack mode" for the photos subdomain in the Cloudflare Dashboard:

Path: Security → Security Level → "I'm under attack mode" → disabled

## Lessons Learned

1. Security features can break legitimate clients
2. Test all client types: browser, mobile app, API
3. Cloudflare Tunnel is not Cloudflare Security; blocks happen at the edge

## Action Items

- [x] Disable "I'm under attack mode" for photos subdomain
- [ ] Create Cloudflare WAF rule for mobile app user-agent
- [ ] Document Cloudflare security settings that break APIs

## Supporting Evidence

[inserted later]
