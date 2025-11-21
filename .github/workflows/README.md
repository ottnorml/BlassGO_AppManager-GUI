# CI/CD Workflows Documentation

This directory contains GitHub Actions workflows for automated testing, building, and releasing the App Manager GUI application.

## Workflows Overview

### 1. Test and Analyze (`test.yml`)

**Trigger:** Every push and pull request

**Purpose:** Ensures code quality by running static analysis and tests

**Steps:**
- Sets up Flutter environment with SDK caching
- Caches pub dependencies for faster runs
- Installs dependencies
- Runs `flutter analyze` to check for code issues
- Runs `flutter test` to validate functionality

**Optimizations:**
- Flutter SDK caching enabled
- Pub dependency caching (speeds up subsequent runs by ~3x)

**When it runs:**
- On every push to any branch
- On every pull request

**Status:** Check the "Test and Analyze" badge on the main README

---

### 2. Build Multi-Platform (`build.yml`)

**Trigger:** Push to main, pull requests to main, or manual dispatch

**Purpose:** Builds the application for all supported platforms to verify build integrity

**Strategy:** Uses GitHub Actions matrix strategy for efficient parallel builds

**Platforms:**
- **Linux (x64)**: Builds using Ubuntu with GTK3 dependencies
- **macOS (Universal)**: Builds universal binary supporting both Intel and Apple Silicon
- **Windows (x64)**: Builds using Windows with Visual Studio build tools

**Matrix Configuration:**
- All platforms build in parallel
- `fail-fast: false` ensures all platforms build even if one fails
- Platform-specific configurations defined in matrix include

**Optimizations:**
- Pub dependency caching per platform
- On pull requests, builds only run after tests pass
- Build provenance attestation for security
- Parallel execution across all platforms

**Artifacts:**
- Build artifacts are uploaded and retained for 7 days
- Each artifact includes cryptographic build attestation
- Can be downloaded from the workflow run page

**When it runs:**
- On push to `main` branch
- On pull requests to `main` branch (only after tests pass)
- Manually via "Run workflow" button in GitHub Actions tab

---

### 3. Release (`release.yml`)

**Trigger:** When a version tag is pushed (e.g., `v1.2.5`)

**Purpose:** Automatically creates GitHub releases with pre-built binaries for all platforms

**Strategy:** Reuses build workflow artifacts instead of rebuilding

**Process:**
1. Triggers the build workflow for the release tag
2. Waits for the build workflow to complete successfully
3. Downloads all build artifacts from the completed build workflow run
4. Creates a GitHub release with the tag
5. Uploads all platform binaries as release assets
6. Generates release notes with installation instructions

**Benefits:**
- **No duplication**: Reuses existing build workflow logic
- **Consistency**: Release binaries built exactly the same as regular builds
- **Efficiency**: No need to maintain duplicate build code
- **Attestation**: Inherits build provenance attestation from build workflow

**Security:**
- All release artifacts include cryptographic build attestation (from build workflow)
- Attestations provide verifiable proof of build provenance
- Links artifacts to source code and build process

**Creating a Release:**

```bash
# 1. Update version in pubspec.yaml
# version: 1.2.6+5

# 2. Commit the version change
git add pubspec.yaml
git commit -m "Bump version to 1.2.6"
git push

# 3. Create and push a tag
git tag v1.2.6
git push origin v1.2.6

# 4. The workflow will automatically create a release with binaries
```

**Release Artifacts:**
- `app_manager_linux_x64.tar.gz` - Linux x64 build
- `app_manager_macos.zip` - macOS universal build
- `app_manager_windows_x64.zip` - Windows x64 build

---

## Platform-Specific Notes

### Linux Build

**Dependencies installed:**
- clang
- cmake
- ninja-build
- pkg-config
- libgtk-3-dev
- liblzma-dev
- libstdc++-12-dev

**Output format:** `.tar.gz` archive containing the bundle

### macOS Build

**Current state:** Builds unsigned universal binary

**For code signing (optional):**

To enable code signing for macOS builds, you need to:

1. Obtain an Apple Developer certificate
2. Export it as a `.p12` file
3. Add the following secrets to your GitHub repository:
   - `APPLE_CERTIFICATE`: Base64-encoded .p12 certificate file
   - `APPLE_CERTIFICATE_PASSWORD`: Password for the certificate
   - `APPLE_TEAM_ID`: Your Apple Developer Team ID

