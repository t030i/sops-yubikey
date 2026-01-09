# age-plugin-yubikey Troubleshooting

## "algorithm error" When Generating Keys

This error typically occurs due to Yubikey initialization or compatibility issues.

### Solution 1: Check Yubikey Status

```bash
# Check Yubikey is detected and PIV applet is working
yubico-piv-tool -a status

# Check firmware version (need 5.2.3+ for best compatibility)
ykman info
```

### Solution 2: Try Without PIN Policy

Some Yubikey firmware versions have issues with `--pin-policy always`:

```bash
# Try without pin-policy flag
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --name "shamil"
```

### Solution 3: Specify Management Key

```bash
# Use default management key explicitly
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --management-key 010203040506070801020304050607080102030405060708 \
  --name "shamil"
```

### Solution 4: Reset PIV Applet

**WARNING: This erases ALL PIV keys on the Yubikey!**

```bash
# Reset PIV applet to factory defaults
yubico-piv-tool -a reset

# Then try generating key again
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --name "shamil"
```

### Solution 5: Try Different Slot

Slot 2 might have an existing key or issue:

```bash
# Try slot 1 instead
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --name "shamil"
```

### Solution 6: Update Firmware

If you have an older Yubikey:

```bash
# Check current firmware
ykman info

# Yubikey 5.2.3+ recommended for age-plugin-yubikey
# Update via: https://www.yubico.com/support/download/yubikey-manager/
```

### Solution 7: List Existing Keys

Check if there's already a key blocking the slot:

```bash
# List all PIV keys
age-plugin-yubikey --list

# If slot is occupied, either use different slot or overwrite:
age-plugin-yubikey --generate \
  --slot 1 \
  --touch-policy always \
  --overwrite \
  --name "shamil"
```

## Alternative: Use Simple age Key Instead of PIV

If Yubikey PIV continues to have issues, you can use a regular age key:

```bash
# Generate a regular age key (stored on disk, not on Yubikey)
age-keygen -o shamil-key.txt

# Use in .sops.yaml
age:
  - age1master...
  - age1shamil...  # From shamil-key.txt public key
```

**Trade-off**: No physical touch requirement, but still secure with PIN-protected Yubikey storage option.

## Getting More Information

```bash
# Enable verbose output
age-plugin-yubikey --generate --slot 1 --touch-policy always --name "shamil" -vv

# Check YubiKey Manager
ykman piv info

# Check PIV certificates
yubico-piv-tool -a status
```
