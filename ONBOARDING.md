# New Team Member Onboarding

Welcome! This guide will help you get set up with SOPS secret management using Yubikey PIV.

## What You'll Need

- [ ] Yubikey hardware device (Yubikey 5 or newer recommended)
- [ ] macOS, Linux, or WSL on Windows
- [ ] Git access to this repository

## Step-by-Step Setup

### 1. Install Required Tools

**macOS**:
```bash
brew install sops age age-plugin-yubikey yubico-piv-tool

# Verify
sops --version && age --version && age-plugin-yubikey --version
```

**Linux (Ubuntu/Debian)**:
```bash
# Install SOPS
wget https://github.com/mozilla/sops/releases/download/v3.8.1/sops_3.8.1_amd64.deb
sudo dpkg -i sops_3.8.1_amd64.deb

# Install age and Yubikey tools
sudo apt install age yubikey-manager yubico-piv-tool

# Install Rust (for age-plugin-yubikey)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Install age-plugin-yubikey
cargo install age-plugin-yubikey

# Verify
sops --version && age --version && age-plugin-yubikey --version
```

**Windows (WSL2 + Ubuntu)**:
```bash
# Install Ubuntu from Microsoft Store, then follow Linux instructions above
```

**Windows (Native - PowerShell)**:
```powershell
# Install via Chocolatey
choco install sops age

# Download Yubikey Manager from: https://www.yubico.com/support/download/

# Install Rust from: https://rustup.rs/
# Then:
cargo install age-plugin-yubikey
```

### 2. Change Your Yubikey PINs

**IMPORTANT**: Do this BEFORE generating keys!

```bash
# Change User PIN (default is 123456)
yubico-piv-tool -a change-pin

# Change PUK - PIN Unlock Key (default is 12345678)
yubico-piv-tool -a change-puk
```

**Tips**:
- User PIN: 6-8 digits you'll remember
- PUK: 8+ digits, save in password manager

### 3. Generate Your Yubikey PIV Key

```bash
# Generate key in PIV slot 1 with touch requirement
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --pin-policy always \
  --name "yourname-work"

# You'll be prompted for:
# - Management key (press Enter for default: 010203040506070801020304050607080102030405060708)
# - Your Yubikey PIN (the one you just set)

# Output will show:
# Generated key in slot 1
# Public key: age1yubikey1qd...
```

### 4. Export Your Public Recipient

```bash
# Export to file
age-plugin-yubikey --identity --slot 1 > yourname-yubikey.txt

# Verify it looks right
cat yourname-yubikey.txt
# Should show: age1yubikey1qd...
```

### 5. Create Identity File

**Important**: Tell SOPS to use your Yubikey:

**macOS / Linux**:
```bash
# Create identity file (references your Yubikey)
age-plugin-yubikey --identity > ~/.sops-age-keys.txt

# Add to shell profile (choose your shell):
# For zsh (macOS default):
echo 'export SOPS_AGE_KEY_FILE=~/.sops-age-keys.txt' >> ~/.zshrc && source ~/.zshrc

# For bash (most Linux):
echo 'export SOPS_AGE_KEY_FILE=~/.sops-age-keys.txt' >> ~/.bashrc && source ~/.bashrc
```

**Windows (PowerShell)**:
```powershell
# Create identity file
age-plugin-yubikey --identity | Out-File -FilePath "$env:USERPROFILE\.sops-age-keys.txt" -Encoding UTF8

# Set environment variable
[System.Environment]::SetEnvironmentVariable('SOPS_AGE_KEY_FILE', "$env:USERPROFILE\.sops-age-keys.txt", 'User')

# Reload (or restart PowerShell)
$env:SOPS_AGE_KEY_FILE = "$env:USERPROFILE\.sops-age-keys.txt"
```

**What this does**: Creates a file that tells SOPS "use my Yubikey to decrypt." Your private key stays on the Yubikey!

### 6. Share Your Public Recipient

Send `yourname-yubikey.txt` to your team lead via:
- Slack DM
- Email
- PR to this repository

**Note**: This is your PUBLIC key - safe to share! Never share your Yubikey or PIN.

### 7. Wait for Access

Your team lead will:
1. Add your recipient to `.sops.yaml`
2. Re-encrypt all secrets
3. Push the changes
4. Notify you when complete (usually < 5 minutes)

### 8. Verify Your Access

```bash
# Pull latest changes
git pull

# Try decrypting a test file
sops -d secrets.enc.yaml

# You'll be prompted for:
# 1. Yubikey PIN
# 2. Touch when LED blinks
```

If you see decrypted secrets, you're all set! 🎉

## Daily Workflow

### Viewing Secrets

```bash
# Quick view
sops -d secrets.enc.yaml

# With less for easier reading
sops -d secrets.enc.yaml | less
```

### Editing Secrets

```bash
# Edit (auto-decrypts, then re-encrypts on save)
sops secrets.enc.yaml

# Touch your Yubikey when LED blinks:
# - Once to open (decrypt)
# - Once to save (re-encrypt)
```

### Adding New Secrets

1. Edit encrypted file: `sops secrets.enc.yaml`
2. Add your secrets
3. Save and exit (auto-re-encrypts)
4. Commit and push:
   ```bash
   git add secrets.enc.yaml
   git commit -m "Add new credentials"
   git push
   ```

## Understanding the Touch Requirement

