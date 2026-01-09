# Yubikey PIV Setup for SOPS

Complete guide for setting up SOPS with Yubikey PIV slots and master key backup.

## Overview

This setup provides maximum security by:
- **Physical touch requirement**: Every secret access requires touching your Yubikey
- **Hardware key storage**: Private keys never leave the Yubikey secure element
- **Master key backup**: Offline recovery key stored safely
- **Modern cryptography**: X25519 elliptic curve keys

## Architecture

```
┌─────────────────────────────────────────┐
│       Encrypted Secret File              │
└─────────────┬───────────────────────────┘
              │
              ├─── Decryption requires ───┐
              │                            │
    ┌─────────▼─────────┐      ┌─────────▼──────────┐
    │   Master Age Key   │      │  Yubikey PIV Slot  │
    │   (offline/safe)   │      │  + Physical Touch  │
    │                    │      │  + PIN             │
    └────────────────────┘      └────────────────────┘
```

## Prerequisites

### macOS

```bash
# Install via Homebrew
brew install sops age age-plugin-yubikey yubico-piv-tool

# Verify installations
sops --version
age --version
age-plugin-yubikey --version
yubico-piv-tool --version

# Verify Yubikey is detected
yubico-piv-tool -a status
```

### Linux (Debian/Ubuntu)

```bash
# Install SOPS
wget https://github.com/mozilla/sops/releases/download/v3.8.1/sops_3.8.1_amd64.deb
sudo dpkg -i sops_3.8.1_amd64.deb

# Install age
sudo apt install age

# Install Yubikey tools
sudo apt install yubikey-manager yubico-piv-tool

# Install Rust (required for age-plugin-yubikey)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Install age-plugin-yubikey
cargo install age-plugin-yubikey

# Verify installations
sops --version
age --version
age-plugin-yubikey --version
ykman --version

# Verify Yubikey is detected
ykman info
```

### Linux (Fedora/RHEL)

```bash
# Install SOPS
sudo dnf install sops

# Install age
sudo dnf install age

# Install Yubikey tools
sudo dnf install yubikey-manager yubico-piv-tool

# Install Rust and age-plugin-yubikey
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
cargo install age-plugin-yubikey
```

### Windows (WSL2 + Ubuntu)

```bash
# First, enable WSL2 and install Ubuntu from Microsoft Store
# Then follow Linux (Debian/Ubuntu) instructions above
```

### Windows (Native - PowerShell)

```powershell
# Install Chocolatey (if not installed)
# Visit: https://chocolatey.org/install

# Install via Chocolatey
choco install sops age

# Download and install Yubikey Manager from:
# https://www.yubico.com/support/download/yubikey-manager/

# Install Rust from: https://rustup.rs/
# Then install age-plugin-yubikey:
cargo install age-plugin-yubikey

# Verify Yubikey is detected
ykman info
```

## Step 1: Generate Master Key

The master key is for backup and CI/CD only. Store it offline.

```bash
# Generate master age key
age-keygen -o master-key.txt

# Display public key
grep "public key:" master-key.txt

# Example output:
# public key: age1u3hs6xmw09l50sjwzhngd098e23ysruhn5mus8lv2drr6evs9s8sn5ygnq
```

**Critical**: Move `master-key.txt` to secure offline storage immediately after setup.

## Step 2: Change Yubikey Default PINs

**IMPORTANT**: Change default PINs before generating keys!

```bash
# Change User PIN (default: 123456)
yubico-piv-tool -a change-pin

# Change PUK (PIN Unlock Key, default: 12345678)
yubico-piv-tool -a change-puk

# Optionally change Management Key (for advanced users)
yubico-piv-tool -a set-mgm-key
```

**Recommendations**:
- **User PIN**: 6-8 digits you'll remember
- **PUK**: 8+ digits, store in password manager
- **Management Key**: Only change if you understand implications

## Step 3: Generate PIV Key on Yubikey

```bash
# Generate key in PIV slot 1 with touch requirement
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --pin-policy always \
  --name "your-name-work"

# You'll be prompted for:
# 1. Management key (press Enter for default if you didn't change it)
# 2. Yubikey PIN

# Output shows your public recipient:
# Generated key in slot 1
# Public key: age1yubikey1qd...
```

### Slot Selection

