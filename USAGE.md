# SOPS Usage Guide

Daily workflows and common operations for SOPS secret management with Yubikey PIV.

## Daily Operations

### Viewing Secrets

```bash
# Decrypt to stdout (requires Yubikey touch)
sops -d secrets.enc.yaml

# Decrypt to file
sops -d secrets.enc.yaml > secrets.yaml

# Decrypt and pipe to grep
sops -d secrets.enc.yaml | grep database
```

### Editing Secrets

The easiest way to update secrets:

```bash
# Opens decrypted content in your $EDITOR
# Automatically re-encrypts when you save and exit
sops secrets.enc.yaml
```

Your Yubikey LED will blink twice:
1. Once when opening (to decrypt)
2. Once when saving (to re-encrypt)

Touch your Yubikey both times.

### Creating New Encrypted Files

```bash
# Method 1: Encrypt existing file
sops -e secrets.yaml > secrets.enc.yaml

# Method 2: Create and edit in one step
sops secrets.enc.yaml
# (If file doesn't exist, SOPS creates it)
```

### File Naming Conventions

**Recommended patterns**:
- `secrets.enc.yaml` - Encrypted secrets
- `config.enc.json` - Encrypted configuration
- `.env.enc` - Encrypted environment variables

**Add to `.gitignore`**:
```gitignore
# Never commit unencrypted secrets
secrets.yaml
config.json
.env

# DO commit encrypted files
!*.enc.yaml
!*.enc.json
!*.enc.env
```

## Team Management

### Adding Team Members

#### For New Team Member

```bash
# 1. Install tools
brew install sops age age-plugin-yubikey

# 2. Generate Yubikey PIV key
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --name "yourname-work"

# 3. Export public recipient
age-plugin-yubikey --identity --slot 1 > yourname.txt

# 4. Create identity file
age-plugin-yubikey --identity > ~/.sops-age-keys.txt
echo 'export SOPS_AGE_KEY_FILE=~/.sops-age-keys.txt' >> ~/.zshrc
source ~/.zshrc

# 5. Send yourname.txt to team lead
```

#### For Team Lead

```bash
# 1. Receive new member's recipient (age1yubikey1...)

# 2. Add to .sops.yaml
age:
  - age1master...
  - age1yubikey1existing...
  - age1yubikey1new...  # Add here

# 3. Re-encrypt all secrets
find . -name "*.enc.*" -exec sops updatekeys {} \;

# 4. Commit and push
git add .sops.yaml **/*.enc.*
git commit -m "Add team member"
git push
```

### Removing Team Members

```bash
# 1. Remove from .sops.yaml
age:
  - age1master...
  # Removed: age1yubikey1departed...

# 2. Re-encrypt ALL secrets (revokes access)
find . -name "*.enc.*" -exec sops updatekeys {} \;

# 3. Commit
git add .sops.yaml **/*.enc.*
git commit -m "Remove team member"
git push
```

## Emergency Procedures

### Lost Yubikey

Use the master key:

```bash
# Set master key environment variable
export SOPS_AGE_KEY_FILE=/path/to/master-key.txt

# Decrypt as normal
sops -d secrets.enc.yaml

# Or inline
SOPS_AGE_KEY_FILE=/path/to/master-key.txt sops -d secrets.enc.yaml
```

Then:
1. Get new Yubikey
2. Generate new PIV key
3. Update `.sops.yaml`
4. Re-encrypt secrets

### Forgotten PIN

```bash
# Unlock with PUK
yubico-piv-tool -a unblock-pin
```

If PUK is also forgotten, Yubikey must be factory reset (all keys lost).

## CI/CD Integration

CI/CD can't provide physical touch, so use master key:

### GitHub Actions

```yaml
- name: Decrypt secrets
  env:
    SOPS_AGE_KEY: ${{ secrets.SOPS_MASTER_KEY }}
  run: |
    sops -d secrets.enc.yaml > secrets.yaml
```

Store the master key private key (starts with `AGE-SECRET-KEY-`) in GitHub Secrets.

### GitLab CI

```yaml
decrypt_secrets:
  script:
    - echo "$SOPS_MASTER_KEY" > /tmp/key.txt
    - SOPS_AGE_KEY_FILE=/tmp/key.txt sops -d secrets.enc.yaml > secrets.yaml
  variables:
    SOPS_MASTER_KEY:
      value: $SOPS_MASTER_KEY
      masked: true
```

