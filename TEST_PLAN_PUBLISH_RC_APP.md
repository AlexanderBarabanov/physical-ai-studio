# Comprehensive Test Plan: publish-rc-app Workflow

## Overview

This test plan validates the `publish-rc-app.yml` workflow for the Physical AI Studio application release candidate publishing process. The workflow builds, signs, and publishes Docker container images to GitHub Container Registry (GHCR).

**Testing Account:** `alexanderbarabanov`  
**Target Registry:** `ghcr.io/alexanderbarabanov/test-image-cpu`  
**Estimated Time:** 2-3 hours for complete test suite

---

## Prerequisites Setup

### 1. Fork Configuration

```powershell
# Clone your fork
git clone https://github.com/alexanderbarabanov/physical-ai-studio.git
cd physical-ai-studio

# Add upstream remote (optional)
git remote add upstream https://github.com/open-edge-platform/physical-ai-studio.git
```

### 2. Enable GitHub Actions in Fork

1. Go to **Settings → Actions → General**
2. Select **"Allow all actions and reusable workflows"**
3. Click **Save**

### 3. Configure GHCR Permissions

1. Go to **Settings → Actions → General → Workflow permissions**
2. Select **"Read and write permissions"**
3. Check **"Allow GitHub Actions to create and approve pull requests"**
4. Click **Save**

### 4. Verify Repository Structure

Ensure these files exist:
- `application/VERSION`
- `application/docker/Dockerfile`
- `application/backend/pyproject.toml`
- `.github/workflows/publish-rc-app.yml`

---

## Test Scenarios

### Test Scenario 1: Valid RC1 Release (Happy Path)

**Purpose:** Validate complete workflow execution for first release candidate

**Setup:**
```powershell
# Create release branch from main
git checkout main
git pull origin main

# Ensure VERSION file exists with correct format
echo "0.1.0" | Out-File -NoNewline -Encoding utf8 application\VERSION

# Create and push release branch
git checkout -b release/app-0.1
git add application\VERSION
git commit -m "Prepare release/app-0.1 branch"
git push origin release/app-0.1
```

**Execute:**
```powershell
# Create and push RC tag on the release branch
git tag app/v0.1.0rc1
git push origin app/v0.1.0rc1
```

**Expected Results:**
- ✅ Workflow triggers automatically
- ✅ Validation job passes all checks:
  - RC number validation
  - VERSION file format validation
  - Tag/VERSION match validation
  - Release branch existence
  - Commit on release branch
- ✅ Image builds: `ghcr.io/alexanderbarabanov/test-image-cpu:0.1.0rc1`
- ✅ Image is signed with Cosign
- ✅ Build completes in ~5-10 minutes

**Verify:**
```powershell
# Check workflow run
# URL: https://github.com/alexanderbarabanov/physical-ai-studio/actions

# Pull and inspect the image (after workflow completes)
docker pull ghcr.io/alexanderbarabanov/test-image-cpu:0.1.0rc1
docker inspect ghcr.io/alexanderbarabanov/test-image-cpu:0.1.0rc1

# Verify labels
docker inspect ghcr.io/alexanderbarabanov/test-image-cpu:0.1.0rc1 --format='{{json .Config.Labels}}' | ConvertFrom-Json | Format-List
```

**Cleanup:** Keep for subsequent tests

---

### Test Scenario 2: Valid RC2 Release

**Purpose:** Validate incremental RC creation on same release branch

**Setup:**
```powershell
# On the existing release branch, make a change
git checkout release/app-0.1

# Make a small change (e.g., update README)
echo "`n# RC2 test" | Add-Content README.md
git add README.md
git commit -m "Test change for RC2"
git push origin release/app-0.1
```

**Execute:**
```powershell
git tag app/v0.1.0rc2
git push origin app/v0.1.0rc2
```

**Expected Results:**
- ✅ Workflow triggers
- ✅ RC number validation passes (rc2)
- ✅ Image builds: `ghcr.io/alexanderbarabanov/test-image-cpu:0.1.0rc2`
- ✅ Image is signed

**Cleanup:** Keep for subsequent tests

---

### Test Scenario 3: Invalid RC Number (rc0)

**Purpose:** Validate RC number format validation

**Execute:**
```powershell
git checkout release/app-0.1
git tag app/v0.1.0rc0
git push origin app/v0.1.0rc0
```

**Expected Results:**
- ❌ Validation job fails with error:
  ```
  Invalid RC number '0'. RC number must be a positive integer (rc1, rc2, etc.)
  ```
- ❌ Build does not proceed
- ❌ No image is published

**Cleanup:**
```powershell
git push --delete origin app/v0.1.0rc0
git tag -d app/v0.1.0rc0
```

---

### Test Scenario 4: Version Mismatch

**Purpose:** Validate tag version must match VERSION file

**Setup:**
```powershell
git checkout release/app-0.1