| Slot | PIV Equivalent | Recommended Use |
|------|----------------|-----------------|
| 1 | Retired Key 1 (82) | **Primary daily work** (recommended) |
| 2 | Retired Key 2 (83) | Secondary/backup key |
| 3-20 | Retired Keys 3-20 (84-95) | Additional keys for different projects |

**Recommendation**: Use slot 1 for your daily work key.

### Touch Policy Options

```bash
# Always require touch (RECOMMENDED - maximum security)
--touch-policy always

# Touch cached for 15 seconds (convenience vs security trade-off)
--touch-policy cached

# Never require touch (NOT RECOMMENDED - defeats hardware security)
--touch-policy never
```

**Always use `--touch-policy always` for maximum security.**

### PIN Policy Options

```bash
# Always require PIN (RECOMMENDED)
--pin-policy always

# PIN once per session (less secure)
--pin-policy once
```

**Always use `--pin-policy always` for maximum security.**

## Step 4: Export Your Public Recipient

```bash
# Export your public recipient string
age-plugin-yubikey --identity --slot 1 > your-name-yubikey.txt

# View your recipient
cat your-name-yubikey.txt
# Output: age1yubikey1qd...
```

**Share this file** with your team lead to be added to `.sops.yaml`.

## Step 5: Create Identity File for SOPS

**Critical**: SOPS needs an identity file to know to use your Yubikey:

### macOS / Linux

```bash
# Create identity file that references your Yubikey
age-plugin-yubikey --identity > ~/.sops-age-keys.txt

# Add to shell profile (choose your shell):

# For zsh (macOS default, some Linux):
echo 'export SOPS_AGE_KEY_FILE=~/.sops-age-keys.txt' >> ~/.zshrc
source ~/.zshrc

# For bash (most Linux, older macOS):
echo 'export SOPS_AGE_KEY_FILE=~/.sops-age-keys.txt' >> ~/.bashrc
source ~/.bashrc

# For fish shell:
echo 'set -gx SOPS_AGE_KEY_FILE ~/.sops-age-keys.txt' >> ~/.config/fish/config.fish
source ~/.config/fish/config.fish
```

### Windows (PowerShell)

```powershell
# Create identity file
age-plugin-yubikey --identity | Out-File -FilePath "$env:USERPROFILE\.sops-age-keys.txt" -Encoding UTF8

# Set environment variable permanently
[System.Environment]::SetEnvironmentVariable('SOPS_AGE_KEY_FILE', "$env:USERPROFILE\.sops-age-keys.txt", 'User')

# Reload environment (or restart PowerShell)
$env:SOPS_AGE_KEY_FILE = "$env:USERPROFILE\.sops-age-keys.txt"
```

### Windows (WSL2)

```bash
# Same as Linux instructions above
age-plugin-yubikey --identity > ~/.sops-age-keys.txt
echo 'export SOPS_AGE_KEY_FILE=~/.sops-age-keys.txt' >> ~/.bashrc
source ~/.bashrc
```

**What this does**:
- Creates a file containing your Yubikey identity reference (not your private key!)
- Private key stays on Yubikey hardware
- SOPS uses this file to know "decrypt using the Yubikey plugin"

## Step 6: Verify Your Setup

```bash
# List PIV keys on your Yubikey
age-plugin-yubikey --list

# Expected output:
# Slot 82: age1yubikey1qd...
#   Name: your-name-work
#   Touch policy: Always
#   PIN policy: Always
```

Verify:
- ✅ Touch policy: Always
- ✅ PIN policy: Always
- ✅ Your recipient string is displayed

## Step 7: Create .sops.yaml Configuration

If you're the first team member:

```yaml
creation_rules:
  - path_regex: .*\.(yaml|json|env|ini|txt)$
    key_groups:
      - age:
          # Master key (offline backup, CI/CD)
          - age1u3hs6xmw09l50sjwzhngd098e23ysruhn5mus8lv2drr6evs9s8sn5ygnq
          # Team Yubikey PIV keys
          - age1yubikey1qd...  # Your Yubikey recipient (from step 4)
```

Replace `age1u3hs6xmw09l50sjwzhngd098e23ysruhn5mus8lv2drr6evs9s8sn5ygnq` with your master key public key.
Replace `age1yubikey1qd...` with your Yubikey recipient string.

