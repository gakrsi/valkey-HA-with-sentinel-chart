# Releasing the Valkey Sentinel HA Helm Chart

This document describes how to release the Valkey Sentinel HA Helm chart.

## Prerequisites

- `helm` CLI installed
- `chart-releaser` (cr) installed: `brew install chart-releaser` or download from [GitHub](https://github.com/helm/chart-releaser)
- GitHub repository created for hosting the chart
- GitHub Personal Access Token with `repo` scope
- Git configured with access to your repository

## Release Methods

### Method 1: Automated Release (Recommended)

Using GitHub Actions for automated releases on tag push.

1. **Push a new tag:**
   ```bash
   git tag v0.1.0
   git push origin v0.1.0
   ```

2. GitHub Actions will automatically:
   - Package the chart
   - Create a GitHub Release
   - Upload the chart package
   - Update the Helm repository index
   - Deploy to GitHub Pages

### Method 2: Manual Release

#### Step 1: Prepare the Release

1. **Update Chart.yaml version:**
   ```bash
   # Edit Chart.yaml and bump the version
   vim Chart.yaml
   ```

2. **Commit the changes:**
   ```bash
   git add Chart.yaml
   git commit -m "Bump chart version to 0.1.0"
   git push
   ```

3. **Create a Git tag:**
   ```bash
   git tag v0.1.0
   git push origin v0.1.0
   ```

#### Step 2: Package the Chart

```bash
# Create release packages directory
mkdir -p .cr-release-packages

# Package the chart
helm package . -d .cr-release-packages
```

#### Step 3: Upload to GitHub Release

Set your GitHub token and repository details:

```bash
export CR_TOKEN="your-github-token"
export CR_OWNER="your-github-username"
export CR_GIT_REPO="valkey-sentinel-ha-helm-chart"
```

Upload the packaged chart:

```bash
cr upload \
  --owner "$CR_OWNER" \
  --git-repo "$CR_GIT_REPO" \
  --package-path .cr-release-packages \
  --token "$CR_TOKEN"
```

#### Step 4: Update Helm Repository Index

Generate or update the index.yaml:

```bash
cr index \
  --owner "$CR_OWNER" \
  --git-repo "$CR_GIT_REPO" \
  --charts-repo "https://${CR_OWNER}.github.io/${CR_GIT_REPO}" \
  --index-path . \
  --package-path .cr-release-packages \
  --token "$CR_TOKEN"
```

#### Step 5: Publish to GitHub Pages

Commit and push the index to gh-pages branch:

```bash
git checkout gh-pages
cp index.yaml .
git add index.yaml
git commit -m "Update Helm repository index for v0.1.0"
git push origin gh-pages
git checkout main
```

Or use the cr command:

```bash
cr pages-index-push \
  --owner "$CR_OWNER" \
  --git-repo "$CR_GIT_REPO" \
  --token "$CR_TOKEN"
```

## Using the Published Chart

Once published, users can add your repository and install the chart:

```bash
# Add the Helm repository
helm repo add valkey-ha https://${CR_OWNER}.github.io/${CR_GIT_REPO}

# Update repository
helm repo update

# Install the chart
helm install my-valkey-ha valkey-ha/valkey-sentinel-ha \
  --namespace valkey-ha \
  --create-namespace
```

## Release Checklist

Before releasing a new version:

- [ ] Update CHANGELOG.md with changes
- [ ] Update Chart.yaml version (semver)
- [ ] Update Chart.yaml appVersion if Valkey version changed
- [ ] Test the chart in a clean environment
- [ ] Run `helm lint .` to validate
- [ ] Update README.md if configuration changed
- [ ] Create and push a git tag
- [ ] Verify GitHub Release was created
- [ ] Test installation from published repository

## Versioning

This chart follows [Semantic Versioning](https://semver.org/):

- **MAJOR**: Incompatible API changes
- **MINOR**: Backwards-compatible new features
- **PATCH**: Backwards-compatible bug fixes

## Troubleshooting

### Chart upload fails

- Verify GitHub token has `repo` scope
- Ensure tag exists: `git tag -l`
- Check if release already exists on GitHub

### Index update fails

- Verify gh-pages branch exists
- Check GitHub Pages is enabled in repository settings
- Ensure index.yaml is accessible

### Users can't install chart

- Verify GitHub Pages is serving the chart
- Check index.yaml at: `https://${CR_OWNER}.github.io/${CR_GIT_REPO}/index.yaml`
- Confirm chart URL is accessible