# Update VERSION file to different version
echo "0.2.0" | Out-File -NoNewline -Encoding utf8 application\VERSION
git add application\VERSION
git commit -m "Change VERSION to 0.2.0"
git push origin release/app-0.1
```

**Execute:**
```powershell
# Try to create RC for 0.1.0 (but VERSION says 0.2.0)
git tag app/v0.1.0rc3
git push origin app/v0.1.0rc3
```

**Expected Results:**
- ❌ Validation fails with error:
  ```
  Tag version '0.1.0' does not match VERSION file '0.2.0'
  ```
- ❌ Build does not proceed

**Cleanup:**
```powershell
# Revert VERSION file
echo "0.1.0" | Out-File -NoNewline -Encoding utf8 application\VERSION
git add application\VERSION
git commit -m "Revert VERSION to 0.1.0"
git push origin release/app-0.1

# Delete bad tag
git push --delete origin app/v0.1.0rc3
git tag -d app/v0.1.0rc3
```

---

### Test Scenario 5: Invalid VERSION Format

**Purpose:** Validate VERSION file semver format

**Setup:**
```powershell
git checkout release/app-0.1

# Set invalid VERSION format (missing patch)
echo "0.1" | Out-File -NoNewline -Encoding utf8 application\VERSION
git add application\VERSION
git commit -m "Invalid VERSION format"
git push origin release/app-0.1
```

**Execute:**
```powershell
git tag app/v0.1.0rc4
git push origin app/v0.1.0rc4
```

**Expected Results:**
- ❌ Validation fails with error:
  ```
  VERSION file contains invalid semver format: '0.1'
  ```
- ❌ Build does not proceed

**Cleanup:**
```powershell
echo "0.1.0" | Out-File -NoNewline -Encoding utf8 application\VERSION
git add application\VERSION
git commit -m "Fix VERSION format"
git push origin release/app-0.1

git push --delete origin app/v0.1.0rc4
git tag -d app/v0.1.0rc4
```

---

### Test Scenario 6: Tag on Wrong Branch (main)

**Purpose:** Validate tag must be on correct release branch

**Execute:**
```powershell
# Create tag on main branch instead of release branch
git checkout main
git tag app/v0.1.0rc5
git push origin app/v0.1.0rc5
```

**Expected Results:**
- ❌ Validation fails with error:
  ```
  Tag 'app/v0.1.0rc5' (commit <sha>) is not on release branch 'release/app-0.1'
  RC tags must be created from commits on the corresponding release branch
  ```
- ❌ Build does not proceed

**Cleanup:**
```powershell
git push --delete origin app/v0.1.0rc5
git tag -d app/v0.1.0rc5
```

---

### Test Scenario 7: Release Branch Doesn't Exist

**Purpose:** Validate release branch must exist before tagging

**Execute:**
```powershell
# Create tag for version 0.2.0 without creating release branch
git checkout main
git tag app/v0.2.0rc1
git push origin app/v0.2.0rc1
```

**Expected Results:**
- ❌ Validation fails with error:
  ```
  Release branch 'release/app-0.2' does not exist
  ```
- ❌ Build does not proceed

**Cleanup:**
```powershell
git push --delete origin app/v0.2.0rc1
git tag -d app/v0.2.0rc1
```

---

### Test Scenario 8: Missing Required Files

**Purpose:** Validate required files existence check

**Setup:**
```powershell
git checkout release/app-0.1

# Temporarily remove Dockerfile (simulate missing file)
git mv application/docker/Dockerfile application/docker/Dockerfile.bak
git commit -m "Remove Dockerfile temporarily"
git push origin release/app-0.1
```

**Execute:**
```powershell
git tag app/v0.1.0rc6
git push origin app/v0.1.0rc6
```

**Expected Results:**
- ❌ Validation fails with error:
  ```
  Required file missing: application/docker/Dockerfile
  ```
- ❌ Build does not proceed

**Cleanup:**
```powershell
git checkout release/app-0.1
git mv application/docker/Dockerfile.bak application/docker/Dockerfile
git commit -m "Restore Dockerfile"
git push origin release/app-0.1