## Step 8: Test Encryption and Decryption

```bash
# Create a test secret file
cat > test-secrets.yaml <<EOF
database:
  password: super_secret_123
api:
  key: sk_test_abc123
EOF

# Encrypt with SOPS (uses .sops.yaml config)
sops -e test-secrets.yaml > test-secrets.enc.yaml

# Test decryption with Yubikey (requires PIN + touch)
sops -d test-secrets.enc.yaml

# You'll be prompted for:
# 1. Yubikey PIN
# 2. Physical touch (LED will blink)

# If you see the decrypted content, setup is successful!
```

## Step 9: Test Master Key Access

```bash
# Test decryption with master key (without Yubikey)
SOPS_AGE_KEY_FILE=master-key.txt sops -d test-secrets.enc.yaml

# This should decrypt without requiring Yubikey
# Verifies master key works for emergency recovery
```

## Step 10: Secure Master Key

**Critical security step**:

```bash
# 1. Copy master-key.txt to secure offline storage:
#    - Physical safe
#    - Password manager (encrypted)
#    - Bank vault

# 2. Verify backup is readable

# 3. Remove from laptop
rm master-key.txt

# Or if keeping temporarily:
chmod 400 master-key.txt  # Read-only for owner
```

**Never commit `master-key.txt` to git!**

## Adding Additional Team Members

### For Team Members

```bash
# 1. Install tools (Prerequisites)
# 2. Change Yubikey PINs (Step 2)
# 3. Generate PIV key (Step 3)
age-plugin-yubikey --generate --slot 1 --touch-policy always --name "yourname-work"

# 4. Export recipient (Step 4)
age-plugin-yubikey --identity --slot 1 > yourname-yubikey.txt

# 5. Send yourname-yubikey.txt to team lead
```

### For Team Lead (Adding Member)

```bash
# 1. Receive member's public recipient (age1yubikey1...)

# 2. Add to .sops.yaml
age:
  - age1master...          # Master key
  - age1yubikey1qd...      # Existing member
  - age1yubikey1qw...      # NEW member

# 3. Re-encrypt all secrets to include new key
find . -name "*.enc.*" -exec sops updatekeys {} \;

# 4. Commit and push
git add .sops.yaml **/*.enc.*
git commit -m "Add new team member to SOPS access"
git push

# 5. Notify new member they have access
```

## Removing Team Members

```bash
# 1. Remove their recipient from .sops.yaml
age:
  - age1master...
  # - age1yubikey1old...  # REMOVED

# 2. Re-encrypt ALL secrets (revokes access)
find . -name "*.enc.*" -exec sops updatekeys {} \;

# 3. Commit changes
git add .sops.yaml **/*.enc.*
git commit -m "Remove departed team member"
git push
```

**This immediately revokes their access to all secrets.**

## CI/CD Integration

CI/CD systems can't provide physical touch, so use the master key:

### GitHub Actions Example

```yaml
name: Deploy
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install SOPS
        run: |
          wget https://github.com/mozilla/sops/releases/download/v3.8.1/sops_3.8.1_amd64.deb
          sudo dpkg -i sops_3.8.1_amd64.deb

      - name: Decrypt secrets
        env:
          SOPS_AGE_KEY: ${{ secrets.SOPS_MASTER_KEY }}
        run: |
          sops -d config.enc.yaml > config.yaml

      - name: Deploy
        run: ./deploy.sh
```

**Add master key to GitHub Secrets**:
1. Copy contents of `master-key.txt` (the private key line starting with `AGE-SECRET-KEY-`)
2. GitHub Repo → Settings → Secrets → New repository secret
3. Name: `SOPS_MASTER_KEY`
4. Value: Paste the private key

## Advanced: Multiple Keys for Different Environments

Use different PIV slots for different security levels:

```bash
# Generate development key (slot 1)
age-plugin-yubikey --generate --slot 1 --name "dev-secrets"

# Generate production key (slot 2)
age-plugin-yubikey --generate --slot 2 --name "prod-secrets"
```

Then configure `.sops.yaml`:

