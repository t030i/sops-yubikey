# Pre-Commit Checklist

Before committing this SOPS setup to your repository, verify everything is in order:

## ✅ Security Checks

### Files That Should NOT Be Committed

Run these checks to ensure sensitive files are excluded:

```bash
# Check that master key is NOT present
ls master-key.txt 2>/dev/null && echo "❌ DANGER: master-key.txt found!" || echo "✅ master-key.txt not present"

# Check that identity file is NOT in repo (should be in ~/)
ls .sops-age-keys.txt 2>/dev/null && echo "❌ WARNING: identity file in repo!" || echo "✅ identity file not in repo"

# Check for unencrypted secrets
ls secrets.yaml config.yaml .env 2>/dev/null && echo "❌ WARNING: Unencrypted secrets found!" || echo "✅ No unencrypted secrets"

# Check for test files
ls test*.yaml 2>/dev/null && echo "⚠️  Test files found (should be in .gitignore)" || echo "✅ No test files"
```

### Files That SHOULD Be Committed

```bash
# Verify required files exist
ls .sops.yaml README.md SETUP.md USAGE.md ONBOARDING.md .gitignore >/dev/null 2>&1 && echo "✅ Core files present" || echo "❌ Missing core files"

# Verify example file exists
ls secrets.example.yaml >/dev/null 2>&1 && echo "✅ Example file present" || echo "❌ Missing example file"
```

## ✅ Configuration Checks

### Verify .sops.yaml

```bash
# Check .sops.yaml has your keys
cat .sops.yaml
```

