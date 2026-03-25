# sheepod_test_app
sheepod_test_app is a CI-CD multi-platform example. 

Create a user (service account) inside yourcompany.sheepod.io and generate a TOKEN.

Simple CI-CD (No reusable)
Create the secrets in your GitHub repo or org according to your company needs:
SHEEPOD_ENDPOINT=https://yourcompany.sheepod.io
SHEEPOD_TOKEN=xxxxx
SHEEPOD_CHANNEL_KEY_ANDROID=xxxxx
SHEEPOD_CHANNEL_KEY_LINUX=xxxxx
SHEEPOD_CHANNEL_KEY_ZIP=xxxxx
SHEEPOD_CHANNEL_KEY_WINDOWS=xxxxx
SHEEPOD_CHANNEL_KEY_IOS=xxxxx

+-----------------+---------------------------------+------------------------+----------+
|                                    zealot Options                                     |
+-----------------+---------------------------------+------------------------+----------+
| Key             | Description                     | Env Var                | Default  |
+-----------------+---------------------------------+------------------------+----------+
| endpoint        | The endpoint of sheepod         | ZEALOT_ENDPOINT        |          |
| token           | The token of user               | ZEALOT_TOKEN           |          |
| channel_key     | The key of app's channel        | ZEALOT_CHANNEL_KEY     |          |
| file            | The path of app file. Optional  | ZEALOT_FILE            |          |
|                 | if you use the `gym`, `ipa`,    |                        |          |
|                 | `xcodebuild` or `gradle`        |                        |          |
|                 | action.                         |                        |          |
| name            | The name of app to display on   | ZEALOT_NAME            |          |
|                 | zealot                          |                        |          |
| changelog       | The changelog of app            | ZEALOT_CHANGELOG       |          |
| slug            | The slug of app                 | ZEALOT_SLUG            |          |
| release_type    | The release type of app         | ZEALOT_RELEASE_TYPE    |          |
| branch          | The name of git branch          | ZEALOT_BRANCH          |          |
| git_commit      | The hash of git commit          | ZEALOT_GIT_COMMIT      |          |
| custom_fields   | The key-value hash of custom    | ZEALOT_CUSTOM_FIELDS   |          |
|                 | fields                          |                        |          |
| password        | The password of app to download | ZEALOT_PASSWORD        |          |
| source          | The name of upload source       | ZEALOT_SOURCE          | fastlane |
| ci_url          | The name of upload source       | ZEALOT_CI_CURL         |          |
| timeout         | Request timeout in seconds      | ZEALOT_TIMEOUT         |          |
| hide_user_token | replase user token to *** to    | ZEALOT_HIDE_USER_TOKEN | true     |
|                 | keep secret                     |                        |          |
| verify_ssl      | Should verify SSL of sheepod    | ZEALOT_VERIFY_SSL      | true     |
|                 | service                         |                        |          |
| fail_on_error   | Should an error uploading app   | ZEALOT_FAIL_ON_ERROR   | false    |
|                 | cause a failure                 |                        |          |
+-----------------+---------------------------------+------------------------+----------+

+-----------------+---------------------------------+------------------------+---------+
|                             zealot_version_check Options                             |
+-----------------+---------------------------------+------------------------+---------+
| Key             | Description                     | Env Var                | Default |
+-----------------+---------------------------------+------------------------+---------+
| endpoint        | The endpoint of sheepod         | ZEALOT_ENDPOINT        |         |
| token           | The token of user               | ZEALOT_TOKEN           |         |
| channel_key     | The key of app's channel        | ZEALOT_CHANNEL_KEY     |         |
| bundle_id       | The bundle id(package name) of  | ZEALOT_BUNDLE_ID       |         |
|                 | app                             |                        |         |
| release_version | The release version of app      | ZEALOT_RELEASE_VERSION |         |
| build_version   | The build version of app        | ZEALOT_BUILD_VERSION   |         |
| git_commit      | The latest git commit of app    | ZEALOT_GIT_COMMIT      |         |
| verify_ssl      | Should verify SSL of sheepod    | ZEALOT_VERIFY_SSL      | true    |
|                 | service                         |                        |         |
| fail_on_error   | Should an error http request    | ZEALOT_FAIL_ON_ERROR   | false   |
|                 | cause a failure? (true/false)   |                        |         |
+-----------------+---------------------------------+------------------------+---------+
  
GitHub Workflow Multiplatform example  


