# firstcs — Automated Update & Release System

## Overview

firstcs uses **Velopack** and **GitHub Actions** to provide automated application releases and automatic updates for Windows users.

The system separates source code from public application releases:

- The source code repository remains private.
- GitHub Actions automatically builds and packages the application.
- A separate public GitHub repository distributes application releases.
- Installed applications check the public release repository for updates.
- Velopack downloads and installs updates automatically.

This architecture allows us to distribute application updates without maintaining a dedicated update server.

---

## 1. Architecture

```text
                 DEVELOPER
                     |
                     |
             Update application
             Version in csproj
                     |
                     v
           PRIVATE GITHUB REPOSITORY
                 firstcs
                     |
                     |
              Push Git Tag
                v1.0.5
                     |
                     v
              GITHUB ACTIONS
                     |
              Build WPF App
                     |
              Publish win-x64
                     |
              Velopack Packaging
                     |
                     v
          PUBLIC GITHUB REPOSITORY
              firstcs-releases
                     |
          +----------+----------+
          |          |          |
          v          v          v
        Setup.exe  Packages   Manifest
          |
          v
       CUSTOMER
          |
          v
    Installed firstcs
          |
          v
    Check for Updates
          |
          v
    Download New Version
          |
          v
    Install and Restart
```

## 2. Technologies

| Technology | Purpose |
|---|---|
| WPF | Windows desktop application |
| .NET 8 | Application framework |
| C# | Application development |
| Velopack | Packaging and automatic updates |
| GitHub Actions | Automated build and release |
| GitHub Releases | Public update distribution |
| PowerShell | Release workflow scripting |

### Version Information

- Velopack NuGet package: `1.2.161`
- Velopack CLI: `1.2.161`
- Target framework: `net8.0-windows`
- Runtime: `win-x64`
- Deployment: Self-contained

The NuGet package and CLI versions are kept synchronized.

## 3. Repository Structure

### Private source repository

Repository:

https://github.com/SachithDilshan404/firstcs

Contains:

- WPF application source code
- Project configuration
- AutoUpdateService
- GitHub Actions workflow
- Application resources and assets

The source repository is not publicly distributed.

### Public release repository

Repository:

https://github.com/SachithDilshan404/firstcs-releases

Contains:

- Windows installer
- Portable application package
- Full NuGet update package
- Velopack release manifest
- GitHub release information

Customers use this repository to download new application versions and receive automatic updates.

## 4. Automatic Update Implementation

The application uses a dedicated `AutoUpdateService` class.

The update source is configured using Velopack's `GithubSource`.

```csharp
using Velopack;
using Velopack.Sources;

var source = new GithubSource(
    "https://github.com/SachithDilshan404/firstcs-releases",
    accessToken: null,
    prerelease: false
);

updateManager = new UpdateManager(source);
```

### Configuration explanation

| Property | Description |
|---|---|
| Repository URL | Public release repository |
| accessToken | `null` because the repository is public |
| prerelease | `false` to receive stable releases only |

### Update process

1. The application initializes its update manager.
2. It checks whether the application is installed through Velopack.
3. It checks the configured GitHub release source.
4. If an update is available, it downloads the update.
5. The application installs the update.
6. Velopack restarts the application.

**Important:** Applications must be installed through the Velopack installer for this update mechanism to work correctly. Running directly from a development or publish folder is not equivalent to a Velopack installation.

## 5. Automated Release Workflow

GitHub Actions handles the entire release process.

Workflow location:

`.github/workflows/release.yml`

### Workflow trigger

The workflow runs when a version tag matching `v*` is pushed.

Example:

```text
v1.0.4
v1.0.5
v1.0.6
```

The workflow validates that the tag follows this format:

```text
vMAJOR.MINOR.PATCH
```

### Release pipeline

**Step 1 — Checkout source**

Downloads the private source repository.

**Step 2 — Setup .NET**

Installs the .NET 8 SDK.

**Step 3 — Extract version**

Reads the version from the Git tag and stores it as `APP_VERSION`.

**Step 4 — Restore dependencies**

```powershell
dotnet restore ./firstcs.csproj
```

**Step 5 — Publish application**

```powershell
dotnet publish ./firstcs.csproj `
    -c Release `
    -r win-x64 `
    --self-contained true `
    -o ./publish
```

**Step 6 — Install Velopack CLI**

```powershell
dotnet tool install --global vpk --version 1.2.161
```

**Step 7 — Download previous release**

Downloads the previous Velopack release when available.

This allows Velopack to generate differential update packages where supported.

The first release does not have a previous release to download.

**Step 8 — Package application**

```powershell
vpk pack `
    --packId firstcs `
    --packVersion "$env:APP_VERSION" `
    --packDir ./publish `
    --mainExe TestApplication.exe `
    --outputDir ./Releases
