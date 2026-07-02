+++
date = '2026-07-01T23:34:12+08:00'
draft = false
title = '🔑 Passwordless SSH Login Setup'
+++

## 📋 Overview

This post walks through setting up passwordless SSH login using public key authentication — how I configured it for `jellyfish@192.168.159.139` from my workstation (XPS15).

## 🤔 Why Passwordless SSH?

- 🤖 **Automation**: Scripts, Ansible, rsync, and CI/CD pipelines all need non-interactive SSH
- 🔒 **Security**: Key-based auth is more secure than passwords (no brute-force risk)
- ⚡ **Convenience**: No more typing passwords dozens of times a day

## Step 1: 🗝️ Generate an SSH Key Pair

If you don't already have a key, generate one. I use Ed25519 — it's fast, secure, and well-supported:

```bash
ssh-keygen -t ed25519 -C "your-name@your-machine"
```

Press **Enter** to accept the default `~/.ssh/id_ed25519`. For passwordless automation, leave the passphrase empty. For better security, set a passphrase and use `ssh-agent`. 🔐

### Verify the key was created:

```bash
ls -la ~/.ssh/
```

You should see both files:

```
-rw------- id_ed25519       ← private key (never share this! 🤫)
-rw-r--r-- id_ed25519.pub   ← public key (safe to share)
```

## Step 2: 📤 Copy the Public Key to the Remote Host

### Method A: `ssh-copy-id` (easiest) ✅

```bash
ssh-copy-id username@remote-host
```

This copies your public key into `~/.ssh/authorized_keys` on the remote. You'll be prompted for the remote password one last time.

### Method B: Manual install 🔧 (when ssh-copy-id isn't available)

Copy your public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Then on the remote machine:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "<paste the public key here>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

## Step 3: ✅ Test the Connection

```bash
ssh -o BatchMode=yes username@remote-host hostname
```

If it returns the remote hostname without asking for a password — you're set. 🎉

### My test output:

```
localhost.localdomain
✅ Passwordless login works!
```

## 🔬 How It Works

```
┌──────────────┐          SSH Request            ┌──────────────┐
│   Your PC    │ ──────────────────────────────►  │  Remote Host │
│              │                                  │              │
│  id_ed25519  │  ←─── Challenge/Response ────   │ authorized   │
│  (private)   │                                  │ _keys        │
└──────────────┘                                  │ (public key) │
                                                  └──────────────┘
```

1. 👋 Your SSH client sends the public key to the remote
2. 🔍 The remote checks if it matches an entry in `~/.ssh/authorized_keys`
3. ✅ If yes, the remote sends a challenge encrypted with that public key
4. 🔐 Your client decrypts it with the private key and sends a response
5. 🎉 If the response is correct — you're logged in without a password

## 📁 Key Management Tips

| File | Purpose | Permission |
|---|---|---|
| `~/.ssh/id_ed25519` | Private key — **never share** 🤫 | `600` |
| `~/.ssh/id_ed25519.pub` | Public key — safe to share ✅ | `644` |
| `~/.ssh/authorized_keys` | Remote side — list of allowed public keys 📋 | `600` |
| `~/.ssh/known_hosts` | Remote host fingerprints 🖐️ | `644` |

## 🛡️ Security Notes

- Ed25519 keys are faster and more secure than older RSA 2048-bit keys ⚡
- Always protect your private key — if compromised, remove it from all `authorized_keys` files 🚨
- Consider adding a passphrase + `ssh-agent` for everyday use 🔐

## 🔜 Next Steps

- Configure `~/.ssh/config` with host shortcuts ⚙️
- Use `ssh-copy-id` for all your servers 🚀
- Automate deployments with Ansible or rsync over SSH 🤖