4. Uncomment the code signing steps in `release.yml`

**Commands to prepare secrets:**

```bash
# macOS/Linux: Encode certificate to base64
base64 Certificates.p12 > certificate.txt
# Or copy directly (macOS): base64 Certificates.p12 | pbcopy

# Linux (alternative - no line wrapping): 
base64 -w 0 Certificates.p12 > certificate.txt

# Windows (PowerShell): Encode certificate to base64
[Convert]::ToBase64String([IO.File]::ReadAllBytes("Certificates.p12")) | Out-File certificate.txt

# Copy the content of certificate.txt and add to GitHub:
# Settings > Secrets and variables > Actions > New repository secret
```

**Output format:** `.zip` archive containing the `.app` bundle

**Note:** Users may need to right-click and select "Open" on first launch for unsigned builds.

### Windows Build

**Build system:** Uses Visual Studio build tools via Flutter

**Output format:** `.zip` archive containing executable and dependencies

---

## Workflow Configuration

### Flutter Version

All workflows use the latest stable version of Flutter. The workflows automatically use the newest stable release to ensure compatibility with the latest Dart SDK requirements.

To pin to a specific Flutter version (if needed):

1. Edit each workflow file
2. Add `flutter-version: 'X.Y.Z'` under the `with:` section of `subosito/flutter-action@v2`
3. Test the build locally first
4. Update all three workflows consistently

### Caching

All workflows use multiple layers of caching to significantly speed up build times:

**Flutter SDK Caching:**
```yaml
- uses: subosito/flutter-action@v2
  with:
    cache: true
```

**Pub Dependency Caching:**

Each workflow caches pub dependencies per platform:

- **Linux/macOS:**
  ```yaml
  - uses: actions/cache@v4
    with:
      path: |
        ~/.pub-cache
        .dart_tool
      key: ${{ runner.os }}-pub-${{ hashFiles('**/pubspec.yaml') }}
  ```

- **Windows:**
  ```yaml
  - uses: actions/cache@v4
    with:
      path: |
        ~\AppData\Local\Pub\Cache
        .dart_tool
      key: ${{ runner.os }}-pub-${{ hashFiles('**/pubspec.yaml') }}
  ```

**Performance Impact:**
- First run: Full build time
- Subsequent runs: ~3x faster due to dependency caching
- Cache invalidation: Automatic when `pubspec.yaml` changes

### Build Attestation

All build artifacts include cryptographic build provenance attestation for supply chain security:

```yaml
- uses: actions/attest-build-provenance@v1
  with:
    subject-path: app_manager_linux_x64.tar.gz
```

**Benefits:**
- Verifiable proof that artifacts were built by GitHub Actions
- Links artifacts to their source code and build process
- Enables verification of artifact authenticity
- Part of supply chain security best practices

**Required Permissions:**
```yaml
permissions:
  id-token: write  # Required for build attestation
  attestations: write  # Required for build attestation
```

**Viewing Attestations:**
- Attestations are visible in the GitHub UI for each artifact
- Can be verified using the GitHub CLI or API

### Conditional Builds on Pull Requests

The build workflow includes a test job that runs first on pull requests:

```yaml
jobs:
  test:
    if: github.event_name == 'pull_request'
    # ... runs tests
  
  build-linux:
    needs: [test]
    if: |
      always() &&
      (github.event_name != 'pull_request' || needs.test.result == 'success')
```

**Benefits:**
- Saves resources by not building if tests fail
- Faster feedback on test failures
- Builds only run after code quality checks pass

### Matrix Strategy

Both build and release workflows use GitHub Actions matrix strategy for efficient parallel builds:

```yaml
strategy:
  fail-fast: false
  matrix:
    include:
      - platform: Linux
        os: ubuntu-latest
        arch: x64
        desktop-flag: linux
        # ... platform-specific configuration
      
      - platform: macOS
        os: macos-latest
        arch: Universal
        desktop-flag: macos
        # ... platform-specific configuration
      
      - platform: Windows
        os: windows-latest
        arch: x64
        desktop-flag: windows
        # ... platform-specific configuration
```

