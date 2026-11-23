# Verifying mailcrypt

mailcrypt runs entirely in your browser, but the page itself comes from a server. If that
server were compromised, it could serve a page that looks the same but leaks your messages
or password. Check the page against a hash published somewhere other than that server.

Page: https://hoddmimes.com/mailcrypt/

## 1. Download the page

Use `curl` or `wget`. A browser's "Save page as" may rewrite the HTML, and then the hash won't match.

```bash
curl -fsSo mailcrypt.html https://hoddmimes.com/mailcrypt/
```

On Windows (PowerShell):

```powershell
Invoke-WebRequest https://hoddmimes.com/mailcrypt/ -OutFile mailcrypt.html
```

## 2. Compute its SHA-256 hash

| System | Command |
|---|---|
| Linux | `sha256sum mailcrypt.html` |
| macOS | `shasum -a 256 mailcrypt.html` |
| Windows (PowerShell) | `Get-FileHash mailcrypt.html -Algorithm SHA256` |
| Windows (cmd) | `certutil -hashfile mailcrypt.html SHA256` |

Windows prints the hash in upper case; compare without regard to case.

## 3. Compare with the list below

The hash must match one of the released versions exactly. If it doesn't, don't use the page.

## 4. Keep and use your checked copy

A check proves only what *that* download contained; the server could serve a different
page another time. For real protection, keep the file you checked and open it from disk
(double-click it) instead of visiting the website:

- **Encrypt:** works the same as online. Mails you write still contain a link to the website
  for recipients who don't keep their own copy.
- **Decrypt:** open your copy, choose **Decrypt**, and paste the whole block or the link from the
  mail. It finds the message in either. The page makes no network requests, so it works offline.

## Released versions

Newest first. Every change to the page, however small, gives a new hash.

<!-- tools/deploy.sh adds a row here when it deploys a hash that isn't listed yet. -->
| Date | SHA-256 of mailcrypt.html | Notes |
|---|---|---|
| 2026-11-23 | `ac5682b7bdc582ea59bd1585b2cf163f5e7086056d756fc7f165730acce4d393` | First release |

## Where this list should live

This file only helps if an attacker who controls hoddmimes.com can't change it too. Publish it
somewhere independent (for example GitHub), not on hoddmimes.com or git.hoddmimes.com, which run
on the same machine. You can also send the current hash to the people you write to over Signal,
together with the password.
