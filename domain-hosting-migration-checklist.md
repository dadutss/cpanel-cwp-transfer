---
name: Domain hosting migration checklist
description: >-
  Use when moving a domain's DNS, website, FTP, or email from one hosting server
  or control panel to another (for example cPanel to CWP), to check that nothing
  is missed.
---
# Domain hosting migration checklist

Goal: move only what the domain actually uses. If email stays at an external provider, do not point MX or SPF at the new web server.

## 0. Baseline
1. Get the domain, the old panel's login address, and the new server's IP.
2. Pull public DNS (NS, SOA, A, AAAA, MX, TXT, CAA, and common hosts like www, mail, ftp, webmail, _dmarc, default._domainkey, and Mailgun hosts such as `mg.<domain>`) with DNS-over-HTTPS (`https://dns.google/resolve?name=...&type=...`) if `dig` isn't available. Check more than one public resolver; they can disagree.
3. Read the full zone in the old panel's Zone Editor, since public lookups can't list every record.
4. Note every record whose value is the old server's IP; those get the new IP, not a straight copy. Leave records that point at an external mail or sending provider as they are.

## 1. Publish CWP zone changes to InMotion nameservers
Whenever a CWP zone is created or edited (account creation, mail lock-in, a preview hostname, or web A cutover):
- Edit the zone with DNS Editor → Edit File, bump the SOA serial, and reload the zone in CWP.
- Then run this command on the CWP server to push the current zone to InMotion public nameservers:
  `/opt/imh-cwp-dns/update_zone <domain>`
  Replace `<domain>` with the domain you just changed, for example `/opt/imh-cwp-dns/update_zone example.com`.
- CWP reload alone is not enough. InMotion nameservers often ingest the auto-created zone at account creation and do not pick up later Edit File + reload until this command runs.
- Confirm more than one public resolver is serving the new SOA serial and the intended records. Flush public resolver caches only after the authoritative nameservers are correct. If the command is unavailable or the public SOA still lags, ask InMotion (Domains ticket) to serve the current zone from the new server.
- If the domain's nameservers have been moved off InMotion to an external DNS host (for example Cloudflare or Porkbun), edit and publish the zone there instead. Do not treat the CWP zone file as live, and skip this InMotion push command.

## 2. Website
- Root A record with the new IP, but only after the new site copy is verified.
- `www` CNAME to the root (or A with the new IP).
- AAAA only if the new server has IPv6.
- For a pre-cutover preview, use a temporary hostname such as `new.<domain>` pointed only at the new server, with the same document root.

## 3. FTP
- `ftp` CNAME to the root (or A with the new IP).
- Confirm FTP accounts exist on the new server.

## 4. Mail routing (MX)
First decide whether mail stays off-box or moves to the new server.

If mail stays at an external provider (for example Rackspace Email):
- Keep the existing MX, SPF, DKIM, and DMARC. Do not replace them with the new panel's defaults.
- In CWP, lock this in immediately after the account is created. The auto-created zone can publish MX pointing at the domain itself, MX targets missing trailing dots, and an SPF for the new server IP, which will break inbound and outbound mail.
- Prefer CWP DNS Editor → Edit File, and use trailing dots on MX and CNAME targets.
- After saving and reloading, push the zone to InMotion nameservers (see section 1) and confirm public resolvers actually serve the corrected serial and records, not only that the file on the new server looks right. Flush public resolver caches only if they are still split after the authoritative zone is correct.

If mail is moving onto the new server:
- Root MX (cPanel default is `0 <domain>`), plus `mail` CNAME or A.
- SPF TXT updated to the new IP (`v=spf1 +a +mx +ip4:NEW_IP ~all`). Needed for forwarders too, since the new server re-sends that mail.
- DKIM: don't copy the old `default._domainkey`; publish the key the new server generates.
- DMARC: copy `_dmarc` as-is.
- Skip root MX only if no mailbox or forwarder on the domain is used.
- After saving and reloading, push the zone as in section 1.

## 5. Forwarders and mail accounts
- Only if mail is moving onto the new server: list every forwarder and mailbox in the old panel (Forwarders and Email Accounts pages); they are not in DNS.
- Recreate them on the new server and send a test message to each forwarder.
- If mail stays off-box, leave those accounts at the external provider.

## 6. Mailgun (or other outside email service)
- Check for subdomains like `mg.<domain>`. If the public lookup is empty, pull the required records from the Mailgun domain settings (receiving MX, SPF TXT, DKIM TXT, tracking CNAME). These point at Mailgun, so they copy as-is.
- Mailgun's "valid" status can be stale; compare against live DNS.
- After the new site copy is up on a preview hostname, send a test from a contact form or other Mailgun path before switching live website DNS.
- After cutover, run Mailgun's domain verify if records changed.

## 7. Other records to look for
- Verification TXT records (Google, Microsoft, etc.), CAA, SRV, and any custom subdomains.

## 8. Cutover and verify
- Lower TTLs ahead of time if possible.
- Switch only the records that should move (usually website and FTP A/AAAA). Do not switch nameservers or mail records unless that was the plan.
- After any CWP zone edit, push the zone as in section 1, then recheck each record against more than one public resolver, load the site, test FTP, and test mail (external provider or on-box forwarders, plus Mailgun if used).
- If resolvers disagree after the authoritative zone is correct, flush major public DNS caches and wait for remaining TTLs.