**Benefits:**
- **Parallel execution**: All platforms build simultaneously
- **DRY principle**: Single job definition for all platforms
- **Fail-safe**: `fail-fast: false` ensures all platforms build even if one fails
- **Maintainability**: Platform configurations defined in one place
- **Extensibility**: Easy to add new platforms or architectures

**Matrix Variables:**
- `platform`: Display name (Linux, macOS, Windows)
- `os`: GitHub runner OS (ubuntu-latest, macos-latest, windows-latest)
- `arch`: Architecture (x64, Universal)
- `build-command`: Platform-specific build command
- `artifact-name`: Output artifact filename
- `package-command`: Platform-specific packaging command
- `cache-path`: Platform-specific cache paths
- `desktop-flag`: Flutter desktop flag (linux, macos, windows)

### Artifact Retention

- **Build workflow**: 7 days
- **Release workflow**: Permanent (attached to release)

---

## Troubleshooting

### Build Failures

**Problem:** Flutter analyze fails

**Solution:** Run `flutter analyze` locally and fix reported issues

**Problem:** Tests fail

**Solution:** Run `flutter test` locally to debug failing tests

**Problem:** Linux build fails with missing dependencies

**Solution:** Ensure all required system libraries are installed in the workflow

**Problem:** macOS build fails

**Solution:** Check that the Xcode version on the runner supports the Flutter version

**Problem:** Windows build fails

**Solution:** Verify Visual Studio build tools are properly configured

### Release Issues

**Problem:** Release is not created

**Solution:** 
- Ensure tag follows the pattern `v*.*.*` (e.g., `v1.2.5`)
- Check workflow permissions allow contents write
- Verify all build jobs completed successfully

**Problem:** Artifacts are missing from release

**Solution:**
- Check individual build job logs
- Verify artifact upload/download steps succeed
- Ensure artifact names match between jobs

---

## Manual Workflow Dispatch

The build workflow can be triggered manually:

1. Go to the "Actions" tab in GitHub
2. Select "Build Multi-Platform" workflow
3. Click "Run workflow"
4. Select branch and click "Run workflow"

This is useful for:
- Testing workflow changes
- Creating test builds without pushing to main
- Debugging build issues

---

## Best Practices

1. **Always test locally first**: Run `flutter analyze`, `flutter test`, and platform builds before pushing

2. **Keep workflows synchronized**: When updating Flutter version or dependencies, update all workflow files

3. **Test before releasing**: Use the build workflow to verify builds work before creating a release tag

4. **Version management**: Always update `pubspec.yaml` version before creating a release tag

5. **Documentation**: Keep this README updated when making workflow changes

---

## Monitoring

### Status Badges

Add status badges to the main README (replace `OWNER/REPO` with your repository path):

```markdown
![Test and Analyze](https://github.com/OWNER/REPO/workflows/Test%20and%20Analyze/badge.svg)
![Build Multi-Platform](https://github.com/OWNER/REPO/workflows/Build%20Multi-Platform/badge.svg)
```

### Notifications

Configure GitHub notifications for workflow failures:
- Settings > Notifications > Actions
- Enable notifications for failed workflows

---

## Future Enhancements

Potential improvements to consider:

1. **Automated version bumping**: Auto-increment version based on commit messages
2. **Changelog generation**: Auto-generate changelogs from commit history
3. **Code coverage**: Add coverage reporting to test workflow
4. **Performance testing**: Add performance benchmarks
5. **Notarization**: Add macOS notarization for enhanced security
6. **Windows signing**: Add code signing for Windows builds
7. **ARM builds**: Add support for ARM64 Linux builds
8. **Android builds**: Add mobile platform builds if needed
9. **Dependency caching optimization**: Fine-tune cache keys for better hit rates
10. **Parallel testing**: Split tests into parallel jobs for faster execution

---

## Recent Improvements

✅ **Implemented:**
- Pub dependency caching (3x faster builds)
- Conditional builds on PRs (only after tests pass)
- Build provenance attestation for supply chain security
- Multi-layer caching (SDK + dependencies)
- **Matrix strategy for parallel platform builds**
- `fail-fast: false` to ensure all platforms build
- **Release workflow reuses build workflow artifacts**

---

## Support

For issues related to CI/CD workflows:
1. Check workflow run logs in the Actions tab
2. Review this documentation
3. Open an issue with workflow logs attached
