# SSH from First Principles

*Why public-key cryptography makes `git push` work — and how to run two GitHub identities on one laptop without breaking anything.*

---

## The problem SSH solves

Before 1995, engineers connected to remote machines using tools like **telnet** and **rsh**. These protocols sent everything over the wire as plain text — your username, your password, every command you typed. Anyone on the same network running a packet sniffer could read your session in real time.

SSH (Secure Shell) was created by Tatu Ylönen in 1995, after a password-sniffing attack on his university network, to solve this by encrypting the entire connection. The attacker sees only random bytes. But SSH does more than encrypt — it also solves the **authentication problem**: how does the server know you are who you claim to be? And how do *you* know you're talking to the real server, not an impersonator?

Both problems are solved by the same tool: **public-key cryptography**.

---

## Symmetric vs asymmetric encryption

**Symmetric encryption** uses one key to encrypt and decrypt. It's fast, but has a fatal bootstrap problem: how do you share the key securely with the other party? If you could share a secret securely, you wouldn't need encryption in the first place.

**Asymmetric encryption** breaks out of this loop with two mathematically linked keys:

- **Public key** — share it with everyone. Post it online. Put it in your GitHub settings.
- **Private key** — never leaves your machine. Ever.

The critical property: what one key encrypts, only the other can decrypt. And you cannot derive the private key from the public key — computing it would take longer than the age of the universe with current hardware.

This gives you two capabilities:

**Encryption** — anyone can encrypt a message with your public key; only you can decrypt it with your private key.

**Signing** — you encrypt (sign) something with your private key; anyone with your public key can verify it. Since only you have the private key, a valid signature proves you created it. This is what SSH authentication actually uses.

> **The padlock analogy.** Alice sends Bob an open padlock — unlocked, anyone can have one. Bob puts his message in a box, clicks the padlock shut. Now only Alice, with the physical key, can open it. SSH signing works in reverse: Alice locks something with her private key (the physical key), and anyone holding her public key (the padlock) can verify it was her.

---

## The SSH handshake

When you type `ssh git@github.com`, this sequence happens before any data flows:

```
Your Machine                                   GitHub Server
     |                                               |
     |-------- TCP connection on port 22 ---------->|
     |                                               |
     |<-- Server's HOST public key + algorithms -----|
     |    (your client checks this against           |
     |     ~/.ssh/known_hosts)                       |
     |                                               |
     |===== Diffie-Hellman key exchange ============|
     |    Both sides compute a shared session key    |
     |    without ever transmitting it.              |
     |    All traffic is now encrypted.              |
     |                                               |
     |<------- Random CHALLENGE (blob of bytes) -----|
     |                                               |
     | [your machine signs the challenge             |
     |  with your PRIVATE key]                       |
     |                                               |
     |-------- Signed challenge ------------------>|
     |                                               |
     |         [server verifies with YOUR            |
     |          PUBLIC KEY from authorized_keys]      |
     |                                               |
     |<---- Authentication successful ---------------|
```

The critical insight: **your private key never left your machine**. The server sent a random challenge; you signed it; the server verified the signature. Since only the holder of the private key can produce a valid signature for that specific challenge, identity is proven without transmitting any secret. This is called challenge-response authentication.

### What is Diffie-Hellman?

It's the magic in step 3 — establishing a shared session key over a public channel without transmitting the key itself. Simplified: both sides agree on a public "color," each picks a secret color and mixes it with the public one, they swap the mixed results, and each mixes the received mix with their secret. Both arrive at the same final color. An eavesdropper sees the public color and the two mixed colors but cannot compute the final shared secret — it's mathematically infeasible. The encrypted tunnel is built from this shared secret.

---

## Key types

| Type | Algorithm | Status |
|------|-----------|--------|
| `rsa` | Integer factoring | Still works, legacy |
| `dsa` | DSA | Deprecated — don't use |
| `ecdsa` | Elliptic curve DSA | Fine, but Ed25519 is better |
| `ed25519` | Edwards curve (Curve25519) | **Use this** |