git push --delete origin app/v0.1.0rc6
git tag -d app/v0.1.0rc6
```

---

### Test Scenario 9: Invalid Tag Format

**Purpose:** Validate tag format pattern matching

**Execute:**
```powershell
# Tag without 'rc' prefix (regular release format)
git checkout release/app-0.1
git tag app/v0.1.0
git push origin app/v0.1.0
```

**Expected Results:**
- ❌ Workflow doesn't trigger (tag pattern doesn't match)
- ℹ️ This is expected behavior - regular releases use different workflow

**Cleanup:**
```powershell
git push --delete origin app/v0.1.0
git tag -d app/v0.1.0
```

---

### Test Scenario 10: Verify Image Metadata

**Purpose:** Validate OCI labels and metadata

**Prerequisites:** Complete Test Scenario 1 successfully

**Execute:**
```powershell
# Pull the image
docker pull ghcr.io/alexanderbarabanov/test-image-cpu:0.1.0rc1

# Inspect labels
docker inspect ghcr.io/alexanderbarabanov/test-image-cpu:0.1.0rc1 --format='{{json .Config.Labels}}' | ConvertFrom-Json | Format-List

# Verify signature (requires cosign)
# Install cosign from: https://docs.sigstore.dev/cosign/installation/
cosign verify ghcr.io/alexanderbarabanov/test-image-cpu:0.1.0rc1 `
  --certificate-identity-regexp='.*' `
  --certificate-oidc-issuer='https://token.actions.githubusercontent.com'
```

**Expected Labels:**
- `org.opencontainers.image.title=Physical AI Studio (cpu)`
- `org.opencontainers.image.version=0.1.0rc1`
- `org.opencontainers.image.created=<timestamp>`
- `org.opencontainers.image.source=https://github.com/alexanderbarabanov/physical-ai-studio`
- `org.opencontainers.image.revision=<commit-sha>`
- `ai.physical.release.type=rc`
- `ai.physical.device=cpu`

---

### Test Scenario 11: Cache Disabled Verification

**Purpose:** Verify build cache is disabled for RC builds

**Prerequisites:** Complete Test Scenario 1

**Execute:**
Monitor the workflow run logs during build

**Expected Results:**
- ✅ Build step should NOT show "importing cache from" messages
- ✅ Clean build from scratch (longer build time expected)
- ✅ All dependencies downloaded fresh
- ✅ No cache layers reused

**Verify in Logs:**
Look for the docker build output - should not contain cache import statements

---

### Test Scenario 12: Patch Release Without Base Tag ⚠️ NEW

**Purpose:** Validate that patch releases require base release tag to exist

**Setup:**
```powershell
# Create release branch for version 0.2.0
git checkout main
echo "0.2.0" | Out-File -NoNewline -Encoding utf8 application\VERSION
git checkout -b release/app-0.2
git add application\VERSION
git commit -m "Prepare release/app-0.2 branch"
git push origin release/app-0.2
```

**Execute:**
```powershell
# Try to create patch release WITHOUT creating base release first
# Increment VERSION to 0.2.1 (patch)
echo "0.2.1" | Out-File -NoNewline -Encoding utf8 application\VERSION
git add application\VERSION
git commit -m "Increment to patch version 0.2.1"
git push origin release/app-0.2

# Create patch RC tag (but base release app/v0.2.0 doesn't exist)
git tag app/v0.2.1rc1
git push origin app/v0.2.1rc1
```

**Expected Results:**
- ❌ Validation fails with error:
  ```
  Patch release tag 'app/v0.2.1rc1' requires base release tag 'app/v0.2.0' to exist
  Cannot release patch version 0.2.1 without base version 0.2.0
  ```
- ❌ Build does not proceed
- ℹ️ Patch validation section shown in logs

**Cleanup:**
```powershell
git push --delete origin app/v0.2.1rc1
git tag -d app/v0.2.1rc1
git push --delete origin release/app-0.2
git checkout main
git branch -D release/app-0.2
```

---

### Test Scenario 13: Valid Patch Release (With Base Tag) ⚠️ NEW

**Purpose:** Validate successful patch release with proper base tag

**Setup:**
```powershell
# Create and complete base release first
git checkout main
echo "0.3.0" | Out-File -NoNewline -Encoding utf8 application\VERSION
git checkout -b release/app-0.3
git add application\VERSION
git commit -m "Prepare release/app-0.3 branch"
git push origin release/app-0.3

# Create base release tag (simulating successful 0.3.0 release)
git tag app/v0.3.0
git push origin app/v0.3.0
```