```

**Step 9 — Upload release**

```powershell
vpk upload github `
    --repoUrl "https://github.com/SachithDilshan404/firstcs-releases" `
    --outputDir ./Releases `
    --publish `
    --releaseName "firstcs $env:APP_VERSION" `
    --tag "v1.0.5"
```

The actual workflow dynamically obtains the release tag rather than hardcoding `v1.0.5`.

## 6. GitHub Authentication

The workflow uses a GitHub Fine-grained Personal Access Token.

### Token configuration

| Setting | Value |
|---|---|
| Token name | `firstcs-release-automation` |
| Repository access | Only `firstcs-releases` |
| Permission | Contents: Read and write |
| Storage | Private repository GitHub Actions secrets |
| Secret name | `RELEASE_TOKEN` |

The workflow accesses this token through:

```yaml
env:
  VPK_TOKEN: ${{ secrets.RELEASE_TOKEN }}
```

The GitHub API request used to check for the previous release also uses the same secret to avoid unauthenticated API rate limits.

**Security rules:**

- Never commit the token into source code.
- Never expose it in application configuration.
- Never distribute it to customers.
- Rotate the token before its expiration.
- Keep repository permissions restricted.

Customers do not need a GitHub access token because the release repository is public.

## 7. How to Publish a New Release

This is the standard release procedure.

### Step 1 — Update application version

Open `firstcs.csproj`.

Find:

```xml
<Version>1.0.5</Version>
```

Change it to the next version.

Example:

```xml
<Version>1.0.6</Version>
```

### Step 2 — Commit and push

Run:

```powershell
git add .
git commit -m "Release version 1.0.6"
git push origin master
```

### Step 3 — Create a release tag

Run:

```powershell
git tag v1.0.6
```

### Step 4 — Push the tag

```powershell
git push origin v1.0.6
```

### Step 5 — Monitor GitHub Actions

Open:

https://github.com/SachithDilshan404/firstcs/actions

Find the workflow:

`Build and Release firstcs`

Wait until the workflow completes successfully.

### Step 6 — Verify the public release

Open:

https://github.com/SachithDilshan404/firstcs-releases/releases

Verify that the new version appears with the Latest badge and contains the expected Velopack assets.

### Step 7 — Test the update

Before announcing a release to customers:

1. Install the previous application version.
2. Publish the new release.
3. Open the installed application.
4. Check for updates.
5. Verify that the update downloads and installs.
6. Verify that the application restarts with the new version.

---

## 8. Release Assets

Each release should contain the following assets:

| Asset | Purpose |
|---|---|
| `firstcs-win-Setup.exe` | Installer for new customers |
| `firstcs-win-Portable.zip` | Portable application distribution |
| `firstcs-X.Y.Z-full.nupkg` | Full application update package |
| `releases.win.json` | Velopack update manifest |
| `RELEASES` | Legacy compatibility release information |

GitHub also automatically generates source-code ZIP and TAR archives. These are generated by GitHub and are not the Velopack application installer.

## 9. Troubleshooting

### GitHub API rate limit exceeded

**Cause:** GitHub's unauthenticated API request limit was exceeded.

**Solution:** Authenticate the previous-release API request using the `RELEASE_TOKEN` secret.

### Repository is empty

**Cause:** GitHub cannot publish a release in a repository that has no commits.

**Solution:** Initialize the public release repository with a `README.md` before publishing the first release.

### Update not detected

Check:

- The release is published, not a draft.
- The public repository URL is correct.
- `releases.win.json` exists.
- The installed application is a Velopack installation.
- The new release version is higher than the installed version.
- The updater is configured for stable releases.

### GitHub Actions fails

Open the failed workflow run and inspect the step that produced the error.

Check the GitHub Actions secret, repository permissions, .NET configuration, Velopack CLI version, and release assets.

### Duplicate release tag

A release tag must be unique. Check whether the tag or release already exists before retrying a failed release.

Avoid deleting published releases or changing existing release tags unless you understand the consequences for installed customers.

---

## 10. Current Implementation Status

| Feature | Status |
|---|---|
| Private source repository | Working |
| Public release repository | Working |
| GitHub Actions automation | Working |
| .NET application publishing | Working |
| Velopack packaging | Working |
| GitHub release upload | Working |
| Automatic update detection | Tested |
| Automatic download | Tested |
| Installation and restart | Tested |
| Version 1.0.4 → 1.0.5 update | Successfully tested |

## 11. Future Improvements

Potential improvements before wider commercial distribution:

- Trusted code signing for Windows releases.
- More comprehensive update failure handling.
- Rollback and recovery strategy.
- Automated release validation.
- Release notifications.
- Update telemetry and diagnostics.
- Staged rollout for larger customer deployments.

---

## Summary

The firstcs update infrastructure provides a lightweight and automated way to distribute Windows application updates.

The developer maintains the private source repository, GitHub Actions builds and publishes new versions, and Velopack handles update delivery to installed customers.

**The complete 1.0.4 → 1.0.5 update process has been successfully tested.**

No dedicated update server is currently required.