Ed25519 keys are smaller, faster to generate, faster to verify, resistant to timing attacks, and have no weak parameter choices. The only reason to use RSA is compatibility with servers predating 2014.

```bash
# Generate a personal Ed25519 key
ssh-keygen -t ed25519 -C "you@gmail.com" -f ~/.ssh/id_ed25519_personal
# -t ed25519       → algorithm
# -C "..."         → comment embedded in public key (label only, not cryptographic)
# -f path          → where to save (avoids overwriting existing keys)

# Two files are created:
~/.ssh/id_ed25519_personal      # private key — never share this
~/.ssh/id_ed25519_personal.pub  # public key — safe to share everywhere
```

The public key is a single line:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ4cRPsfn7hfBUdbl1+ua3PFOtSP8QnvacCunhfqSYuD you@gmail.com
#  ↑ type    ↑ key material (base64)                                                    ↑ comment
```

### File permissions matter

SSH refuses to use a private key file that's readable by others:

```bash
chmod 700 ~/.ssh                        # directory: only you can read/write/enter
chmod 600 ~/.ssh/id_ed25519_personal    # private key: only you
# Public key can be 644 — it's meant to be shared
```

### The passphrase

When ssh-keygen asks for a passphrase, it encrypts the private key file at rest. Even if someone steals the file, they can't use it without the passphrase. The SSH agent (see below) lets you enter it once per session.

---

## The ~/.ssh/ directory

```
~/.ssh/
├── config                     ← client configuration (host aliases, key routing)
├── known_hosts                ← servers you've connected to (their fingerprints)
├── authorized_keys            ← (on servers) public keys allowed to log in
├── id_rsa                     ← a private key (RSA, legacy)
├── id_rsa.pub                 ← its public key
├── id_ed25519_personal        ← a private key (Ed25519, modern)
└── id_ed25519_personal.pub    ← its public key
```

**authorized_keys** is the server-side file. When you paste a public key into GitHub's SSH settings, GitHub appends it to an `authorized_keys` file associated with your account. When you connect, the server uses this to verify your signature:

```
# Each line = one public key allowed to log in
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ4cRPsfn7... you@gmail.com
ssh-rsa     AAAAB3NzaC1yc2EAAAADAQABAAACAQDN...    you@corp.com
```

---

## known_hosts and Trust On First Use

SSH authenticates mutually — the server proves its identity too. Every SSH server has its own key pair (the "host key"). On first connection, SSH shows you the server's fingerprint:

```
The authenticity of host 'github.com (20.207.73.82)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

When you type `yes`, the fingerprint is saved to `~/.ssh/known_hosts`. Every subsequent connection, SSH verifies the fingerprint matches. A mismatch triggers a loud warning — protection against someone swapping in a fake server between connections.

> GitHub publishes their official fingerprints at `docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints`. Verify before accepting on first connection.

This is **Trust On First Use (TOFU)**: the first connection is when you're most vulnerable; after that you're protected against future changes.

---

## SSH agent

If your private key has a passphrase, you'd type it on every `git push`. The SSH agent solves this: a background process that holds decrypted private keys in memory. You unlock once via `ssh-add`; the agent handles all signing for the rest of the session. The key material never leaves the agent — SSH asks it to sign things rather than getting the key itself.

```bash
# Start the agent (usually auto-started on macOS)
eval "$(ssh-agent -s)"

# Add a key — prompts for passphrase once
ssh-add ~/.ssh/id_ed25519_personal

# On macOS — save passphrase to Keychain (persists across reboots)
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_personal

# See what's loaded
ssh-add -l
# 256 SHA256:+DiY3wv... you@gmail.com (ED25519)
```

---

## ~/.ssh/config

The config file sets default options for SSH connections — and more importantly, lets you define **host aliases** that map to different keys. SSH reads top-to-bottom; first match wins for each setting.

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa

Host github.com-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
```

`github.com-personal` is not a real hostname — it's an alias that only exists in this file. When SSH sees it, it connects to `github.com` (the real `HostName`) but uses the personal key. The alias in the remote URL is how you control which key is used.

---

## Multiple GitHub identities on one machine

One machine, two GitHub accounts, one SSH binary. The problem: by default SSH tries keys in a fixed order — the corporate key wins and you authenticate as your corporate identity for everything.

The solution is the host alias. Each repo's remote URL contains the alias, which maps to a specific key in `~/.ssh/config`:

```bash
# Corporate repo — URL uses "github.com" → id_rsa → authenticates as corp-user
git remote -v
# origin  git@github.com:corp/service.git

# Personal repo — URL uses "github.com-personal" → id_ed25519_personal → authenticates as personal-user
git remote -v
# origin  git@github.com-personal:personal-user/repo.git
```

When you run `git push` in the personal repo, Git calls SSH with `github.com-personal` as the hostname. SSH matches your config block, resolves it to the real `github.com`, and uses `id_ed25519_personal`. GitHub sees the personal key and grants access to your personal account. The corporate repos are completely unaffected — their URLs use `github.com` which matches the first block.

### Setting up the personal remote

```bash
git remote add origin git@github.com-personal:your-username/repo.git
# Note: "github.com-personal" not "github.com"
# This one URL change is the entire isolation mechanism
```

### Testing each identity

```bash
ssh -T git@github.com           # Hi corp-user!
ssh -T git@github.com-personal  # Hi personal-user!
```

If the second one asks for a username and password, the key auth failed. The key is either not in the agent (`ssh-add -l` to check) or not uploaded to GitHub (Settings → SSH and GPG keys → New SSH key → paste `.pub` contents).

---

## Debugging

`ssh -v` is your primary tool:

```bash
ssh -vT git@github.com-personal 2>&1 | grep -E "config|Offering|accepts|succeeded|denied"
```

Key lines to look for:

```
debug1: /Users/you/.ssh/config line 11: Applying options for github.com-personal
debug1: Hostname is 'github.com'              ← alias resolved correctly
debug1: Offering public key: ~/.ssh/id_ed25519_personal
debug1: Server accepts key                    ← accepted ✓
debug1: Authentication succeeded
```

If the key is rejected and SSH falls back to other keys or interactive auth, the public key is not in GitHub's settings for that account.

```bash
ssh-keygen -R github.com        # remove stale known_hosts entry (if server key changed)
ssh -G github.com-personal      # show all effective config options for this alias
ssh-keygen -l -f ~/.ssh/id_ed25519_personal.pub  # show fingerprint (compare to GitHub's settings page)
```

---

## The complete picture

When you run `git push` in a repo with `git@github.com-personal:username/repo.git` as the remote:

1. Git reads the remote URL, calls SSH with hostname `github.com-personal`
2. SSH reads `~/.ssh/config`, matches the `Host github.com-personal` block
3. SSH connects to `github.com` (the real HostName) on port 22
4. GitHub sends its host key — SSH verifies against `known_hosts`
5. Both sides run Diffie-Hellman — an encrypted session key is established
6. GitHub sends a random challenge
7. SSH agent signs the challenge with `id_ed25519_personal`
8. GitHub verifies the signature against the public key stored for your personal account
9. Authentication succeeds — you're identified as your personal account
10. `git push` proceeds over the encrypted, authenticated connection

Your private key never left your machine. The corporate setup was never touched. Two identities, clean isolation, one config file.

---

## Further reading

- Ylönen & Lonvick, [*The Secure Shell (SSH) Protocol Architecture*](https://tools.ietf.org/html/rfc4251) — the RFC
- Boneh & Shoup, [*A Graduate Course in Applied Cryptography*](https://toc.cryptobook.us/) — the math behind asymmetric crypto
- GitHub, [*Connecting to GitHub with SSH*](https://docs.github.com/en/authentication/connecting-to-github-with-ssh) — official setup docs
- GitHub, [*SSH key fingerprints*](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints) — verify before trusting on first connect
