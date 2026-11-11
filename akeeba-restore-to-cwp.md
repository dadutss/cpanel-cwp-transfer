---
name: Akeeba restore to CWP
description: >-
  Use when moving a Joomla site onto a CWP server by restoring an Akeeba backup,
  especially before public DNS is pointed at the new server.
---
# Akeeba restore to CWP

Goal: restore a Joomla site onto a CWP account without touching the live site or public web DNS until the restore is verified. Prefer the control-panel GUI and File Manager. For DNS, MX, SPF, DKIM, DMARC, and Mailgun records, also follow the domain-hosting-migration checklist. Leave external email in place unless the owner asked to move mail.

Manage PHP and other server settings strictly per site. Never change the server-wide PHP default or another live site's PHP to fix the site being moved.

## 1. Baseline the live site
- Confirm the site is not already on the new server.
- Record live DNS (NS, SOA, A/www/ftp/webmail, MX, SPF, DKIM, DMARC, Mailgun) and where email actually lives (for example Rackspace vs on-box).
- If `/administrator` redirects to the homepage, the site likely uses Akeeba Admin Tools with a secret URL. Use the owner's usual query parameter (check memory) rather than guessing.

## 2. Get the latest backup
- The easiest source is the live site's Akeeba Backup component (Manage Backups), not a hunt through S3.
- Download the newest full-site archive (`.jpa` / `.jps`). Do not restore or delete backups on the live site.

## 3. Create the CWP account
- Create the account, then confirm the actual username. CWP may silently truncate usernames (often to 8 characters).
- Create a MySQL database and user with full privileges, host usually `localhost`. Use a short database and username if possible; CWP password resets on long names can fail with an "Invalid userdata" error.
- Set PHP for this domain only. Copy `php.ini` and `.user.ini` from a known-good account on the same server into the new account's document root. Do not change the source account, the server default, or other sites.

## 4. Lock in email DNS immediately
Do this as soon as the account exists, before anything else public can depend on the new zone.
- CWP auto-creates a zone whose default MX/SPF can break off-box mail (MX pointing at the domain itself, or targets missing trailing dots like `mx1.example.com.thedomain.com`, and an SPF for the new server IP).
- In the DNS Editor, use Edit File rather than Edit Records. Put trailing dots on MX and CNAME targets.
- Keep the existing off-box MX, SPF, DKIM, DMARC, and Mailgun records. Leave live website A/www records on the old server.
- Reload the zone, then push it to InMotion public nameservers with `/opt/imh-cwp-dns/update_zone <domain>` on the CWP server. Replace `<domain>` with the domain you just changed, for example `/opt/imh-cwp-dns/update_zone example.com`. CWP reload alone is not enough. Confirm what public resolvers actually serve — not just the CWP file. If the command is unavailable or the public SOA still lags, have InMotion serve the current zone from the new server, and do not call mail done until public SOA, MX, and SPF match the corrected file.
- If the domain's nameservers have been moved off InMotion to an external DNS host (for example Cloudflare or Porkbun), edit and publish the zone there instead, and skip the InMotion push command.
- If public DNS is split after the authoritative zone is fixed, request a cache flush at major public resolvers (for example Google Public DNS and Cloudflare 1.1.1.1). Other ISP caches will expire on their own with the TTL.

## 5. Add the preview hostname
- Add a temporary subdomain `new.<domain>` on the new account, with the same document root as the site (not a subdirectory).
- Point only that subdomain's A record at the new server. Leave live apex and `www` on the old server.
- After adding that record in CWP, push the zone with `/opt/imh-cwp-dns/update_zone <domain>` and confirm the hostname resolves publicly before relying on it.
- Attach it to the same per-domain PHP as the site will use, and issue SSL for it when the hostname resolves.

## 6. Upload Kickstart and the archive
- Upload official Akeeba Kickstart and the backup into the new account's `public_html` with File Manager.
- Files uploaded this way may be owned by root. If PHP returns 500 and the error log mentions `min_uid`, run CWP's Fix Account Permissions so the account user owns the files.
- Newer Kickstart (9.x) shows an "Insecure setup detected" gate if the file is still named `kickstart.php`. Rename it (the owner's usual name, if known, otherwise a non-obvious name) and open that filename instead. Do not invent an unlock password unless the owner uses one.

## 7. Open Kickstart only on the new server
- Never open Kickstart or the restored site at the public domain while DNS still points at the old server.
- Prefer `https://new.<domain>/` (and the renamed Kickstart file there) as the working URL until everything is loaded.
- Fallback if that subdomain is not live yet: a new-server userdir URL such as `https://<server-hostname>/~USERNAME/`. Kickstart may work there, but after restore the site or Admin Tools firewall may allow only the apex and `www` and return HTTP 421 or refuse the host. Do not rely on a local `/etc/hosts` override: a browser using DNS over HTTPS may still hit the live site.

## 8. Restore the site
- Extract into the new `public_html`. Only delete existing files first if you are sure you are on the new account (placeholder plus Kickstart and the archive), never the live server.
- Point Joomla at the new database. If the original user or password fails, create a new short-named database and user, finish the restore with those, and delete empty unused originals only after the new ones work and the owner has approved that cleanup.
- During Site Setup, the official live URL can stay `https://<domain>`. Check the copy on `new.<domain>` instead of changing the canonical URL to the preview hostname.
- Do not delete Kickstart or the backup archive without the owner's approval.

## 9. Verify on `new.<domain>`
- Add `new.<domain>` to Admin Tools firewall allowed domains on the new copy only. Do not change the official live site URL, and do not open the public domain while it still points at the old server.
- After restore, a live `.htaccess` is often missing (only `.htaccess.admintools` may be present). The homepage then works, but SEF inner URLs 404, especially with nginx in front. The owner generates `.htaccess` from Admin Tools; write or apply it through Admin Tools (.htaccess Maker), never by hand-editing the file.
- Confirm the homepage, inner pages, SSL, the correct PHP version, and a contact-form or Mailgun send on `new.<domain>`.

## 10. Cut over web DNS only
- Only after the preview checks out and public mail DNS is correct, switch the web A/AAAA records (and `ftp`/`www` if they follow the site). `www` may stay a CNAME to the apex.
- Leave Rackspace, Mailgun, and other external mail records untouched unless the owner asked to move mail.
- After the CWP zone edit, bump the serial, reload, and push the zone with `/opt/imh-cwp-dns/update_zone <domain>`. Recheck public resolvers and flush public caches if they are still split. After cutover, issue or renew SSL for the live hostnames if needed.
- After cutover, the temporary `new.<domain>` subdomain can be removed, and Kickstart plus the archive can be removed from `public_html` so they are not left publicly reachable, both with the owner's approval.

## CWP admin notes
- Stay on the owner's usual HTTPS panel URL. Do not follow redirects to ports 2087 or 2083.
- "Security token failed" means the login form is stale; reload for a fresh token.
- "Invalid Session" after a successful login is often fixed by clearing cookies for that site and signing in again on a single tab.