**Execute:**
```powershell
# Now create patch release
git checkout release/app-0.3
echo "0.3.1" | Out-File -NoNewline -Encoding utf8 application\VERSION
git add application\VERSION
git commit -m "Increment to patch version 0.3.1"
git push origin release/app-0.3

# Create patch RC tag
git tag app/v0.3.1rc1
git push origin app/v0.3.1rc1
```

**Expected Results:**
- ✅ Validation passes (base tag `app/v0.3.0` exists)
- ✅ Patch validation message shown in logs:
  ```
  Patch version detected: 0.3.1
  Base version: 0.3.0
  Expected base tag: app/v0.3.0
  ✓ Base release tag app/v0.3.0 exists
  ```
- ✅ Image builds: `ghcr.io/alexanderbarabanov/test-image-cpu:0.3.1rc1`
- ✅ Image is signed

**Cleanup:** Keep for Scenario 14

---

### Test Scenario 14: Multiple Sequential Patches ⚠️ NEW

**Purpose:** Validate multiple patches from same base version

**Prerequisites:** Complete Test Scenario 13 first

**Setup:**
```powershell
# Use existing release/app-0.3 with base tag app/v0.3.0
git checkout release/app-0.3

# Create 0.3.2 (second patch)
echo "0.3.2" | Out-File -NoNewline -Encoding utf8 application\VERSION
git add application\VERSION
git commit -m "Increment to patch version 0.3.2"
git push origin release/app-0.3
```

**Execute:**
```powershell
git tag app/v0.3.2rc1
git push origin app/v0.3.2rc1
```

**Expected Results:**
- ✅ Validation passes (still checks for base `app/v0.3.0`, not `app/v0.3.1`)
- ✅ Image builds: `ghcr.io/alexanderbarabanov/test-image-cpu:0.3.2rc1`
- ℹ️ Note: Validation always checks for `major.minor.0` base, not previous patch

**Cleanup:**
```powershell
git push --delete origin app/v0.3.0
git push --delete origin app/v0.3.1rc1
git push --delete origin app/v0.3.2rc1
git tag -d app/v0.3.0
git tag -d app/v0.3.1rc1
git tag -d app/v0.3.2rc1
git push --delete origin release/app-0.3
git checkout main
git branch -D release/app-0.3
```

---

## Complete Cleanup

After completing all tests, clean up all artifacts:

```powershell
# Delete all test tags
$tags = @(
  "app/v0.1.0rc1",
  "app/v0.1.0rc2"
  # Add any other tags created during testing
)

foreach ($tag in $tags) {
  git push --delete origin $tag 2>$null
  git tag -d $tag 2>$null
}

# Delete release branches
$branches = @(
  "release/app-0.1",
  "release/app-0.2",
  "release/app-0.3"
)

foreach ($branch in $branches) {
  git push --delete origin $branch 2>$null
  git branch -D $branch 2>$null
}

# Return to main
git checkout main
git pull origin main
```

**Delete Published Images:**
1. Go to https://github.com/alexanderbarabanov?tab=packages
2. Find `test-image-cpu` package
3. Click on it → Package settings → Delete package (if needed)

---

## Monitoring & Debugging

### View Workflow Runs
```
https://github.com/alexanderbarabanov/physical-ai-studio/actions/workflows/publish-rc-app.yml
```

### Check Published Images
```
https://github.com/alexanderbarabanov?tab=packages
https://github.com/alexanderbarabanov/test-image-cpu/pkgs/container/test-image-cpu
```

### View Workflow Run Details
1. Click on the workflow run
2. Expand "Validate tag and VERSION" job to see validation output
3. Expand "Build & Push RC" job to see build logs
4. Check "Sign Docker image" step for signing confirmation

### Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| 403 Forbidden on GHCR | Workflow permissions | Enable "Read and write permissions" in Settings → Actions |
| Workflow doesn't trigger | Tag pattern mismatch | Ensure tag matches `app/v[0-9]+\.[0-9]+\.[0-9]+rc[0-9]+` |
| Cache errors | Expected with `use-cache: false` | These warnings are expected for RC builds |
| Sign-image failures | Missing `id-token: write` | Permission is already configured in workflow |
| Build context warnings | `libs=library` unused | Expected with minimal Dockerfile, not an error |

---

## Success Criteria Checklist

Use this checklist to verify complete workflow functionality:

- [ ] ✅ Valid RC tags trigger successful builds
- [ ] ❌ Invalid RC numbers (rc0) are rejected
- [ ] ❌ VERSION mismatches are caught
- [ ] ❌ Invalid semver formats are rejected
- [ ] ✅ Tags must be on correct release branch
- [ ] ❌ Tags on wrong branches are rejected
- [ ] ❌ Missing release branches are detected
- [ ] ❌ Missing required files are caught
- [ ] ✅ **Patch releases require base release tag to exist**
- [ ] ✅ **Multiple patches can be released from same base**
- [ ] ✅ Images are published to `ghcr.io/alexanderbarabanov/test-image-cpu`
- [ ] ✅ Images are signed with Cosign
- [ ] ✅ OCI metadata labels are correct
- [ ] ✅ Build cache is disabled for RC builds
- [ ] ✅ Workflow completes in reasonable time (~5-10 minutes)

---

## Recommended Test Execution Order

Execute tests in this order for comprehensive validation:

| Order | Scenario | Type | Duration | Purpose |
|-------|----------|------|----------|---------|
| 1 | Scenario 1 | ✅ Pass | 10 min | Happy path baseline |
| 2 | Scenario 2 | ✅ Pass | 10 min | Incremental RC |
| 3 | Scenario 10 | ✅ Pass | 5 min | Metadata verification |
| 4 | Scenario 11 | ✅ Pass | 2 min | Cache verification |
| 5 | Scenario 3 | ❌ Fail | 1 min | RC number validation |
| 6 | Scenario 4 | ❌ Fail | 2 min | Version mismatch |
| 7 | Scenario 5 | ❌ Fail | 2 min | Semver format |
| 8 | Scenario 6 | ❌ Fail | 1 min | Wrong branch |
| 9 | Scenario 7 | ❌ Fail | 1 min | Missing branch |
| 10 | Scenario 8 | ❌ Fail | 2 min | Missing files |
| 11 | Scenario 9 | ℹ️ Skip | 1 min | Pattern mismatch |
| 12 | Scenario 12 | ❌ Fail | 2 min | Patch without base |
| 13 | Scenario 13 | ✅ Pass | 10 min | Valid patch release |
| 14 | Scenario 14 | ✅ Pass | 10 min | Multiple patches |

**Total Time:** ~60-90 minutes (including wait times for builds)

**Quick Smoke Test** (if short on time):
- Scenario 1 (happy path)
- Scenario 3 (invalid RC)
- Scenario 12 (patch validation)
- Scenario 13 (valid patch)

---

## Validation Summary Table

| Validation Check | Scenario(s) | Expected Behavior |
|-----------------|-------------|-------------------|
| RC number format | 3 | Only positive integers allowed (rc1, rc2, ...) |
| VERSION semver | 5 | Must be `major.minor.patch` format |
| Tag/VERSION match | 4 | Tag version must match VERSION file |
| Release branch exists | 7 | Branch `release/app-X.Y` must exist |
| Tag on release branch | 6 | Commit must be ancestor of release branch |
| Required files | 8 | Dockerfile, pyproject.toml, VERSION must exist |
| **Patch base tag** | 12, 13, 14 | For patch > 0, base tag `X.Y.0` must exist |

---

## Notes

1. **Build Time:** ~5-10 minutes per successful build due to no cache usage
2. **Failed Validations:** Fail in seconds, saving CI time
3. **Image Retention:** Test images can be deleted after verification
4. **Tag Format:** Strict pattern matching prevents accidental triggers
5. **Security:** Cache disabled, keyless signing, minimal permissions
6. **Patch Logic:** Always validates against `.0` base, not previous patch

---

## Next Steps After Testing

Once all tests pass:

1. **Update Image Reference:** Change `alexanderbarabanov/test-image-cpu` to `open-edge-platform/physical-ai-studio-cpu` in workflow
2. **Enable Additional Devices:** Add `xpu` and `cuda` to device matrix when ready
3. **Create Release Promotion Workflow:** Implement step 7 from RELEASE.md for promoting RC to final release
4. **Document Process:** Update team documentation with tested workflow
5. **Monitor Production:** Track first production RC release closely

---

## References

- [RELEASE.md](../RELEASE.md) - Official release process documentation
- [publish-rc-app.yml](../.github/workflows/publish-rc-app.yml) - Workflow source
- [Dockerfile](../application/docker/Dockerfile) - Minimal test Dockerfile
- [Cosign Documentation](https://docs.sigstore.dev/cosign/) - Image signing reference