**Verify**:
- [ ] Master key (age1...) is present
- [ ] Your Yubikey recipient (age1yubikey1...) is present
- [ ] No PGP/GPG keys (we're PIV-only)
- [ ] Path regex matches your file patterns

### Verify .gitignore

```bash
# Check .gitignore excludes sensitive files
cat .gitignore | grep -E "master-key|secrets.yaml|.env|test"
```

**Verify**:
- [ ] master-key.txt is excluded
- [ ] Unencrypted secrets are excluded
- [ ] Test files are excluded
- [ ] Encrypted files (*.enc.*) are allowed

## ✅ Documentation Checks

### Verify All Docs Are Complete

```bash
# Check documentation files
for doc in README.md SETUP.md USAGE.md ONBOARDING.md; do
  echo "Checking $doc..."
  grep -q "identity" $doc && echo "  ✅ Mentions identity file" || echo "  ❌ Missing identity file instructions"
  grep -q "slot 1" $doc && echo "  ✅ Uses correct slot numbers" || echo "  ⚠️  Check slot numbers"
done
```

**Verify each doc mentions**:
- [ ] README.md: Identity file creation
- [ ] SETUP.md: Step 5 creates identity file
- [ ] USAGE.md: Team member setup includes identity file
- [ ] ONBOARDING.md: Step 5 creates identity file

## ✅ Functional Tests

### Test Your Setup Works

```bash
# 1. Verify identity file exists (in home directory)
ls ~/.sops-age-keys.txt && echo "✅ Identity file exists" || echo "❌ Identity file missing"

# 2. Verify environment variable is set
echo $SOPS_AGE_KEY_FILE | grep -q ".sops-age-keys.txt" && echo "✅ SOPS_AGE_KEY_FILE set" || echo "❌ Environment variable not set"

# 3. Create test file to verify encryption works
cat > /tmp/test-sops.yaml <<EOF
test: value
EOF

# 4. Encrypt test file
sops -e /tmp/test-sops.yaml > /tmp/test-sops.enc.yaml && echo "✅ Encryption works" || echo "❌ Encryption failed"

# 5. Decrypt with Yubikey (will require touch)
sops -d /tmp/test-sops.enc.yaml >/dev/null && echo "✅ Decryption with Yubikey works" || echo "❌ Decryption failed"

# 6. Cleanup
rm /tmp/test-sops*.yaml
```

## ✅ Git Preparation

### Initialize Repository (If Not Already)

```bash
# Check if git repo exists
if [ ! -d .git ]; then
  echo "Initializing git repository..."
  git init
  echo "✅ Git initialized"
else
  echo "✅ Git already initialized"
fi
```

### Stage Files for Commit

```bash
# Add all documentation and config
git add README.md SETUP.md USAGE.md ONBOARDING.md TROUBLESHOOTING.md PRE-COMMIT-CHECKLIST.md

# Add configuration
git add .sops.yaml .gitignore

# Add example file
git add secrets.example.yaml

# Check what will be committed
git status
```

### Verify Nothing Sensitive Is Staged

```bash
# Double check staged files
echo "Files staged for commit:"
git diff --cached --name-only

# Verify master key is NOT staged
git diff --cached --name-only | grep -q "master-key.txt" && echo "❌ DANGER: master-key.txt is staged!" || echo "✅ master-key.txt not staged"

# Verify no unencrypted secrets are staged
git diff --cached --name-only | grep -qE "^secrets\.yaml$|^config\.yaml$|^\.env$" && echo "❌ DANGER: Unencrypted secrets staged!" || echo "✅ No unencrypted secrets staged"
```

## ✅ Final Checks

### Run Complete Verification

```bash
# All checks in one command
echo "=== SECURITY CHECKS ==="
! ls master-key.txt 2>/dev/null && echo "✅ master-key.txt not in repo" || echo "❌ REMOVE master-key.txt!"
! ls .sops-age-keys.txt 2>/dev/null && echo "✅ identity file not in repo" || echo "❌ REMOVE .sops-age-keys.txt!"
! ls secrets.yaml 2>/dev/null && echo "✅ No unencrypted secrets.yaml" || echo "❌ REMOVE secrets.yaml!"

echo ""
echo "=== CONFIGURATION CHECKS ==="
[ -f .sops.yaml ] && echo "✅ .sops.yaml exists" || echo "❌ .sops.yaml missing"
[ -f .gitignore ] && echo "✅ .gitignore exists" || echo "❌ .gitignore missing"

echo ""
echo "=== DOCUMENTATION CHECKS ==="
for doc in README.md SETUP.md USAGE.md ONBOARDING.md; do
  [ -f "$doc" ] && echo "✅ $doc exists" || echo "❌ $doc missing"
done

echo ""
echo "=== GIT CHECKS ==="
git diff --cached --name-only | grep -qE "master-key|secrets\.yaml|\.env$" && echo "❌ SENSITIVE FILES STAGED!" || echo "✅ No sensitive files staged"
```

## 🚀 Ready to Commit

If all checks pass:

```bash
# Commit with descriptive message
git commit -m "Initial SOPS setup with Yubikey PIV

- Yubikey PIV-based encryption with physical touch requirement
- Master key backup for emergency recovery
- Complete documentation (README, SETUP, USAGE, ONBOARDING)
- Secure .gitignore configuration
- Example secrets template

Security: All sensitive keys excluded from repository"

# View commit
git log -1 --stat
```

## 📝 Post-Commit

After committing:

1. **Secure your master key**:
   ```bash
   # Master key should already be deleted from repo
   # Verify it's in safe storage (password manager, physical safe, etc.)
   ```

2. **Verify identity file**:
   ```bash
   # Should be in home directory, not in repo
   ls -la ~/.sops-age-keys.txt
   ```

3. **Test clone works** (optional):
   ```bash
   # In another directory
   git clone <your-repo> /tmp/test-clone
   cd /tmp/test-clone
   # Verify no master-key.txt exists
   # Verify documentation is complete
   ```

4. **Share with team**:
   - Point team members to ONBOARDING.md
   - Ensure they have Yubikeys
   - Be ready to add their recipients to .sops.yaml

## ⚠️ If Checks Fail

### Master Key Found in Repo

```bash
# Remove it immediately
rm master-key.txt
git rm master-key.txt 2>/dev/null  # If already tracked

# Move to safe storage
# DO NOT commit master key!
```

### Unencrypted Secrets Found

```bash
# Remove unencrypted secrets
rm secrets.yaml config.yaml .env

# Encrypt them first
sops -e secrets.yaml > secrets.enc.yaml

# Then commit only encrypted versions
```

### Identity File in Repo

```bash
# Remove from repo
rm .sops-age-keys.txt

# It should be in home directory only
ls ~/.sops-age-keys.txt
```

## 📚 What Gets Committed

**Safe to commit**:
- ✅ .sops.yaml (contains only public keys)
- ✅ .gitignore
- ✅ README.md, SETUP.md, USAGE.md, ONBOARDING.md
- ✅ secrets.example.yaml (example template)
- ✅ *.enc.yaml (encrypted secrets)

**NEVER commit**:
- ❌ master-key.txt (private master key)
- ❌ .sops-age-keys.txt (identity file)
- ❌ secrets.yaml (unencrypted secrets)
- ❌ test-*.yaml (test files)

## 🎯 Summary

Before `git commit`, ensure:
- [ ] No master-key.txt in repo
- [ ] No .sops-age-keys.txt in repo
- [ ] No unencrypted secrets in repo
- [ ] .sops.yaml has your keys
- [ ] .gitignore excludes sensitive files
- [ ] All documentation is complete
- [ ] Test encryption/decryption works
- [ ] Git status shows only safe files

✅ **If all checked, you're ready to commit!**
