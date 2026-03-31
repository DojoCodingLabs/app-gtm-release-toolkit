# Codemagic Configuration Reference

Complete `codemagic.yaml` for a Flutter app with three workflows.

## File Structure

```
project-root/
├── codemagic.yaml
├── scripts/
│   ├── generate_config.sh
│   ├── quality_checks.sh
│   └── upload_symbols.sh
└── lib/core/env/
    ├── env_ci.dart       # Template (committed)
    └── env_ci.g.dart     # Generated (.gitignore)
```

## Complete codemagic.yaml

```yaml
workflows:
  # ── Workflow 1: PR Quality Gate ──────────────────────────
  pr-quality-gate:
    name: PR Quality Gate
    max_build_duration: 30
    instance_type: linux_x2

    triggering:
      events:
        - pull_request
      branch_patterns:
        - pattern: develop
          include: true
          source: true

    environment:
      flutter: stable

    scripts:
      - name: Install dependencies
        script: flutter pub get

      - name: Run quality checks
        script: chmod +x scripts/quality_checks.sh && ./scripts/quality_checks.sh

    artifacts:
      - coverage/lcov.info

  # ── Workflow 2: Android Pipeline ─────────────────────────
  android-pipeline:
    name: Android Build & Release
    max_build_duration: 60
    instance_type: linux_x2

    triggering:
      events:
        - push
      branch_patterns:
        - pattern: develop
          include: true
        - pattern: staging
          include: true
        - pattern: production
          include: true

    environment:
      flutter: stable
      android_signing:
        - android_keystore    # Upload in Codemagic UI > Code signing
      groups:
        - staging_secrets     # STAGING_BASE_URL, STAGING_API_KEY
        - production_secrets  # PROD_BASE_URL, PROD_API_KEY
        - firebase_credentials # FIREBASE_TOKEN, FIREBASE_ANDROID_APP_ID, FIREBASE_GROUPS
        - sentry_credentials  # SENTRY_AUTH_TOKEN, SENTRY_ORG, SENTRY_PROJECT

    scripts:
      - name: Install dependencies
        script: flutter pub get

      - name: Detect environment
        script: |
          BRANCH=$(git rev-parse --abbrev-ref HEAD)
          if [ "$BRANCH" = "develop" ]; then
            echo "ENV=dev" >> $CM_ENV
          elif [ "$BRANCH" = "staging" ]; then
            echo "ENV=staging" >> $CM_ENV
          else
            echo "ENV=production" >> $CM_ENV
          fi

      - name: Generate config
        script: |
          chmod +x scripts/generate_config.sh
          if [ "$ENV" = "dev" ]; then
            ./scripts/generate_config.sh dev "https://dev.api.example.com" "dev_placeholder"
          elif [ "$ENV" = "staging" ]; then
            ./scripts/generate_config.sh staging "$STAGING_BASE_URL" "$STAGING_API_KEY"
          else
            ./scripts/generate_config.sh production "$PROD_BASE_URL" "$PROD_API_KEY"
          fi

      - name: Build Android
        script: |
          if [ "$ENV" = "production" ]; then
            flutter build appbundle --release \
              --obfuscate \
              --split-debug-info=build/symbols
          else
            flutter build appbundle --release
          fi

      - name: Upload Sentry symbols
        script: |
          if [ "$ENV" = "production" ]; then
            chmod +x scripts/upload_symbols.sh
            ./scripts/upload_symbols.sh "$(git rev-parse --short HEAD)"
          fi

      - name: Distribute to Firebase
        script: |
          if [ "$ENV" = "dev" ] || [ "$ENV" = "staging" ]; then
            firebase appdistribution:distribute \
              build/app/outputs/bundle/release/app-release.aab \
              --app "$FIREBASE_ANDROID_APP_ID" \
              --groups "$FIREBASE_GROUPS" \
              --token "$FIREBASE_TOKEN"
          fi

    artifacts:
      - build/app/outputs/bundle/release/app-release.aab
      - build/symbols/**

    publishing:
      google_play:
        credentials: $GOOGLE_PLAY_SERVICE_ACCOUNT_JSON
        track: production
        submit_as_draft: true
      # Only publishes when ENV=production; for dev/staging Firebase handles it

  # ── Workflow 3: iOS Pipeline ─────────────────────────────
  ios-pipeline:
    name: iOS Build & Release
    max_build_duration: 90
    instance_type: mac_mini_m2

    triggering:
      events:
        - push
      branch_patterns:
        - pattern: develop
          include: true
        - pattern: staging
          include: true
        - pattern: production
          include: true

    environment:
      flutter: stable
      ios_signing:
        distribution_type: app_store
        bundle_identifier: com.example.myapp  # Replace with actual bundle ID
      groups:
        - staging_secrets
        - production_secrets
        - app_store_credentials  # APP_STORE_CONNECT keys
        - sentry_credentials

    scripts:
      - name: Install dependencies
        script: flutter pub get

      - name: Detect environment
        script: |
          BRANCH=$(git rev-parse --abbrev-ref HEAD)
          if [ "$BRANCH" = "develop" ]; then
            echo "ENV=dev" >> $CM_ENV
          elif [ "$BRANCH" = "staging" ]; then
            echo "ENV=staging" >> $CM_ENV
          else
            echo "ENV=production" >> $CM_ENV
          fi

      - name: Generate config
        script: |
          chmod +x scripts/generate_config.sh
          if [ "$ENV" = "dev" ]; then
            ./scripts/generate_config.sh dev "https://dev.api.example.com" "dev_placeholder"
          elif [ "$ENV" = "staging" ]; then
            ./scripts/generate_config.sh staging "$STAGING_BASE_URL" "$STAGING_API_KEY"
          else
            ./scripts/generate_config.sh production "$PROD_BASE_URL" "$PROD_API_KEY"
          fi

      - name: Install CocoaPods
        script: |
          cd ios && pod install

      - name: Build iOS
        script: |
          if [ "$ENV" = "dev" ]; then
            flutter build ios --release --no-codesign
          elif [ "$ENV" = "production" ]; then
            flutter build ipa --release \
              --obfuscate \
              --split-debug-info=build/symbols \
              --export-options-plist=/Users/builder/export_options.plist
          else
            flutter build ipa --release \
              --export-options-plist=/Users/builder/export_options.plist
          fi

      - name: Upload Sentry symbols
        script: |
          if [ "$ENV" = "production" ]; then
            chmod +x scripts/upload_symbols.sh
            ./scripts/upload_symbols.sh "$(git rev-parse --short HEAD)"
          fi

    artifacts:
      - build/ios/ipa/*.ipa
      - build/symbols/**
      - /tmp/xcodebuild_logs/*.log

    publishing:
      app_store_connect:
        api_key: $APP_STORE_CONNECT_PRIVATE_KEY
        key_id: $APP_STORE_CONNECT_KEY_IDENTIFIER
        issuer_id: $APP_STORE_CONNECT_ISSUER_ID
        submit_to_testflight: true
        submit_to_app_store: false  # Manual promotion to production
```