Every time you decrypt or encrypt, your Yubikey LED will blink - this is a **security feature**:

✅ **Good**: You're aware when secrets are accessed
✅ **Good**: Malware can't silently decrypt secrets
✅ **Good**: Physical presence required

**Always touch when LED blinks** (15 second timeout).

## Troubleshooting

### Can't Decrypt Files

**Error**: "No identities found"

**Solutions**:
```bash
# 1. Check Yubikey is connected and key exists
age-plugin-yubikey --list

# 2. Verify you're in the .sops.yaml
cat .sops.yaml | grep -A 5 "age:"

# 3. Ask team lead if you were added
```

### Yubikey Not Detected

```bash
# Check if Yubikey is seen
yubico-piv-tool -a status

# If not, try:
# - Reinsert Yubikey
# - Try different USB port
# - Restart computer
```

### Touch Timeout

If you don't touch within 15 seconds:
- No problem! Just run the command again
- Touch more quickly next time

### PIN Locked

If you enter wrong PIN 3 times, Yubikey locks:

```bash
# Unlock with PUK
yubico-piv-tool -a unblock-pin

# Enter your PUK when prompted
```

If PUK is also forgotten, contact IT - Yubikey needs factory reset.

### Wrong Slot

If you generated key in wrong slot:

```bash
# List what you have
age-plugin-yubikey --list

# Regenerate in slot 1 if needed
age-plugin-yubikey --generate --slot 1 --touch-policy always
```

## Security Best Practices

### ✅ DO

- **Keep your Yubikey PIN private** (don't share, don't write down)
- **Lock your computer** when away (Cmd+Ctrl+Q on Mac, Win+L on Windows)
- **Remove Yubikey** when not in use (especially in public)
- **Report lost Yubikey immediately** to team lead
- **Touch when LED blinks** (signals secret access)

### ❌ DON'T

- **Never share your Yubikey** with anyone
- **Never write down your PIN**
- **Never commit unencrypted secrets** to git
- **Never share secrets via Slack/email** (use SOPS)
- **Never ignore LED blink** (means secrets being accessed)

## Yubikey PIN Management

### Default PINs (Change These!)

- **User PIN**: 123456 (CHANGE IMMEDIATELY!)
- **PUK**: 12345678 (CHANGE IMMEDIATELY!)
- **Management Key**: 010203040506070801020304050607080102030405060708

### Changing PINs After Setup

```bash
# Change User PIN
yubico-piv-tool -a change-pin

# Change PUK
yubico-piv-tool -a change-puk
```

## Getting Help

- **Can't decrypt**: Check with team lead if you were added
- **Yubikey issues**: Reinsert, try different port, restart
- **Lost Yubikey**: Contact team lead immediately
- **Technical questions**: See [USAGE.md](USAGE.md) or [SETUP.md](SETUP.md)

## Quick Reference Card

Save this for daily use:

```bash
# View secrets (requires touch)
sops -d secrets.enc.yaml

# Edit secrets (requires 2 touches: open + save)
sops secrets.enc.yaml

# Check your Yubikey
age-plugin-yubikey --list

# Change PIN
yubico-piv-tool -a change-pin

# Unlock after wrong PIN
yubico-piv-tool -a unblock-pin
```

## What Happens When You Leave

When you leave the team:
1. Team lead removes your recipient from `.sops.yaml`
2. All secrets are re-encrypted (revokes your access)
3. You return company Yubikey (if applicable)
4. You delete local repository clone

Your access is automatically revoked - no manual key deletion needed.

## Common Questions

**Q: Can I use the same Yubikey for multiple projects?**
A: Yes! Use different PIV slots (82, 83, 84, etc.) for different projects.

**Q: What if I forget to touch?**
A: Command times out after 15 seconds. Just run it again and touch promptly.

**Q: Can I disable the touch requirement?**
A: No - this is a critical security feature. Touch ensures physical presence.

**Q: What if my Yubikey breaks?**
A: Team lead can use master key for recovery. You get new Yubikey and regenerate key.

**Q: Can I decrypt without my Yubikey?**
A: No (that's the point!). Only the master key (held by team lead) can decrypt without Yubikey.

**Q: Is my private key backed up?**
A: No - it's generated inside your Yubikey and never leaves. That's why we have a master key backup.

## Success Checklist

After setup, you should be able to:

- [ ] Run `age-plugin-yubikey --list` and see your key in slot 1
- [ ] Run `sops -d secrets.enc.yaml` and see decrypted secrets
- [ ] Yubikey LED blinks when decrypting
- [ ] You can touch Yubikey to complete decryption
- [ ] Your PIN is changed from default (123456)
- [ ] Your PUK is changed from default (12345678)

If all checked, you're ready to work with secrets! 🚀

---

## For Team Leads: Adding This User

```bash
# 1. Receive their yourname-yubikey.txt file

# 2. Copy their recipient (age1yubikey1...)
cat yourname-yubikey.txt

# 3. Add to .sops.yaml
age:
  - age1master...
  - age1yubikey1existing...
  - age1yubikey1new...  # Add here

# 4. Re-encrypt all files
find . -name "*.enc.*" -exec sops updatekeys {} \;

# 5. Commit and push
git add .sops.yaml **/*.enc.*
git commit -m "Add new team member"
git push

# 6. Notify team member they have access
```
