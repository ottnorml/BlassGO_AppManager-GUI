# CI/CD Workflows Documentation

This directory contains GitHub Actions workflows for automated testing, building, and releasing the App Manager GUI application.

## Workflows Overview

### 1. Test and Analyze (`test.yml`)

**Trigger:** Every push and pull request

**Purpose:** Ensures code quality by running static analysis and tests

**Steps:**
- Sets up Flutter environment
- Installs dependencies
- Runs `flutter analyze` to check for code issues
- Runs `flutter test` to validate functionality

**When it runs:**
- On every push to any branch
- On every pull request

**Status:** Check the "Test and Analyze" badge on the main README

---

### 2. Build Multi-Platform (`build.yml`)

**Trigger:** Push to main, pull requests to main, or manual dispatch

**Purpose:** Builds the application for all supported platforms to verify build integrity

**Platforms:**
- **Linux (x64)**: Builds using Ubuntu with GTK3 dependencies
- **macOS (Universal)**: Builds universal binary supporting both Intel and Apple Silicon
- **Windows (x64)**: Builds using Windows with Visual Studio build tools

**Artifacts:**
- Build artifacts are uploaded and retained for 7 days
- Can be downloaded from the workflow run page

**When it runs:**
- On push to `main` branch
- On pull requests to `main` branch
- Manually via "Run workflow" button in GitHub Actions tab

---

### 3. Release (`release.yml`)

**Trigger:** When a version tag is pushed (e.g., `v1.2.5`)

**Purpose:** Automatically creates GitHub releases with pre-built binaries for all platforms

**Process:**
1. Builds application for all three platforms (Linux, macOS, Windows)
2. Packages each build into appropriate archive formats
3. Creates a GitHub release with the tag
4. Uploads all platform binaries as release assets
5. Generates release notes with installation instructions

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
# Encode certificate to base64
base64 -i Certificates.p12 | pbcopy

# Add to GitHub:
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

All workflows use Flutter 3.24.0 (stable channel). To update:

1. Edit the `flutter-version` in each workflow file
2. Test the build locally first
3. Update all three workflows consistently

### Caching

All workflows enable Flutter SDK caching to speed up subsequent runs:

```yaml
- uses: subosito/flutter-action@v2
  with:
    cache: true
```

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

Add status badges to the main README:

```markdown
![Test and Analyze](https://github.com/ottnorml/BlassGO_AppManager-GUI/workflows/Test%20and%20Analyze/badge.svg)
![Build Multi-Platform](https://github.com/ottnorml/BlassGO_AppManager-GUI/workflows/Build%20Multi-Platform/badge.svg)
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

---

## Support

For issues related to CI/CD workflows:
1. Check workflow run logs in the Actions tab
2. Review this documentation
3. Open an issue with workflow logs attached