```yaml
creation_rules:
  # Development secrets (all team has access)
  - path_regex: dev/.*\.yaml$
    key_groups:
      - age:
          - age1master...
          - age1yubikey1slot82...  # All team members

  # Production secrets (limited access)
  - path_regex: prod/.*\.yaml$
    key_groups:
      - age:
          - age1master...
          - age1yubikey1slot83senior1...  # Senior team only
          - age1yubikey1slot83senior2...
```

## Security Verification Checklist

After setup, verify:

- [ ] Touch policy is "Always" (`age-plugin-yubikey --list`)
- [ ] PIN policy is "Always"
- [ ] Yubikey PIN changed from default (123456)
- [ ] PUK changed from default (12345678)
- [ ] Decryption requires physical touch (LED blinks)
- [ ] Master key stored offline (not on laptop)
- [ ] Master key works for emergency access
- [ ] `.gitignore` prevents committing master key
- [ ] Test encryption/decryption works

## Troubleshooting

### "No identities found"

**Cause**: No PIV key in Yubikey or wrong slot

**Solution**:
```bash
# List what's in your Yubikey
age-plugin-yubikey --list

# Generate key if missing
age-plugin-yubikey --generate --slot 1 --touch-policy always
```

### "Touch timeout"

**Cause**: Didn't touch Yubikey within 15 seconds

**Solution**: Run command again, touch promptly when LED blinks

### "PIN verification failed"

**Cause**: Wrong PIN entered

**Solution**:
```bash
# Verify PIN manually
yubico-piv-tool -a verify-pin

# If locked (3 failed attempts), unlock with PUK
yubico-piv-tool -a unblock-pin
```

### "Slot already occupied"

**Cause**: Key already exists in that slot

**Solution**:
```bash
# Use a different slot (2, 3, etc.)
age-plugin-yubikey --generate --slot 2

# Or overwrite (destroys existing key!)
age-plugin-yubikey --generate --slot 1 --overwrite
```

### Multiple Yubikeys Detected

If you have multiple Yubikeys:

```bash
# List all Yubikeys with serial numbers
age-plugin-yubikey --list

# Specify which Yubikey to use
age-plugin-yubikey --serial 12345678 --identity --slot 1
```

## Security Best Practices

### ✅ DO

- Use `--touch-policy always` (maximum security)
- Use `--pin-policy always`
- Change default PINs immediately
- Store master key offline only
- Test recovery procedures
- Re-encrypt when removing team members
- Use slot 1 for consistency
- Keep Yubikey firmware updated

### ❌ DON'T

- Never use `--touch-policy never` (defeats security)
- Never commit master key to git
- Never share your Yubikey or PIN
- Never store master key on laptop permanently
- Never ignore LED blink (signals secret access)
- Never skip PIN changes from defaults

## Recovery Scenarios

### Lost Yubikey

1. Use master key to decrypt:
   ```bash
   SOPS_AGE_KEY_FILE=/path/to/master-key.txt sops -d secrets.enc.yaml
   ```

2. Get new Yubikey and generate new key

3. Update `.sops.yaml` with new recipient

4. Re-encrypt all secrets

### Forgotten PIN

```bash
# Unlock with PUK (hope you stored it!)
yubico-piv-tool -a unblock-pin

# If PUK also forgotten: Yubikey must be factory reset (all keys lost)
yubico-piv-tool -a reset
# Then regenerate keys
```

### Master Key Lost

**Critical**: If master key is lost and all Yubikeys fail:
- Secrets become unrecoverable
- Must regenerate all secrets manually
- **This is why secure master key backup is essential**

## Hardware Attestation

Verify your key was generated on genuine Yubikey hardware:

```bash
# Attest key in slot 1
age-plugin-yubikey --attest --slot 1

# Verifies cryptographic proof of hardware generation
```

This proves to others that your private key was generated inside the Yubikey and never exposed.

## Resources

- age-plugin-yubikey: https://github.com/str4d/age-plugin-yubikey
- SOPS: https://github.com/mozilla/sops
- Yubikey PIV Guide: https://developers.yubico.com/PIV/
- Age encryption: https://github.com/FiloSottile/age

## Next Steps

After setup:
1. Read [USAGE.md](USAGE.md) for daily workflows
2. Share [ONBOARDING.md](ONBOARDING.md) with new team members
3. Set up CI/CD integration
4. Test emergency recovery procedures
5. Document your team's key management policies
