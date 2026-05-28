# Private Git Repository Configuration

Configure ArgoCD to access your **private** Git repository.

## Table of Contents
- [When to Use This](#when-to-use-this)
- [Quick Check: Is Your Repository Private?](#quick-check-is-your-repository-private)
- [Method 1: HTTPS with Personal Access Token (Recommended)](#method-1-https-with-personal-access-token-recommended)
- [Method 2: SSH Keys (Alternative)](#method-2-ssh-keys-alternative)
- [Security Best Practices](#security-best-practices)
- [Troubleshooting](#troubleshooting)
- [Removing Configuration](#removing-configuration)
- [Related Documentation](#related-documentation)

## When to Use This

**Only needed if your forked repository is private.**

If your repository is public, ArgoCD can access it without credentials - skip this step entirely.

## Quick Check: Is Your Repository Private?

```bash
# Try to access your repository without authentication
curl -I https://github.com/YOUR_USERNAME/fab-rig

# If you see "HTTP/2 200" → Public (no configuration needed)
# If you see "HTTP/2 404" → Private (configure credentials below)
```

---

## Method 1: HTTPS with Personal Access Token (Recommended)

### Step 1: Create GitHub Personal Access Token

1. **Open GitHub Token Settings**
   - Visit: https://github.com/settings/tokens/new
   - Or navigate: GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token

2. **Configure the Token**
   - **Note (Name):** Enter `ArgoCD fab-rig access`
   - **Expiration:** Select duration
     - `90 days` - Recommended (good security, needs rotation)
     - `1 year` - Less maintenance
     - `No expiration` - Most convenient (less secure)
   - **Select scopes:**
     - ✅ Check **`repo`** (Full control of private repositories)
       - This automatically checks all sub-items under repo
     - Leave all other scopes unchecked

3. **Generate and Save Token**
   - Click green **"Generate token"** button at bottom
   - **IMPORTANT:** Copy the token immediately
     - Format: `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
     - You won't be able to see it again!
   - Save it temporarily in a secure location (password manager or secure note)

### Step 2: Create Repository Secret File

1. **Copy the example template:**
   ```bash
   cp manifests/01-private-repo-config/github-repo-secret.example.yaml \
      manifests/01-private-repo-config/github-repo-secret.yaml
   ```

2. **Edit the file:**
   ```bash
   vi manifests/01-private-repo-config/github-repo-secret.yaml
   ```

3. **Replace placeholders with your values:**

   **Before (template):**
   ```yaml
   stringData:
     type: git
     url: https://github.com/YOUR_GITHUB_USERNAME/fab-rig.git
     username: YOUR_GITHUB_USERNAME
     password: YOUR_GITHUB_TOKEN
   ```

   **After (your values):**
   ```yaml
   stringData:
     type: git
     url: https://github.com/srikantt/Infrabric-deployer.git
     username: bbenshab
     password: ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

   - Replace `YOUR_GITHUB_USERNAME` with your GitHub username (appears **twice**: in url and username)
   - Replace `YOUR_GITHUB_TOKEN` with the token you copied in Step 1

4. **Save and close** the file (`:wq` in vi, or `Ctrl+O` then `Ctrl+X` in nano)

### Step 3: Apply the Secret to OpenShift

```bash
# Apply the secret to the cluster
oc apply -f manifests/01-private-repo-config/github-repo-secret.yaml
```

**Expected output:**
```
secret/fab-rig-repo created
```

### Step 4: Restart ArgoCD Repository Server

ArgoCD needs to reload to pick up the new credentials:

```bash
# Restart the repo server pod
oc delete pod -n openshift-gitops -l app.kubernetes.io/name=openshift-gitops-repo-server

# Wait for it to be ready (~30 seconds)
oc wait --for=condition=ready pod -l app.kubernetes.io/name=openshift-gitops-repo-server -n openshift-gitops --timeout=60s
```

**Expected output:**
```
pod "openshift-gitops-repo-server-xxxxxxxxx-xxxxx" deleted
pod/openshift-gitops-repo-server-xxxxxxxxx-xxxxx condition met
```

### Step 5: Verify Configuration

1. **Check the secret was created:**
   ```bash
   oc get secret fab-rig-repo -n openshift-gitops
   ```

   **Expected output:**
   ```
   NAME           TYPE     DATA   AGE
   fab-rig-repo   Opaque   4      30s
   ```

2. **Verify ArgoCD recognizes it:**
   ```bash
   oc get secret -n openshift-gitops -l argocd.argoproj.io/secret-type=repository
   ```

   **Expected output:**
   ```
   NAME           TYPE     DATA   AGE
   fab-rig-repo   Opaque   4      30s
   ```

3. **Test ArgoCD can access your repository:**
   ```bash
   # Check if root-app syncs successfully
   oc get application root-app -n openshift-gitops
   ```

   **Expected output (after a few seconds):**
   ```
   NAME       SYNC STATUS   HEALTH STATUS
   root-app   Synced        Healthy
   ```

   If you see `Synced` and `Healthy`, the configuration is working! ✅

---

## Method 2: SSH Keys (Alternative)

If you prefer SSH authentication over HTTPS tokens:

### Step 1: Generate SSH Key

```bash
# Generate a new SSH key specifically for ArgoCD
ssh-keygen -t ed25519 -C "argocd@fab-rig" -f ~/.ssh/argocd-fab-rig

# When prompted:
# - "Enter passphrase": Press Enter (leave empty for automated access)
# - "Enter same passphrase again": Press Enter
```

**Expected output:**
```
Generating public/private ed25519 key pair.
Your identification has been saved in /home/user/.ssh/argocd-fab-rig
Your public key has been saved in /home/user/.ssh/argocd-fab-rig.pub
```

### Step 2: Add Public Key to GitHub

1. **Copy the public key:**
   ```bash
   cat ~/.ssh/argocd-fab-rig.pub
   ```

2. **Add to GitHub:**
   - Visit: https://github.com/settings/ssh/new
   - **Title:** `ArgoCD fab-rig`
   - **Key:** Paste the entire output from the `cat` command above
     - Should start with `ssh-ed25519 AAAA...`
   - Click **Add SSH key**

### Step 3: Create ArgoCD Secret with SSH Key

```bash
# Create secret with the private key
oc create secret generic fab-rig-repo-ssh \
  --from-file=sshPrivateKey=$HOME/.ssh/argocd-fab-rig \
  -n openshift-gitops

# Label it so ArgoCD recognizes it as a repository credential
oc label secret fab-rig-repo-ssh \
  argocd.argoproj.io/secret-type=repository \
  -n openshift-gitops

# Add repository URL and type
oc patch secret fab-rig-repo-ssh -n openshift-gitops \
  --type merge \
  -p '{"stringData":{"type":"git","url":"git@github.com:YOUR_USERNAME/fab-rig.git"}}'
```

**Replace `YOUR_USERNAME` with your GitHub username**

### Step 4: Update Root App to Use SSH URL

Edit `rig/baremetal/bootstrap/root-app.yaml`:

```yaml
spec:
  source:
    repoURL: git@github.com:YOUR_USERNAME/fab-rig.git  # Changed from https://
```

Then restart ArgoCD repo server (same as Method 1, Step 4).

## Security Best Practices

1. **Token Expiration:** Use tokens with expiration dates and rotate them regularly
2. **Minimum Scope:** Only grant `repo` scope, nothing more
3. **Secret Management:** Never commit `github-repo-secret.yaml` to Git
   - Already in `.gitignore` as `*-secret.yaml`
4. **Token Storage:** Store tokens securely (e.g., password manager)
5. **Revoke Old Tokens:** When rotating, revoke old tokens immediately

## Troubleshooting

### Error: "authentication required: Repository not found"

This means ArgoCD can't access your repository. Check:

1. **Is repository private?**
   ```bash
   curl -I https://github.com/YOUR_USERNAME/fab-rig
   ```

2. **Is secret created?**
   ```bash
   oc get secret fab-rig-repo -n openshift-gitops
   ```

3. **Is secret labeled correctly?**
   ```bash
   oc get secret fab-rig-repo -n openshift-gitops --show-labels
   # Should have: argocd.argoproj.io/secret-type=repository
   ```

4. **Are credentials correct?**
   ```bash
   # Test with curl
   curl -u YOUR_USERNAME:YOUR_TOKEN https://api.github.com/repos/YOUR_USERNAME/fab-rig
   # Should return repository JSON, not 404
   ```

### Secret Applied But Still Failing

ArgoCD might need to refresh its repository cache:

```bash
# Restart ArgoCD repo server to pick up new credentials
oc delete pod -n openshift-gitops -l app.kubernetes.io/name=openshift-gitops-repo-server

# Wait for pod to restart (~30 seconds)
oc wait --for=condition=ready pod -l app.kubernetes.io/name=openshift-gitops-repo-server -n openshift-gitops --timeout=60s
```

## Removing Configuration

To switch back to public repository or remove credentials:

```bash
# Delete the secret
oc delete secret fab-rig-repo -n openshift-gitops

# Delete local file (contains credentials)
rm manifests/01-private-repo-config/github-repo-secret.yaml
```

## Related Documentation

- [GitHub Personal Access Tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
- [ArgoCD Private Repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
- [Main Deployment Guide](../../rig/baremetal/bootstrap/README.md)