## Codemagic-Specific Setup

### Code Signing (Android)

1. Go to **Codemagic UI > Teams > Code signing identities**
2. Upload `upload-keystore.jks`
3. Enter keystore password, key alias, key password
4. Reference name: `android_keystore` (matches `android_signing` in YAML)

Codemagic auto-generates `key.properties` and places it in the build — no manual Gradle config needed in CI.

### Code Signing (iOS)

1. Go to **Codemagic UI > Teams > Code signing identities > iOS**
2. Upload `.p12` distribution certificate
3. Upload `.mobileprovision` provisioning profile
4. Or use **automatic code signing** with App Store Connect API key

For automatic signing, Codemagic manages certificates and profiles:
```yaml
ios_signing:
  distribution_type: app_store
  bundle_identifier: com.example.myapp
```

### Environment Variable Groups

Create these groups in **Codemagic UI > Teams > Environment variables**:

| Group | Variables |
|-------|-----------|
| `staging_secrets` | `STAGING_BASE_URL`, `STAGING_API_KEY` |
| `production_secrets` | `PROD_BASE_URL`, `PROD_API_KEY` |
| `firebase_credentials` | `FIREBASE_TOKEN`, `FIREBASE_ANDROID_APP_ID`, `FIREBASE_GROUPS` |
| `app_store_credentials` | `APP_STORE_CONNECT_PRIVATE_KEY`, `APP_STORE_CONNECT_KEY_IDENTIFIER`, `APP_STORE_CONNECT_ISSUER_ID` |
| `sentry_credentials` | `SENTRY_AUTH_TOKEN`, `SENTRY_ORG`, `SENTRY_PROJECT` |

### $CM_ENV Special File

Writing to `$CM_ENV` makes variables available to all subsequent steps in the workflow. This is how environment detection propagates:

```bash
echo "ENV=staging" >> $CM_ENV
# Now $ENV is available in all following steps
```

### Webhook Notifications

Add to any workflow for Slack/Discord notifications:

```yaml
publishing:
  slack:
    channel: '#releases'
    notify_on_build_start: false
    notify:
      success: true
      failure: true
```

## Trigger Cascade

```
PR opened → develop
  └─ pr-quality-gate runs (Linux)

PR merged → develop
  ├─ android-pipeline (ENV=dev, Firebase)
  └─ ios-pipeline (ENV=dev, unsigned)

develop merged → staging
  ├─ android-pipeline (ENV=staging, Firebase)
  └─ ios-pipeline (ENV=staging, TestFlight)

staging merged → production
  ├─ android-pipeline (ENV=production, Play Store + Sentry)
  └─ ios-pipeline (ENV=production, App Store + Sentry)
```
