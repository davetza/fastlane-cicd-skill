---
name: fastlane-cicd
description: Sets up Fastlane CI/CD pipeline with match code signing for iOS apps. Use when setting up automated builds, TestFlight deployment, or App Store releases for Xcode projects.
license: MIT
metadata:
  author: davetza
  version: "1.0.0"
---

# Fastlane CI/CD Setup for iOS Apps

Complete guide for setting up automated CI/CD pipelines for iOS apps using Fastlane, match for code signing, and GitHub Actions for deployment to TestFlight.

## When to Use This Skill

Use `/fastlane-cicd` when:
- Setting up CI/CD for a new iOS project
- Migrating from manual builds to automated deployment
- Configuring TestFlight or App Store deployment
- Setting up code signing with match
- Creating GitHub Actions workflows for iOS

## Prerequisites

Before starting, ensure you have:
- [ ] Apple Developer account
- [ ] App Store Connect access
- [ ] GitHub repository for the iOS project
- [ ] Bundle identifier registered in Apple Developer portal

## Setup Process

### Phase 1: Apple Setup (Manual Steps)

#### 1. Create App Store Connect API Key
1. Go to https://appstoreconnect.apple.com → Users and Access → Integrations → App Store Connect API
2. Click "+" to generate key with **App Manager** access
3. Download the `.p8` file immediately (one-time download)
4. Note the **Key ID** and **Issuer ID**

#### 2. Register App in App Store Connect (if needed)
1. Go to https://appstoreconnect.apple.com/apps
2. Create: New App → iOS → Your app name → Select Bundle ID

#### 3. Create Certificates Repository
- Create a **private** GitHub repo for certificates (e.g., `username/appname-certificates`)
- Leave empty - match will populate it

#### 4. Generate SSH Deploy Key
```bash
ssh-keygen -t ed25519 -C "fastlane-match" -f ~/.ssh/match_deploy_key
# Add PUBLIC key to certificates repo as deploy key (with write access)
# Add PRIVATE key as MATCH_DEPLOY_KEY GitHub secret
```

### Phase 2: File Creation

Create these files in your project:

#### `Gemfile`
```ruby
source "https://rubygems.org"
gem "fastlane", "~> 2.220"
```

#### `.ruby-version`
```
3.2
```

#### `fastlane/Appfile`
```ruby
app_identifier("YOUR_BUNDLE_ID")
apple_id(ENV["APPLE_ID"] || "your@email.com")
team_id("YOUR_TEAM_ID")
```

#### `fastlane/Matchfile`
```ruby
git_url("git@github.com:USERNAME/REPO-certificates.git")
storage_mode("git")
type("appstore")
app_identifier(["YOUR_BUNDLE_ID"])
team_id("YOUR_TEAM_ID")
username("your@email.com")
```

#### `fastlane/Fastfile`
See the complete Fastfile template below.

#### `.github/workflows/fastlane.yml`
See the complete workflow template below.

### Phase 3: GitHub Secrets

Add these secrets to your GitHub repository:

| Secret | Description |
|--------|-------------|
| `MATCH_DEPLOY_KEY` | SSH private key for certificates repo |
| `MATCH_PASSWORD` | Encryption password for match certificates |
| `APP_STORE_CONNECT_API_KEY` | Base64-encoded .p8 file (`base64 -i AuthKey.p8`) |
| `APP_STORE_CONNECT_KEY_ID` | 10-character key ID |
| `APP_STORE_CONNECT_ISSUER_ID` | UUID issuer ID |

Plus any app-specific secrets (API keys, etc.)

### Phase 4: Initialize Match

Run locally to create certificates:
```bash
bundle install
bundle exec fastlane match appstore --api_key_path /path/to/api_key.json
```

The `api_key.json` format:
```json
{
  "key_id": "YOUR_KEY_ID",
  "issuer_id": "YOUR_ISSUER_ID",
  "key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----",
  "in_house": false
}
```

---

## Complete Templates

### Fastfile Template

