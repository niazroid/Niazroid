name: GohstVpn Android APK

on:
  workflow_dispatch:
  push:
    branches:
      - main
      - master

permissions:
  contents: read

concurrency:
  group: gohstvpn-android-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-debug-apk:
    name: Build Android debug APK
    runs-on: ubuntu-24.04
    timeout-minutes: 90

    steps:
      - name: Checkout GohstVpn source
        uses: actions/checkout@v4

      # Optional: user-provided private repository containing the missing
      # Karing modules. See GOHSTVPN_GITHUB_BUILD.md for the expected layout.
      - name: Checkout optional private build parts
        if: ${{ vars.GOHSTVPN_PARTS_REPO != '' }}
        uses: actions/checkout@v4
        with:
          repository: ${{ vars.GOHSTVPN_PARTS_REPO }}
          token: ${{ secrets.GOHSTVPN_PARTS_TOKEN }}
          path: .build-parts
          persist-credentials: false

      - name: Restore optional private build parts
        shell: bash
        run: |
          set -euo pipefail
          if [[ -d .build-parts ]]; then
            for part in lib/app/utils lib/app/local_services lib/app/private android/libbox; do
              if [[ -d ".build-parts/$part" ]]; then
                mkdir -p "$(dirname "$part")"
                cp -a ".build-parts/$part" "$part"
              fi
            done
            if [[ -d .build-parts/vpn-service ]]; then
              cp -a .build-parts/vpn-service "${GITHUB_WORKSPACE}/../vpn-service"
            fi
          fi
          # Alternative for a complete repository: include vpn-service at root.
          if [[ -d vpn-service && ! -d "${GITHUB_WORKSPACE}/../vpn-service" ]]; then
            cp -a vpn-service "${GITHUB_WORKSPACE}/../vpn-service"
          fi

      - name: Check Karing source completeness
        shell: bash
        run: |
          set -euo pipefail
          missing=0
          for part in             lib/app/utils             lib/app/local_services             lib/app/private             android/libbox; do
            if [[ ! -d "$part" ]]; then
              echo "::error title=Missing source::Directory $part is absent from this Karing source archive."
              missing=1
            fi
          done
          if [[ ! -f "${GITHUB_WORKSPACE}/../vpn-service/pubspec.yaml" ]]; then
            echo "::error title=Missing vpn-service::Karing requires ../vpn-service/pubspec.yaml, which is not public in the supplied archive. Supply this dependency from an authorized source."
            missing=1
          fi
          if (( missing )); then
            echo 'The uploaded Karing ZIP is incomplete. See GOHSTVPN_GITHUB_BUILD.md.'
            exit 1
          fi

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          channel: stable
          flutter-version: '3.44.9'
          cache: true

      - name: Install Android SDK and NDK
        shell: bash
        run: |
          set -euo pipefail
          sdkmanager --install             'platforms;android-35'             'build-tools;36.0.0'             'ndk;28.2.13676358'

      - name: Check Flutter installation
        run: flutter --version

      - name: Restore Flutter packages
        run: flutter pub get

      - name: Build GohstVpn debug APK
        run: flutter build apk --debug

      - name: Prepare downloadable APK
        shell: bash
        run: |
          set -euo pipefail
          mkdir -p dist
          cp build/app/outputs/flutter-apk/app-debug.apk dist/GohstVpn-android-debug.apk
          cd dist
          sha256sum GohstVpn-android-debug.apk > GohstVpn-android-debug.apk.sha256

      - name: Upload APK to GitHub Actions Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: GohstVpn-debug-APK-${{ github.run_number }}
          path: dist/
          if-no-files-found: error
          retention-days: 30
