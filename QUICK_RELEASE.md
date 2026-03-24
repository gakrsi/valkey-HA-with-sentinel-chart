# Quick Release Guide

## Prerequisites

1. **Create GitHub Repository**
   ```bash
   # On GitHub, create a new repository named: valkey-sentinel-ha-helm-chart
   # Then initialize it locally:
   cd /home/gakrsi/workspace/valkey-sentinel-HA-helm-chart
   git init
   git add .
   git commit -m "Initial commit: Valkey Sentinel HA Helm Chart v0.1.0"
   git branch -M main
   git remote add origin git@github.com:YOUR_USERNAME/valkey-sentinel-ha-helm-chart.git
   git push -u origin main
   ```

2. **Enable GitHub Pages**
   - Go to repository Settings → Pages
   - Source: Deploy from a branch
   - Branch: `gh-pages` / `/ (root)`
   - Save

3. **Create gh-pages branch**
   ```bash
   git checkout --orphan gh-pages
   git rm -rf .
   echo "# Valkey Sentinel HA Helm Chart Repository" > README.md
   git add README.md
   git commit -m "Initialize gh-pages"
   git push origin gh-pages
   git checkout main
   ```

## Release Steps

### Automated Release (Recommended)

Just push a tag and GitHub Actions will handle everything:

```bash
# Tag the release
git tag v0.1.0
git push origin v0.1.0
```

That's it! GitHub Actions will:
- ✅ Package the chart
- ✅ Create a GitHub Release
- ✅ Upload the chart package
- ✅ Update the Helm repository index
- ✅ Publish to GitHub Pages

### Manual Release (If needed)

If you need to release manually:

```bash
# 1. Set environment variables
export CR_TOKEN="YOUR_GITHUB_TOKEN"
export CR_OWNER="YOUR_GITHUB_USERNAME"
export CR_GIT_REPO="valkey-sentinel-ha-helm-chart"

# 2. Create and push tag
git tag v0.1.0
git push origin v0.1.0

# 3. Upload chart
cr upload \
  --owner "$CR_OWNER" \
  --git-repo "$CR_GIT_REPO" \
  --package-path .cr-release-packages \
  --token "$CR_TOKEN"

# 4. Update index
cr index \
  --owner "$CR_OWNER" \
  --git-repo "$CR_GIT_REPO" \
  --charts-repo "https://${CR_OWNER}.github.io/${CR_GIT_REPO}" \
  --package-path .cr-release-packages \
  --token "$CR_TOKEN"

# 5. Push index to gh-pages
git checkout gh-pages
git pull origin gh-pages
cp index.yaml .
git add index.yaml
git commit -m "Update index for v0.1.0"
git push origin gh-pages
git checkout main
```

## Verify Release

1. **Check GitHub Release**
   - Visit: `https://github.com/YOUR_USERNAME/valkey-sentinel-ha-helm-chart/releases`
   - Should see v0.1.0 with attached chart package

2. **Check Helm Repository**
   ```bash
   # Add repository
   helm repo add valkey-ha https://YOUR_USERNAME.github.io/valkey-sentinel-ha-helm-chart

   # Search for chart
   helm search repo valkey-ha

   # Should output:
   # NAME                          CHART VERSION   APP VERSION   DESCRIPTION
   # valkey-ha/valkey-sentinel-ha  0.1.0          7.2           A Helm chart for deploying Valkey with Sentin...
   ```

3. **Test Installation**
   ```bash
   helm install test-valkey valkey-ha/valkey-sentinel-ha \
     --namespace test \
     --create-namespace \
     --dry-run
   ```

## Current Package

The chart has already been packaged and is ready:

```
.cr-release-packages/valkey-sentinel-ha-0.1.0.tgz
```

## Next Steps

1. Create GitHub repository
2. Enable GitHub Pages
3. Push code to GitHub
4. Push tag `v0.1.0`
5. GitHub Actions will automatically release the chart

## Users Can Install With

```bash
# Add repository
helm repo add valkey-ha https://YOUR_USERNAME.github.io/valkey-sentinel-ha-helm-chart

# Update repositories
helm repo update

# Install chart
helm install my-valkey-ha valkey-ha/valkey-sentinel-ha \
  --namespace valkey-ha \
  --create-namespace

# For OpenShift
helm install my-valkey-ha valkey-ha/valkey-sentinel-ha \
  -f https://raw.githubusercontent.com/YOUR_USERNAME/valkey-sentinel-ha-helm-chart/main/values-openshift.yaml \
  --namespace valkey-ha \
  --create-namespace
```
