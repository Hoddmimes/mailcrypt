# mailcrypt

Password-based encryption for ordinary email, done entirely in the browser.

Live at **https://hoddmimes.com/mailcrypt/**. Check the page before you trust it; see [VERIFY.md](VERIFY.md).

## Purpose

mailcrypt lets two people exchange confidential text over any mail client without installing
anything, creating accounts or exchanging keys. They agree on a password some other way
(Signal, SMS, phone). Then:

1. The sender opens the page, writes the message, enters the password and clicks **Encrypt**.
2. They paste the output, a link plus an armored text block, into a normal mail.
3. The recipient clicks the link (or opens the page and pastes the block), enters the password and
   reads the message. **Write an encrypted reply** brings the password over to the encrypt view.

The server only delivers a static HTML file. It never sees the plaintext, the password or the
ciphertext.

This is a deliberate trade-off: a shared static password is weaker than public-key mail
encryption (PGP, S/MIME), but it works for people who will never set those up. See
[Security](#security) for what this does and does not protect against.

## What the mail looks like

```
Encrypted message. Open this link and enter the password you were given:
https://hoddmimes.com/mailcrypt/#v1.<salt>.<iv>.<ciphertext>

-----BEGIN MAILCRYPT MESSAGE-----
Decrypt at: https://hoddmimes.com/mailcrypt/
v1.Xq3...   (the same token, wrapped at 64 characters)
-----END MAILCRYPT MESSAGE-----
```

- The ciphertext goes in the URL **fragment** (after `#`). Browsers never send the fragment to
  the server, so it doesn't appear in web server logs.
- The armored block is always included because mail clients can break long links. The link is
  left out when it would be longer than 2000 characters, or when the page has no public address
  (opened from disk with `mailcrypt-home` empty).
- The decrypt parser accepts quoted replies (`> `), re-wrapped lines and surrounding mail text.
  It finds the token in a pasted block, a pasted link or a bare token.

## Implementation

The whole application is one self-contained file, [`mailcrypt.html`](mailcrypt.html): HTML,
one inline `<style>` and one inline `<script>`. There are no dependencies, no build step and no
network requests.

### Cryptography (WebCrypto only)

| Item | Value |
|---|---|
| Cipher | AES-256-GCM (128-bit tag) |
| Key derivation | PBKDF2-HMAC-SHA256, 600,000 iterations |
| Salt | 16 random bytes, new for every message |
| IV / nonce | 12 random bytes, new for every message |
| Associated data | the constant string `mailcrypt-v1` |
| Randomness | `crypto.getRandomValues` |
| Token format | `v1.` + base64url(salt) + `.` + base64url(iv) + `.` + base64url(ciphertext‖tag) |

The `v1` prefix fixes every parameter above, so the format can change later (for example to
Argon2id or OpenPGP) without breaking old messages. A page that sees an unknown version says
the message was made with a newer mailcrypt.

**Passwords** are normalised the same way on both sides before key derivation: Unicode NFC,
leading and trailing whitespace trimmed, internal runs of whitespace collapsed to one space. This
way a password typed slightly differently still works. Encryption requires at least
8 characters, counted after normalisation by code point. Decryption doesn't enforce the minimum.

### Code layout

The script has two parts:

- **`MC` core**: no DOM access. It has `encrypt`, `decrypt`, `extractToken`, `mailText` and
  password helpers, and is exported through `module.exports` when that exists so the tests can
  run the exact code the page ships.
- **UI**: tabs, show/hide password, copy to clipboard, loading the token from the fragment
  (also on `hashchange`), and reply.

Opening the page with a `#v1...` fragment goes straight to Decrypt with the password field
focused. Opening it without a fragment shows Encrypt.

The address written into mails comes from `<meta name="mailcrypt-home">`. If that is empty, the
page uses its own `http(s)` address. If it is empty and the page was opened from a file, no link
is written.

### Tools

| File | Purpose |
|---|---|
| `tools/test.mjs` | Node tests of the core: round trips, wrong password, tampering, parsing of quoted and wrapped mail, version handling, password rules. Run `node tools/test.mjs`. |
| `tools/csp-hash.mjs` | Recomputes the CSP `sha256-` hashes for the inline script and style after an edit and prints the page's SHA-256. `--check` exits non-zero if they are stale. |
| `tools/deploy.sh` | Checks, tests, installs and verifies a deployment (see below). |
| `deploy/mailcrypt.conf` | Apache `<Directory>` block with the security headers. |
| `VERIFY.md` | How users verify the page, and the list of released page hashes. |
| `DESIGN-NOTES.md` | Design decisions and alternatives considered. |

Development workflow: edit `mailcrypt.html`, run `node tools/csp-hash.mjs` (or the browser will
block the edited script), then run `node tools/test.mjs`. Node 18 or later is needed (global
WebCrypto).

## Security

### What it protects against

- **Mail providers, relays, backups and anyone reading the mailbox** see only ciphertext. They
  can't read it without the password.
- **Tampering**: AES-GCM authenticates the ciphertext. If the message is changed or damaged,
  decryption fails ("Wrong password, or the message was damaged") instead of showing altered text.
- **Server logs**: the ciphertext is in the URL fragment or pasted into the page, so it never
  reaches the server. The page also sends `Referrer-Policy: no-referrer`.
- **Exfiltration by injected code**: the page sets a strict Content Security Policy in a
  `<meta>` tag, which also applies to saved copies:
  `default-src 'none'; script-src 'sha256-…'; style-src 'sha256-…'; base-uri 'none'; form-action 'none'`.
  Only the exact inline script and style can run, and the page can't load or send anything
  (no `fetch`, images, fonts or form posts). The server also sends
  `frame-ancestors 'none'` (no framing or clickjacking) and `X-Content-Type-Options: nosniff`.

### The main risk: trusting the server

A browser-crypto page is only as trustworthy as the server that delivers it. If someone
compromises hoddmimes.com, they can serve a modified page with a new CSP that leaks passwords
and messages. Mitigations:

- The page is a single static file with no third-party code, so it is easy to audit and hash.
- Each released version's SHA-256 is listed in [VERIFY.md](VERIFY.md). That list should be
  published somewhere the server's owner can't change (for example GitHub), not on hoddmimes.com.
- **Best practice:** download the page once, check its hash, and open that local copy from disk.
  It works fully offline and decrypts both pasted blocks and pasted links. A checked local copy
  is not affected by later server changes.

### Known limitations (accepted)

- **The password is the only secret.** Anyone with the password can read every message
  encrypted with it and can also write messages that decrypt correctly. There's no sender
  authentication and no forward secrecy. Reusing a password for many messages or many people
  increases the exposure.
- **Offline guessing.** Someone with a captured message can try passwords without limit.
  PBKDF2 at 600,000 iterations slows each guess, but short or common passwords can still be
  found. Use a long passphrase of random words. Whitespace normalisation makes "a  b" and "a b"
  the same password, which reduces the number of possible passwords very slightly.
- **Message length is visible.** There's no padding, so the ciphertext length shows the
  plaintext length (plus a fixed overhead).
- **Endpoint security.** Malware or browser extensions on either machine can read the page. The
  decrypted text, the clipboard and the browser history (the fragment link) stay on the device.
- **Availability.** Messages depend on a copy of the page (or a reimplementation of the
  documented format) being available. The format is plain WebCrypto, so it can be decrypted
  independently, but not with standard tools like `gpg`.
- **Metadata** (sender, recipient, subject, time) is ordinary mail and isn't protected. Don't
  put secrets in the subject line.

## Deployment

The page is a static file. Any HTTPS web server works; there's nothing to run server-side.

### Current deployment (hoddmimes)

- Host `hoddmimes`: Fedora, Apache, vhost `hoddmimes.com`, docroot `/var/www/hoddmimes`.
- Page: `/var/www/hoddmimes/mailcrypt/mailcrypt.html`, served at `https://hoddmimes.com/mailcrypt/`.
- Config: [`deploy/mailcrypt.conf`](deploy/mailcrypt.conf) → `/etc/httpd/conf.d/mailcrypt.conf`
  (a server-level `<Directory>` block, so the vhost file is not touched). It sets
  `DirectoryIndex`, disables indexes and symlinks, and adds the headers
  `Content-Security-Policy: frame-ancestors 'none'`, `X-Content-Type-Options: nosniff`,
  `Referrer-Policy: no-referrer` and `Cache-Control: no-cache`.

Deploy with:

```bash
tools/deploy.sh                 # default host: hoddmimes
HOST=otherhost tools/deploy.sh  # same layout on another host
```

The script:

1. Fails if the CSP hashes are stale (`csp-hash.mjs --check`) or a test fails.
2. Copies the page and config over SSH and installs them with `sudo`.
3. If the Apache config changed: backs up the old one (outside `conf.d`), runs `httpd -t`,
   restores the backup if the test fails, and otherwise reloads Apache.
4. Downloads the live page and compares its SHA-256 with the local file. It fails on a mismatch.
5. If the hash is new, adds a row to the release table in `VERIFY.md`. Fill in the Notes column
   and **publish the updated list somewhere independent** of the server.

Requirements: SSH access to the host with `sudo`, and `node`, `curl`, `sha256sum` locally.

### Deploying elsewhere

1. Set `<meta name="mailcrypt-home" content="https://your.host/path/">` in `mailcrypt.html` to the
   public address, or leave it empty to use whatever address the page is served from. Run
   `node tools/csp-hash.mjs` afterwards. (Changing the meta tag doesn't change the CSP hashes,
   but it does change the page hash you publish.)
2. Serve the file over HTTPS. Add the same headers as `deploy/mailcrypt.conf`. In nginx, for example:
   ```nginx
   location /mailcrypt/ {
       index mailcrypt.html;
       add_header Content-Security-Policy "frame-ancestors 'none'" always;
       add_header X-Content-Type-Options "nosniff" always;
       add_header Referrer-Policy "no-referrer" always;
       add_header Cache-Control "no-cache" always;
   }
   ```
   Don't add a header CSP that is looser than the meta one. Browsers apply both, so a header
   can only make the policy stricter, but a stray `script-src` there would block the page.
3. Publish the resulting SHA-256 for your users, as in `VERIFY.md`.
4. To use `tools/deploy.sh`, change `URL` and the paths under `/var/www/hoddmimes` and
   `/etc/httpd` to match your server.

## Verifying a copy

```bash
curl -fsSo mailcrypt.html https://hoddmimes.com/mailcrypt/
sha256sum mailcrypt.html      # compare with VERIFY.md
```

See [VERIFY.md](VERIFY.md) for macOS and Windows commands and the list of released hashes.
