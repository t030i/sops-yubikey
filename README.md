# SOPS + Yubikey PIV Setup

Secure secret management using [SOPS](https://github.com/mozilla/sops) with Yubikey PIV hardware keys and offline master key backup.

## Features

- 🔐 **Physical Touch Required** - Every decryption requires touching your Yubikey (prevents silent malware attacks)
- 🔑 **Hardware-Backed Keys** - Private keys never leave the Yubikey secure element
- 💾 **Offline Master Key** - Emergency backup stored safely offline
- 👥 **Team-Friendly** - Easy onboarding with individual Yubikeys per team member
- 🔒 **Modern Crypto** - X25519 elliptic curve cryptography via age-plugin-yubikey

## Quick Start

### Prerequisites

```bash
brew install sops age age-plugin-yubikey yubico-piv-tool
```

### Setup (5 minutes)

```bash
# 1. Generate master key (backup offline!)
age-keygen -o master-key.txt

# 2. Change Yubikey PINs (default: 123456 / 12345678)
yubico-piv-tool -a change-pin
yubico-piv-tool -a change-puk

# 3. Generate Yubikey PIV key with touch policy
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --pin-policy always \
  --name "your-name"

# 4. Create identity file for SOPS
age-plugin-yubikey --identity > ~/.sops-age-keys.txt
echo 'export SOPS_AGE_KEY_FILE=~/.sops-age-keys.txt' >> ~/.zshrc
source ~/.zshrc

# 5. Configure SOPS (see .sops.yaml example in repo)

# 6. Test it!
echo "secret: password123" > test.yaml
sops -e test.yaml > test.enc.yaml
sops -d test.enc.yaml  # Touch Yubikey when LED blinks ✨
```

### Daily Usage

```bash
# Edit secrets (touch Yubikey twice: open + save)
sops secrets.enc.yaml

# Decrypt to view
sops -d secrets.enc.yaml

# Encrypt new file
sops -e secrets.yaml > secrets.enc.yaml
```

## Documentation

- **[SETUP.md](SETUP.md)** - Complete setup guide with all steps
- **[USAGE.md](USAGE.md)** - Daily workflows and team management
- **[ONBOARDING.md](ONBOARDING.md)** - Guide for new team members
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** - Common issues and solutions
- **[PRE-COMMIT-CHECKLIST.md](PRE-COMMIT-CHECKLIST.md)** - Verify before committing

## Why Yubikey PIV?

Traditional secret management often relies on keys stored on disk. With Yubikey PIV:

| Security Feature | Traditional | Yubikey PIV |
|-----------------|-------------|-------------|
| Physical touch required | ❌ | ✅ |
| Malware can silently decrypt | ✅ | ❌ |
| User awareness of access | ❌ | ✅ (LED blinks) |
| Private key extractable | ✅ | ❌ |
| Hardware-backed crypto | ❌ | ✅ |

**Example**: If malware tries to decrypt secrets, your Yubikey LED blinks - you'll notice and can prevent the attack by not touching.

## Architecture

```
┌─────────────────────────────────────┐
│      Encrypted Secret File          │
└──────────────┬──────────────────────┘
               │
               ├─── Decryption ───┐
               │                   │
    ┌──────────▼────────┐    ┌────▼─────────────┐
    │   Master Age Key   │    │  Yubikey PIV     │
    │   (offline/safe)   │    │  + Touch + PIN   │
    └────────────────────┘    └──────────────────┘
```

## Team Workflow

### Adding Team Members

1. Team member generates Yubikey PIV key
2. Shares public recipient (`age1yubikey1...`)
3. Team lead adds to `.sops.yaml`
4. Re-encrypt all secrets: `sops updatekeys secrets.enc.yaml`
5. Team member can now decrypt (with Yubikey touch)

### Removing Team Members

1. Remove their recipient from `.sops.yaml`
2. Re-encrypt all secrets (revokes their access)
3. Commit changes

## CI/CD Integration

CI/CD can't provide physical touch, so use the master key:

```yaml
# GitHub Actions example
- name: Decrypt secrets
  env:
    SOPS_AGE_KEY: ${{ secrets.SOPS_MASTER_KEY }}
  run: sops -d secrets.enc.yaml > secrets.yaml
```

## Security Best Practices

✅ **DO**:
- Store master key offline (safe, password manager)
- Use `--touch-policy always` for all Yubikeys
- Change default PINs immediately
- Re-encrypt when removing team members
- Commit encrypted files to git

❌ **DON'T**:
- Never commit master-key.txt
- Never commit unencrypted secrets
- Never disable touch policy
- Never share Yubikey or PIN
- Never ignore LED blink (signals secret access)

## Example .sops.yaml

```yaml
creation_rules:
  - path_regex: .*\.(yaml|json|env|ini|txt)$
    key_groups:
      - age:
          # Master key (offline backup, CI/CD)
          - age1u3hs6xmw09l50sjwzhngd098e23ysruhn5mus8lv2drr6evs9s8sn5ygnq
          # Team Yubikeys (daily use with touch)
          - age1yubikey1qd...  # Team member 1
          - age1yubikey1qw...  # Team member 2
```

## Requirements

- **Yubikey 5** or newer (with PIV support)
- **macOS, Linux, or WSL** on Windows
- **SOPS 3.8+**
- **age 1.1+**
- **age-plugin-yubikey 0.5+**

## Troubleshooting

**"No identities found"**
```bash
# Create identity file
age-plugin-yubikey --identity > ~/.sops-age-keys.txt
export SOPS_AGE_KEY_FILE=~/.sops-age-keys.txt
```

**"Touch timeout"**
- Touch Yubikey more quickly when LED blinks (15 second timeout)

**"Algorithm error"**
```bash
# Try without pin-policy flag
age-plugin-yubikey --generate --slot 1 --touch-policy always --name "yourname"
```

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for more solutions.

## Contributing

Contributions welcome! Feel free to:
- Report issues
- Suggest documentation improvements
- Share your team's workflow adaptations

## Resources

- [SOPS](https://github.com/mozilla/sops) - Secrets OPerationS
- [age](https://github.com/FiloSottile/age) - Simple, modern file encryption
- [age-plugin-yubikey](https://github.com/str4d/age-plugin-yubikey) - Age plugin for Yubikey PIV
- [Yubikey PIV](https://developers.yubico.com/PIV/) - Yubikey PIV documentation

## License

This documentation is provided as-is for educational and practical use. Adapt it to your team's needs.

## Support

- 📖 Read the [complete setup guide](SETUP.md)
- 🐛 [Open an issue](https://github.com/shamil2/sops-yubikey/issues) for bugs or questions
- 💡 Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues

---

**Security Note**: This setup provides strong security through hardware-backed keys and physical touch requirements. However, no system is perfect - always follow your organization's security policies and best practices.