```ruby
# Fastfile for iOS CI/CD
default_platform(:ios)

platform :ios do

  before_all do
    # Xcode version selected in GitHub Actions workflow
  end

  # ==========================================================================
  # TESTING
  # ==========================================================================

  desc "Run tests without code signing (for PRs)"
  lane :test_ci do
    generate_secrets_xcconfig if is_ci

    scan(
      project: "YOUR_PROJECT.xcodeproj",
      scheme: "YOUR_SCHEME",
      device: "iPhone 15",
      clean: true,
      code_coverage: true,
      xcargs: "CODE_SIGN_IDENTITY='' CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO"
    )
  end

  # ==========================================================================
  # DEPLOYMENT
  # ==========================================================================

  desc "Build and upload to TestFlight"
  lane :beta do
    generate_secrets_xcconfig if is_ci

    api_key = setup_api_key_if_ci
    UI.user_error!("API key required for beta lane") unless api_key

    setup_signing(api_key: api_key)

    if is_ci && ENV["GITHUB_RUN_NUMBER"]
      increment_build_number(
        build_number: ENV["GITHUB_RUN_NUMBER"],
        xcodeproj: "YOUR_PROJECT.xcodeproj"
      )
    end

    update_code_signing_settings(
      use_automatic_signing: false,
      path: "YOUR_PROJECT.xcodeproj",
      team_id: "YOUR_TEAM_ID",
      profile_name: "match AppStore YOUR_BUNDLE_ID",
      code_sign_identity: "Apple Distribution"
    )

    build_app(
      project: "YOUR_PROJECT.xcodeproj",
      scheme: "YOUR_SCHEME",
      configuration: "Release",
      export_method: "app-store",
      output_directory: "./build",
      output_name: "App.ipa",
      clean: true
    )

    upload_to_testflight(
      api_key: api_key,
      skip_waiting_for_build_processing: true,
      distribute_external: false
    )

    clean_build_artifacts
  end

  # ==========================================================================
  # HELPERS
  # ==========================================================================

  private_lane :setup_signing do |options|
    api_key = options[:api_key]

    if is_ci
      create_keychain(
        name: "fastlane_keychain",
        password: ENV["MATCH_PASSWORD"],
        default_keychain: true,
        unlock: true,
        timeout: 3600,
        lock_when_sleeps: false
      )

      match(
        type: "appstore",
        api_key: api_key,
        readonly: true,
        keychain_name: "fastlane_keychain",
        keychain_password: ENV["MATCH_PASSWORD"]
      )
    else
      match(type: "appstore")
    end
  end

  private_lane :setup_api_key_if_ci do
    if is_ci && ENV["APP_STORE_CONNECT_API_KEY"]
      api_key_content = Base64.decode64(ENV["APP_STORE_CONNECT_API_KEY"])
      api_key_path = File.join(Dir.tmpdir, "AuthKey.p8")
      File.write(api_key_path, api_key_content)

      app_store_connect_api_key(
        key_id: ENV["APP_STORE_CONNECT_KEY_ID"],
        issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
        key_filepath: api_key_path,
        in_house: false
      )
    else
      nil
    end
  end

  private_lane :generate_secrets_xcconfig do
    # Customize this for your app's secrets
    project_root = File.expand_path("..", Dir.pwd)
    secrets_path = File.join(project_root, "Secrets.xcconfig")

    secrets_content = <<~XCCONFIG
      // Auto-generated for CI builds
      API_KEY = #{ENV["API_KEY"] || ""}
    XCCONFIG

    File.write(secrets_path, secrets_content)
    UI.success("Generated Secrets.xcconfig at #{secrets_path}")
  end

end
```

### GitHub Actions Workflow Template

```yaml
name: Fastlane CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      lane:
        description: 'Fastlane lane to run'
        required: true
        default: 'test_ci'
        type: choice
        options:
          - test_ci
          - beta

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Test
    runs-on: macos-14
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
      - run: bundle exec fastlane test_ci

  deploy:
    name: Deploy to TestFlight
    runs-on: macos-14
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Setup SSH for match
        env:
          MATCH_DEPLOY_KEY: ${{ secrets.MATCH_DEPLOY_KEY }}
        run: |
          mkdir -p ~/.ssh
          echo "$MATCH_DEPLOY_KEY" > ~/.ssh/match_deploy_key
          chmod 600 ~/.ssh/match_deploy_key
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          echo "Host github.com" >> ~/.ssh/config
          echo "  IdentityFile ~/.ssh/match_deploy_key" >> ~/.ssh/config
          echo "  IdentitiesOnly yes" >> ~/.ssh/config

      - name: Run beta lane
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
          APP_STORE_CONNECT_KEY_ID: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          GITHUB_RUN_NUMBER: ${{ github.run_number }}
        run: bundle exec fastlane beta

      - name: Cleanup SSH
        if: always()
        run: rm -rf ~/.ssh/match_deploy_key
```

---

## Common Issues & Solutions

### "invalid curve name" Error
The API key JSON needs the key content inline, not a file path:
```json
{
  "key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----"
}
```

### "Missing Secrets.xcconfig"
Ensure `generate_secrets_xcconfig` writes to project root:
```ruby
project_root = File.expand_path("..", Dir.pwd)
```

### Project Format Too New
Update Xcode version in workflow:
```yaml
run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer
```

### "No profiles found"
Use `update_code_signing_settings` before `build_app`:
```ruby
update_code_signing_settings(
  use_automatic_signing: false,
  profile_name: "match AppStore YOUR_BUNDLE_ID"
)
```

### Missing App Icon
App Store requires 1024x1024 PNG icon in asset catalog.

---

## Useful Commands

```bash
# Install dependencies
bundle install

# Run tests locally
bundle exec fastlane test

# Build and deploy to TestFlight
bundle exec fastlane beta

# Sync certificates
bundle exec fastlane match appstore

# Add new device
bundle exec fastlane add_device name:"iPhone" udid:"..."
```

## References

- [Fastlane Documentation](https://docs.fastlane.tools/)
- [Match Documentation](https://docs.fastlane.tools/actions/match/)
- [App Store Connect API](https://developer.apple.com/documentation/appstoreconnectapi)
- [GitHub Actions for iOS](https://docs.github.com/en/actions/deployment/deploying-xcode-applications)
