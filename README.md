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

## Verifying a copy

```bash
curl -fsSo mailcrypt.html https://hoddmimes.com/mailcrypt/
sha256sum mailcrypt.html      # compare with VERIFY.md
```

See [VERIFY.md](VERIFY.md) for macOS and Windows commands and the list of released hashes.