## Advanced Workflows

### Rotating All Keys

When team changes significantly:

```bash
# 1. Update .sops.yaml with new recipients

# 2. Rotate all files
find . -name "*.enc.*" -exec sops updatekeys {} \;

# 3. Verify
sops -d secrets.enc.yaml
```

### Different Keys for Different Environments

```yaml
creation_rules:
  # Development (all team)
  - path_regex: dev/.*\.yaml$
    key_groups:
      - age:
          - age1master...
          - age1yubikey1all_team...

  # Production (limited access)
  - path_regex: prod/.*\.yaml$
    key_groups:
      - age:
          - age1master...
          - age1yubikey1senior_only...
```

### Partial Encryption

Encrypt only specific fields:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*\.yaml$
    encrypted_regex: '^(password|secret|key|token)$'
    key_groups:
      - age: [...]
```

Only fields matching the regex will be encrypted.

### Exec Mode

Run commands with decrypted secrets as environment variables:

```bash
# Export decrypted secrets as env vars
sops exec-env secrets.enc.env 'npm run start'

# Or with file replacement
sops exec-file secrets.enc.yaml './script.sh {}'
```

## Troubleshooting

### "No identities found"

**Cause**: Yubikey PIV key not detected

**Solution**:
```bash
# Check Yubikey
age-plugin-yubikey --list

# If empty, generate key
age-plugin-yubikey --generate --slot 1 --touch-policy always
```

### "Touch timeout"

**Cause**: Didn't touch Yubikey within 15 seconds

**Solution**: Run command again, touch when LED blinks

### "MAC mismatch" Error

**Cause**: File was manually edited or corrupted

**Solution**: Restore from git history or re-encrypt from unencrypted source

### File Won't Decrypt After Key Rotation

**Cause**: File wasn't re-encrypted after `.sops.yaml` changes

**Solution**:
```bash
sops updatekeys file.enc.yaml
```

### Multiple Yubikeys Detected

```bash
# List all with serial numbers
age-plugin-yubikey --list

# Specify which to use
age-plugin-yubikey --serial 12345678 --identity --slot 1
```

## Security Best Practices

### ✅ DO

- Always use `--touch-policy always` for new keys
- Commit encrypted files to git
- Store master key offline only
- Remove team members promptly when they leave
- Re-encrypt secrets after removing members
- Test emergency recovery procedures

### ❌ DON'T

- Never commit unencrypted secrets
- Never commit `master-key.txt`
- Never share Yubikey or PIN
- Never disable touch policy
- Never store master key on laptop permanently
- Never ignore LED blink (signals secret access)

## Common Workflows

### Starting Work on New Machine

```bash
# 1. Clone repository
git clone <repo>

# 2. Install tools
brew install sops age age-plugin-yubikey

# 3. Connect Yubikey

# 4. Verify access
age-plugin-yubikey --list

# 5. Decrypt and work
sops secrets.enc.yaml
```

### Sharing Secrets with New Service

```bash
# 1. Edit encrypted file
sops secrets.enc.yaml

# 2. Add new secrets
# 3. Save and exit (auto-re-encrypts)

# 4. Commit and push
git add secrets.enc.yaml
git commit -m "Add new service credentials"
git push
```

### Working with Multiple Projects

If you have different Yubikey keys for different projects:

```bash
# List all your PIV keys
age-plugin-yubikey --list

# Slot 82: work-project-a
# Slot 83: work-project-b

# SOPS automatically uses the right key based on .sops.yaml
```

## Quick Reference

```bash
# View secrets
sops -d secrets.enc.yaml

# Edit secrets
sops secrets.enc.yaml

# Encrypt file
sops -e secrets.yaml > secrets.enc.yaml

# Re-encrypt after key changes
sops updatekeys secrets.enc.yaml

# Use master key
SOPS_AGE_KEY_FILE=master-key.txt sops -d secrets.enc.yaml

# Check Yubikey
age-plugin-yubikey --list

# Change PIN
yubico-piv-tool -a change-pin

# Unlock with PUK
yubico-piv-tool -a unblock-pin
```

## Additional Resources

- SOPS Documentation: https://github.com/mozilla/sops
- age-plugin-yubikey: https://github.com/str4d/age-plugin-yubikey
- Yubikey PIV: https://developers.yubico.com/PIV/