```
name: Build Flutter Multiplatform and Upload to Sheepod

on:
  workflow_dispatch:

env:
  FLUTTER_VERSION: "3.24.0"
  APP_NAME: "sheepod_test_app" 
  
  SHEEPOD_ENDPOINT: ${{ secrets.SHEEPOD_ENDPOINT }}
  SHEEPOD_TOKEN: ${{ secrets.SHEEPOD_TOKEN }}
  SHEEPOD_CHANNEL_KEY_ANDROID: ${{ secrets.SHEEPOD_CHANNEL_KEY_ANDROID }}
  SHEEPOD_CHANNEL_KEY_LINUX: ${{ secrets.SHEEPOD_CHANNEL_KEY_LINUX }}
  SHEEPOD_CHANNEL_KEY_ZIP: ${{ secrets.SHEEPOD_CHANNEL_KEY_ZIP }}
  SHEEPOD_CHANNEL_KEY_WINDOWS: ${{ secrets.SHEEPOD_CHANNEL_KEY_WINDOWS }}
  SHEEPOD_CHANNEL_KEY_IOS: ${{ secrets.SHEEPOD_CHANNEL_KEY_IOS }}

jobs:
  # ==========================================
  # 1. BUILD ANDROID & LINUX (Ubuntu)
  # ==========================================
  build-ubuntu:
    name: Build Android & Linux
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
          
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Install Linux Build Deps
        run: |
          sudo apt-get update
          sudo apt-get install -y libgtk-3-dev libblkid-dev liblzma-dev libgcrypt20-dev pkg-config

      - name: Generate Platform Folders
        run: flutter create --project-name=${{ env.APP_NAME }} --platforms=android,linux .

      - name: Flutter Pub Get
        run: flutter pub get

      - name: Build Android APK
        run: flutter build apk --release --split-per-abi

      - name: Build Linux & Package (Tar + Zip)
        run: |
          flutter build linux --release
          cd build/linux/x64/release/bundle
          tar -czf ../../../../../${{ env.APP_NAME }}.tar.gz .
          zip -r ../../../../../${{ env.APP_NAME }}.zip .

      - name: Upload Android
        run: |
          APK_FILE=$(find build/app/outputs/flutter-apk -name "*arm64-v8a-release.apk" | head -n 1)
          if [ -n "$APK_FILE" ] && [ -f "$APK_FILE" ]; then
            curl -fL -X POST "${SHEEPOD_ENDPOINT}/api/apps/upload" \
              -F "token=${SHEEPOD_TOKEN}" \
              -F "channel_key=${SHEEPOD_CHANNEL_KEY_ANDROID}" \
              -F "file=@$APK_FILE" \
              -F "release_type=release" \
              -F "release_version=1.0.0" \
              -F "build_version=${GITHUB_RUN_NUMBER}" \
              -F "changelog=Build Android #${GITHUB_RUN_NUMBER}"
          fi

      - name: Upload Linux (tar.gz)
        run: |
          if [ -f "${{ env.APP_NAME }}.tar.gz" ]; then
            curl -fL -X POST "${SHEEPOD_ENDPOINT}/api/apps/upload" \
              -F "token=${SHEEPOD_TOKEN}" \
              -F "channel_key=${SHEEPOD_CHANNEL_KEY_LINUX}" \
              -F "file=@${{ env.APP_NAME }}.tar.gz" \
              -F "release_type=release" \
              -F "release_version=1.0.0" \
              -F "build_version=${GITHUB_RUN_NUMBER}" \
              -F "changelog=Build Linux #${GITHUB_RUN_NUMBER}"
          fi

      - name: Upload Linux Zip to ZIP Channel
        run: |
          if [ -f "${{ env.APP_NAME }}.zip" ]; then
            curl -fL -X POST "${SHEEPOD_ENDPOINT}/api/apps/upload" \
              -F "token=${SHEEPOD_TOKEN}" \
              -F "channel_key=${SHEEPOD_CHANNEL_KEY_ZIP}" \
              -F "file=@${{ env.APP_NAME }}.zip" \
              -F "release_type=release" \
              -F "release_version=1.0.0" \
              -F "build_version=${GITHUB_RUN_NUMBER}" \
              -F "changelog=Build Linux ZIP #${GITHUB_RUN_NUMBER}"
          fi

  # ==========================================
  # 2. BUILD WINDOWS (Windows Server)
  # ==========================================
  build-windows:
    name: Build Windows
    runs-on: windows-latest
    defaults:
      run:
        shell: bash
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Generate Platform Folders
        run: flutter create --project-name=${{ env.APP_NAME }} --platforms=windows .

      - name: Flutter Pub Get
        run: flutter pub get

      - name: Build Windows (Native EXE)
        run: flutter build windows --release

      - name: Upload Windows
        run: |
          EXE_FILE="build/windows/x64/runner/Release/${{ env.APP_NAME }}.exe"
          if [ -f "$EXE_FILE" ]; then
            curl -fL -X POST "${SHEEPOD_ENDPOINT}/api/apps/upload" \
              -F "token=${SHEEPOD_TOKEN}" \
              -F "channel_key=${SHEEPOD_CHANNEL_KEY_WINDOWS}" \
              -F "file=@$EXE_FILE" \
              -F "release_type=release" \
              -F "release_version=1.0.0" \
              -F "build_version=${GITHUB_RUN_NUMBER}" \
              -F "changelog=Build Windows #${GITHUB_RUN_NUMBER}"
          else
            echo "Error: .exe file not found!" && exit 1
          fi

  # ==========================================
  # 3. BUILD iOS (macOS) *Simulates ipa packaging without codesign utizing Zip
  # ==========================================
  build-ios:
    name: Build iOS
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Generate Platform Folders
        run: flutter create --project-name=${{ env.APP_NAME }} --platforms=ios .

      - name: Flutter Pub Get
        run: flutter pub get

      - name: Build iOS (No Codesign)
        run: flutter build ios --release --no-codesign

      - name: Package into Real IPA
        run: |
          cd build/ios/iphoneos
          mkdir Payload
          mv Runner.app Payload/
          zip -r ../../../${{ env.APP_NAME }}.ipa Payload

      - name: Upload iOS IPA
        run: |
          IPA_FILE="${{ env.APP_NAME }}.ipa"
          if [ -f "$IPA_FILE" ]; then
            curl -fL -X POST "${SHEEPOD_ENDPOINT}/api/apps/upload" \
              -F "token=${SHEEPOD_TOKEN}" \
              -F "channel_key=${SHEEPOD_CHANNEL_KEY_IOS}" \
              -F "file=@$IPA_FILE" \
              -F "release_type=release" \
              -F "release_version=1.0.0" \
              -F "build_version=${GITHUB_RUN_NUMBER}" \
              -F "changelog=Build iOS #${GITHUB_RUN_NUMBER}"
          else
            echo "Error: IPA file not found!" && exit 1
          fi
```
