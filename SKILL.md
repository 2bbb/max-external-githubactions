---
name: max-external-githubactions
description: Create and maintain GitHub Actions workflows for Max/MSP min-api external projects, including macOS universal builds, Windows mxe64 builds, release packaging, Developer ID signing, Apple notarization, Gatekeeper-safe downloads, and private submodule CI setup.
---

# max-external-githubactions

Max/MSP external (min-api) プロジェクト用 GitHub Actions ワークフロー。

## 3段階構成

プロジェクトの成熟度に合わせて段階的に追加できる:

| 段階 | 内容 | トリガー |
|---|---|---|
| **1. Build only** | ビルド確認 + artifact アップロード | push / PR |
| **2. Build + Package** | 配布用zipをartifactにアップロード | push / PR |
| **3. Build + Package + Release** | Release 発行時にzipをassetsに添付 | push / PR / release published |

---

## 1. Build only (最小構成)

ビルドが通ることだけ確認する。プロジェクト初期に推奨。

```yaml
name: Build

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-macos:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Configure and Build
        run: cmake -B build && cmake --build build --config Release

      - name: Upload externals
        uses: actions/upload-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-macos
          path: externals/

  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Configure and Build
        run: cmake -B build -G "Visual Studio 17 2022" -A x64 && cmake --build build --config Release

      - name: Upload externals
        uses: actions/upload-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-windows
          path: externals/
```

---

## 2. Build + Package

両プラットフォームのビルド結果を所定のディレクトリ構造にまとめ、
プラットフォーム別 zip として artifact にアップロードする。

### 配布パッケージ構成

```
<package-name>/
├── externals/          # .mxo (macOS) / .mxe64 (Windows)
├── help/               # .maxhelp ファイル (あれば)
├── extra/              # 追加ファイル (あれば)
├── patchers/           # デモパッチ等 (あれば)
├── package-info.json
├── LICENSE
└── README.md
```

### ワークフロー

```yaml
name: Build & Package

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-macos:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Configure and Build
        run: cmake -B build && cmake --build build --config Release

      - name: Upload externals
        uses: actions/upload-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-macos
          path: externals/

  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Configure and Build
        run: cmake -B build -G "Visual Studio 17 2022" -A x64 && cmake --build build --config Release

      - name: Upload externals
        uses: actions/upload-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-windows
          path: externals/

  package:
    needs: [build-macos, build-windows]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download macOS externals
        uses: actions/download-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-macos
          path: dist/${{ github.event.repository.name }}/externals/

      - name: Download Windows externals
        uses: actions/download-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-windows
          path: dist/${{ github.event.repository.name }}/externals/

      - name: Assemble package
        run: |
          mkdir -p "dist/${{ github.event.repository.name }}/"
          cp package-info.json "dist/${{ github.event.repository.name }}/" 2>/dev/null || true
          cp LICENSE "dist/${{ github.event.repository.name }}/" 2>/dev/null || true
          cp README.md "dist/${{ github.event.repository.name }}/" 2>/dev/null || true
          cp -r help "dist/${{ github.event.repository.name }}/" 2>/dev/null || true
          cp -r extra "dist/${{ github.event.repository.name }}/" 2>/dev/null || true
          cp -r patchers "dist/${{ github.event.repository.name }}/" 2>/dev/null || true

      - name: Create archive
        run: |
          cd dist
          zip -r "${{ github.event.repository.name }}.zip" "${{ github.event.repository.name }}/"

      # 注意: Max external (.mxo) bundle をユーザーに配る zip は、可能なら macOS runner 上で
      # ditto -c -k --keepParent によって作ること。Ubuntu の zip は通常 artifact 用としては十分だが、
      # macOS bundle metadata/permission の扱いが原因で Gatekeeper/Max ロード時の挙動が変わり得る。
      # Gatekeeper 対応が必要な配布物は、下の「macOS Gatekeeper 対応」セクションの通り
      # valid bundle identifier + codesign + notarytool + ditto を使うこと。

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-latest
          path: dist/*.zip
```

`${{ github.event.repository.name }}` でリポジトリ名が自動適用される。
手動でのプレースホルダ置換は不要。

1つの zip に macOS (.mxo) と Windows (.mxe64) の両方を含める。
Max は実行環境に合わない拡張子を自動で無視するため、混在していても問題ない。

---

## 3. Build + Package + Release

Release 発行時に自動的にパッケージをビルドして assets に添付する。

段階 2 のワークフローに以下を追加:

### トリガーに release を追加

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  release:
    types: [published]
```

### package job に permissions と Release upload step を追加

```yaml
  package:
    permissions:
      contents: write
    needs: [build-macos, build-windows]
    runs-on: ubuntu-latest
    steps:
      # ... (assemble, create archive steps は段階2と同じ)

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-latest
          path: dist/*.zip

      - name: Upload to Release
        if: github.event_name == 'release'
        uses: softprops/action-gh-release@v2
        with:
          files: dist/*.zip
```

### リリース手順

1. タグを打つ: `git tag v1.0.0 && git push --tags`
2. GitHub 上で Release を作成 (タグを指定)
3. CI が自動ビルド → zip を Release assets に添付

---

## macOS Gatekeeper 対応 (Developer ID 署名 + notarization)

ユーザーがダウンロードした `.mxo` で
「Apple はマルウェアが含まれていないことを検証できませんでした」系の警告が出る場合、
原因は **Developer ID 署名 + notarization 不足** だけに限らない。
実運用では以下も警告・ロード失敗・不安定挙動の原因になる。

- `CFBundleIdentifier` / signing identifier が空、`.`、`com.acme...`、未置換 `${PRODUCT_NAME...}` などの壊れた値
- Ubuntu/Linux の `zip -r` で macOS bundle metadata や permission の扱いが雑になった archive
- `.mxo/Contents/MacOS/*` に実行 permission が無い、または bundle 構造が崩れている
- quarantine/provenance の付き方や展開経路の違い

`xattr -dr com.apple.quarantine` はローカル開発用の逃げであって、Release の根本解決ではない。
ただし、secret を使わない内部向け CI では unsigned 配布を許容する判断もあり得る。その場合は Gatekeeper-safe と呼ばない。

判断境界:

- **Apple の検証警告を public download で消す必要がある場合だけ**、Developer ID 署名 + notarization を導入する。
- ユーザーが notarization は不要、または workflow 変更は不要と言った場合、CI/release workflow に署名・notarization・secret 必須化を追加しない。
- secrets が無いことだけを理由に main の package/release 更新を失敗させる変更を勝手に入れない。これは配布方針の変更であり、明示要求が必要。
- secret-free CI では missing secrets は `::notice::` に留め、署名・notarization step だけを skip する。
- `xattr -dr com.apple.quarantine` はユーザーのローカル回避策であって、agent から提案・実行するのは明示要求がある時だけ。

必要なもの:

- **public download で Apple の検証警告を消す**: Developer ID Application 証明書、Apple notarization 用認証情報、macOS runner で作った notarize 済み zip が必要。
- **notarization しない配布を続ける**: 有効な `jp.2bit.*` bundle identifier、ad-hoc codesign、実行権限、macOS `ditto` zip が最低ライン。ただし Apple の検証警告は仕様として残る。
- **ユーザーの手元で開くだけ**: macOS の「開く」/「このまま開く」等のユーザー承認が必要。repo/CI 側の修正ではない。

結論:

- 最低限、Release に載せる macOS `.mxo` は valid な `jp.2bit.*` bundle identifier を持たせる。
- macOS bundle/リソースを含む archive は macOS runner の `ditto -c -k --keepParent` で作る。
- `.mxo/Contents/MacOS/*` は package 前に `chmod 755` で正規化する。
- Apple 検証済みにしたい場合だけ `.mxo` を **Developer ID Application** 証明書で署名し、Release archive を `xcrun notarytool submit --wait` で notarize する。
- 配布する zip は **notarytool に投げたそのもの** にする。notarize 後に Ubuntu などで zip を作り直すな。
- Bundle identifier / signing identifier は `jp.2bit.*` を使う。例: `jp.2bit.bbb.artnet.controller`。
- ただし、このセクションは「Gatekeeper-safe release を要求された時だけ」の実装手順であり、通常の build/package workflow に自動適用しない。

### Secret-free CI の現実解

GitHub Secrets を入れない方針の場合、CI は失敗させずに unsigned package を作ってよい。
ただし以下を守る。

- missing secrets を `::error::` にして main/release を落とさない。`::notice::` に留める。
- signing / notarization step は credentials が揃っている時だけ実行する。
- unsigned package を Gatekeeper-safe と書かない。
- それでも `jp.2bit.*` bundle identifier、macOS `ditto` zip、`chmod 755` は維持する。

この設計は「警告が絶対に出ない」保証ではない。目的は、壊れた bundle / 雑な zip による余計な警告を減らしつつ、secret 無しでも CI artifact/latest release を継続生成すること。

### GitHub Secrets

Developer ID signing / notarization を CI で行う repository には以下を登録する。
secret-free CI の場合は登録不要だが、その場合の zip は unsigned / non-notarized として扱う。

| Secret | 内容 |
|---|---|
| `MACOS_CERTIFICATE_P12` | Developer ID Application 証明書を書き出した `.p12` の base64 |
| `MACOS_CERTIFICATE_PASSWORD` | `.p12` のパスワード |
| `APPLE_ID` | Apple Developer アカウントの Apple ID |
| `APPLE_TEAM_ID` | Team ID |
| `APPLE_APP_SPECIFIC_PASSWORD` | notarization 用 App-specific password |
| `DEVELOPER_ID_APPLICATION` | `Developer ID Application: ... (TEAMID)` の署名名 |

`.p12` はローカルで base64 化して登録する。

```bash
base64 -i developer-id-application.p12 | pbcopy
```

### 署名 identifier の規則

`jp.2bit.*` 以外を使わない。プロジェクト名・external 名から以下のように作る。

```bash
BUNDLE_ID_PREFIX="jp.2bit"
EXTERNAL_NAME="bbb.artnet.controller"
SIGNING_IDENTIFIER="${BUNDLE_ID_PREFIX}.${EXTERNAL_NAME}"
# => jp.2bit.bbb.artnet.controller
```

external 名に identifier として不正な文字が混ざる場合だけ、英数字・`.`・`-` 以外を `-` に潰す。

```bash
sanitize_identifier_component() {
  printf '%s' "$1" | tr '[:upper:]' '[:lower:]' | sed -E 's/[^a-z0-9.-]+/-/g; s/^-+//; s/-+$//'
}
```

CMake 側で bundle identifier を指定できる場合も `jp.2bit.*` に揃える。テンプレート由来の
`com.acme...` や未置換 `${PRODUCT_NAME...}` を Release に混ぜるな。

```cmake
set_target_properties(<target> PROPERTIES
  XCODE_ATTRIBUTE_PRODUCT_BUNDLE_IDENTIFIER "jp.2bit.<external-name>"
  MACOSX_BUNDLE_GUI_IDENTIFIER "jp.2bit.<external-name>"
)
```

### macOS build job への署名 step

Release/tag/main で Developer ID の secrets が揃っている時だけ秘密鍵を import し、`.mxo` bundle を署名する。
PR、特に fork 由来 PR で秘密情報を使う設計にするな。
secret-free CI を許容する場合は、missing secrets で job を失敗させず署名 step だけ skip する。

```yaml
  build-macos:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Configure and Build
        run: cmake -B build && cmake --build build --config Release

      - name: Check macOS signing secrets
        id: macos_signing
        if: github.event_name == 'release' || startsWith(github.ref, 'refs/tags/') || (github.event_name == 'push' && github.ref == 'refs/heads/main')
        shell: bash
        env:
          MACOS_CERTIFICATE_P12: ${{ secrets.MACOS_CERTIFICATE_P12 }}
          MACOS_CERTIFICATE_PASSWORD: ${{ secrets.MACOS_CERTIFICATE_PASSWORD }}
          DEVELOPER_ID_APPLICATION: ${{ secrets.DEVELOPER_ID_APPLICATION }}
        run: |
          set -euo pipefail
          missing=0
          for name in MACOS_CERTIFICATE_P12 MACOS_CERTIFICATE_PASSWORD DEVELOPER_ID_APPLICATION; do
            if [ -z "${!name:-}" ]; then
              echo "::notice::$name is not set; macOS externals will be uploaded unsigned"
              missing=1
            fi
          done
          if [ "$missing" -eq 0 ]; then
            echo "available=true" >> "$GITHUB_OUTPUT"
          else
            echo "available=false" >> "$GITHUB_OUTPUT"
          fi

      - name: Import Developer ID certificate
        if: steps.macos_signing.outputs.available == 'true'
        shell: bash
        env:
          CERTIFICATE_P12: ${{ secrets.MACOS_CERTIFICATE_P12 }}
          CERTIFICATE_PASSWORD: ${{ secrets.MACOS_CERTIFICATE_PASSWORD }}
          KEYCHAIN_PASSWORD: ${{ github.run_id }}-${{ github.run_attempt }}
        run: |
          CERT_PATH="$RUNNER_TEMP/developer-id.p12"
          KEYCHAIN_PATH="$RUNNER_TEMP/app-signing.keychain-db"
          printf '%s' "$CERTIFICATE_P12" | base64 --decode > "$CERT_PATH"
          security create-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"
          security set-keychain-settings -lut 21600 "$KEYCHAIN_PATH"
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"
          security import "$CERT_PATH" -P "$CERTIFICATE_PASSWORD" -A -t cert -f pkcs12 -k "$KEYCHAIN_PATH"
          security list-keychains -d user -s "$KEYCHAIN_PATH" login.keychain-db
          security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"

      - name: Sign Max externals
        if: steps.macos_signing.outputs.available == 'true'
        shell: bash
        env:
          DEVELOPER_ID_APPLICATION: ${{ secrets.DEVELOPER_ID_APPLICATION }}
          BUNDLE_ID_PREFIX: jp.2bit
        run: |
          set -euo pipefail
          sanitize_identifier_component() {
            printf '%s' "$1" | tr '[:upper:]' '[:lower:]' | sed -E 's/[^a-z0-9.-]+/-/g; s/^-+//; s/-+$//'
          }
          find externals -name "*.mxo" -type d -print0 | while IFS= read -r -d '' mxo; do
            name="$(basename "$mxo" .mxo)"
            component="$(sanitize_identifier_component "$name")"
            identifier="${BUNDLE_ID_PREFIX}.${component}"
            codesign --force --deep --options runtime --timestamp \
              --identifier "$identifier" \
              --sign "$DEVELOPER_ID_APPLICATION" \
              "$mxo"
            codesign --verify --deep --strict --verbose=2 "$mxo"
          done

      - name: Upload externals
        uses: actions/upload-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-macos
          path: externals/
```

### notarize 済み Release archive を作る job

Gatekeeper 対応の Release asset を作る場合は、最終 archive 作成も notarization も macOS runner で行う。
Windows artifact を同梱したいなら macOS runner で Windows artifact も download してから `ditto` で固める。

```yaml
  package-release:
    if: github.event_name == 'release' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: write
    needs: [build-macos, build-windows]
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download macOS externals
        uses: actions/download-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-macos
          path: dist/${{ github.event.repository.name }}/externals/

      - name: Download Windows externals
        uses: actions/download-artifact@v4
        with:
          name: ${{ github.event.repository.name }}-windows
          path: dist/${{ github.event.repository.name }}/externals/

      - name: Assemble package
        shell: bash
        run: |
          set -euo pipefail
          pkg="dist/${{ github.event.repository.name }}"
          mkdir -p "$pkg"
          cp package-info.json "$pkg/" 2>/dev/null || true
          cp LICENSE "$pkg/" 2>/dev/null || true
          cp README.md "$pkg/" 2>/dev/null || true
          cp -R help "$pkg/" 2>/dev/null || true
          cp -R extra "$pkg/" 2>/dev/null || true
          cp -R patchers "$pkg/" 2>/dev/null || true
          find "$pkg/externals" -path '*/Contents/MacOS/*' -type f -exec chmod 755 {} + 2>/dev/null || true

      - name: Create macOS-preserving archive
        shell: bash
        run: |
          set -euo pipefail
          cd dist
          ditto -c -k --keepParent "${{ github.event.repository.name }}" "${{ github.event.repository.name }}.zip"

      - name: Check notarization secrets
        id: notarization
        if: github.event_name == 'release' || startsWith(github.ref, 'refs/tags/') || (github.event_name == 'push' && github.ref == 'refs/heads/main')
        shell: bash
        env:
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
        run: |
          set -euo pipefail
          missing=0
          for name in APPLE_ID APPLE_TEAM_ID APPLE_APP_SPECIFIC_PASSWORD; do
            if [ -z "${!name:-}" ]; then
              echo "::notice::$name is not set; package will be uploaded without notarization"
              missing=1
            fi
          done
          if [ "$missing" -eq 0 ]; then
            echo "available=true" >> "$GITHUB_OUTPUT"
          else
            echo "available=false" >> "$GITHUB_OUTPUT"
          fi

      - name: Notarize archive
        if: steps.notarization.outputs.available == 'true'
        shell: bash
        env:
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
        run: |
          set -euo pipefail
          xcrun notarytool submit "dist/${{ github.event.repository.name }}.zip" \
            --apple-id "$APPLE_ID" \
            --team-id "$APPLE_TEAM_ID" \
            --password "$APPLE_APP_SPECIFIC_PASSWORD" \
            --wait

      - name: Upload to Release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/${{ github.event.repository.name }}.zip
```

`stapler` は `.app` / `.pkg` / `.dmg` 向け。zip 配布では、notarize に通した zip をそのまま配る。
notarize 後に別 job で zip を作り直すと、ユーザー側では未 notarize 扱いになり得る。

### ローカル開発だけの回避策

ユーザーが明示的に求めた場合、かつ開発中に自分の Mac だけで検証したい場合のみ quarantine を消してよい。
Release 手順の代わりにしてはいけない。

```bash
xattr -dr com.apple.quarantine "/Users/$USER/Documents/Max 9/Packages/<package-name>"
```

---

## 前提

- `deps/min-api/` に min-api が submodule として配置されていること
- `cmake/bbb_external.cmake` で `bbb_add_external()` が定義されていること
- macOS 専用 external は `bbb_add_external(MACOS_ONLY ...)` でガードすること

## 生成物

- macOS: `externals/*.mxo` (universal binary: x86_64 + arm64)
- Windows: `externals/*.mxe64` (x64)

## 注意点

### submodule は public リポジトリにすること

`submodules: recursive` で CI 上にチェックアウトするため、
依存する submodule リポジトリはすべて public に設定すること。
private のまま使う場合は deploy key の設定が必要（下記「Private submodule の場合」を参照）。

### submodules: recursive が必須

min-api の中に max-sdk-base がネストした submodule として含まれている。
`--init` だけでは max-sdk-base が取得できず、`max-pretarget.cmake` が見つからないエラーになる。
必ず `submodules: recursive` を指定すること。

### bbb_add_external() の MACOS_ONLY / WIN32_ONLY

Windows 対応するには、macOS 専用 API を使う external をビルド対象から除外する必要がある。
max-external skill の `cmake/bbb_external.cmake` には既に `MACOS_ONLY` / `WIN32_ONLY` オプションが実装済み。
各 external の CMakeLists.txt で以下のように指定すること:

```cmake
# macOS 専用 (CommonCrypto, FSEvents, popen など)
bbb_add_external(MACOS_ONLY)

# Windows 専用
bbb_add_external(WIN32_ONLY)

# クロスプラットフォーム (オプションなし)
bbb_add_external()
```

### macOS 専用 API の例

以下の API/ヘッダは Windows に存在しないため `MACOS_ONLY` でガードすること:
- `CommonCrypto/CommonDigest.h` → CryptoAPI または OpenSSL で代替
- `CoreServices/CoreServices.h` → `ReadDirectoryChangesW` で代替
- `popen()` / `pclose()` → `_popen()` / `_pclose()` (ただし挙動が異なる)
- `<regex.h>` (POSIX) → C++11 `<regex>` で代替
- `<unistd.h>` → Windows に存在しない

---

## Private submodule の場合

private リポジトリを submodule として参照する場合、`actions/checkout` の
`submodules: recursive` だけでは認証エラーになる。SSH deploy key を使う。

### セットアップ手順

1. **SSH 鍵ペアを生成** (ローカル):

```bash
ssh-keygen -t ed25519 -C "ci-deploy-key" -f deploy_key -N ""
```

2. **公開鍵を private submodule リポジトリに登録**:
   - 対象リポジトリ → Settings → Deploy keys → Add deploy key
   - **Read-only** でよい（CI は clone だけする）
   - Title: `ci-deploy-key` 等

3. **秘密鍵をメインリポジトリの Secret に登録**:
   - メインリポジトリ → Settings → Secrets → New repository secret
   - Name: `<SUBMODULE>_DEPLOY_KEY` (例: `NOZZLE_DEPLOY_KEY`)
   - Value: base64 エンコードした秘密鍵

```bash
# macOS
base64 -i deploy_key | pbcopy

# Linux
base64 -w 0 deploy_key

# Windows (PowerShell)
[Convert]::ToBase64String([IO.File]::ReadAllBytes("deploy_key")) | Set-Clipboard
```

base64 にする理由: 改行や特殊文字が GitHub Secret の入力で壊れるのを防ぐため。

### ワークフロー例

```yaml
jobs:
  build-macos:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: false  # ← まずは自分だけ checkout

      - name: Setup SSH deploy key
        env:
          DEPLOY_KEY: ${{ secrets.NOZZLE_DEPLOY_KEY }}
        run: |
          mkdir -p ~/.ssh
          printf '%s' "$DEPLOY_KEY" | base64 -D > ~/.ssh/deploy_key  # macOS (BSD). Linux: base64 -d
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          cat >> ~/.ssh/config <<EOF
          Host github.com
            IdentityFile ~/.ssh/deploy_key
          EOF

      - name: Checkout submodules
        run: git submodule update --init --recursive

      # ... 通常のビルド手順 ...
```

### Windows の場合

Windows runner では `base64 -d` の代わりに PowerShell を使う:

```yaml
      - name: Setup SSH deploy key
        env:
          DEPLOY_KEY: ${{ secrets.NOZZLE_DEPLOY_KEY }}
        shell: pwsh
        run: |
          mkdir -Force ~/.ssh
          [System.IO.File]::WriteAllBytes("$HOME/.ssh/deploy_key", [System.Convert]::FromBase64String($env:DEPLOY_KEY))
          icacls "$HOME/.ssh/deploy_key" /inheritance:r /grant:r "$env:USERNAME:R"
          ssh-keyscan github.com | Out-File -Encoding ascii -Append $HOME/.ssh/known_hosts
          @"
          Host github.com
            IdentityFile ~/.ssh/deploy_key
          "@ | Out-File -Encoding ascii -Append $HOME/.ssh/config
```

### 注意

- 秘密鍵は **base64 エンコードして** Secret に登録すること（改行破損を防ぐ）
- `ssh-keyscan` でホスト鍵を `known_hosts` に追加し、`StrictHostKeyChecking no` は避けること
- `.gitmodules` の submodule URL は SSH 形式 (`git@github.com:owner/repo.git`) にすること。HTTPS 形式の場合は認証エラーになる（または `git config --global url."git@github.com:".insteadOf "https://github.com/"` で HTTPS を SSH に自動置換する設定を行うことも可能）
- private submodule が複数ある場合は、リポジトリごとに個別の鍵ペアを作成してください（GitHub では同じデプロイキーを複数のリポジトリに登録することはできません）
- 複数の private submodule がある場合は、`.gitmodules` でホストエイリアス（例: `sub1.github.com:owner/repo.git`）を設定し、SSH config でエイリアスごとに個別の鍵を紐付ける必要があります。同じ `Host github.com` に複数の `IdentityFile` を追加しても、GitHub が最初の鍵で拒否すると次の鍵を試行できません

### マルチ submodule 設定例

`.gitmodules`:
```ini
[submodule "sub1"]
  path = sub1
  url = git@sub1.github.com:owner/repo1.git

[submodule "sub2"]
  path = sub2
  url = git@sub2.github.com:owner/repo2.git
```

`~/.ssh/config` (CI 側):
```text
Host sub1.github.com
  HostName github.com
  IdentityFile ~/.ssh/deploy_key_sub1
  HostKeyAlias github.com
  IdentitiesOnly yes

Host sub2.github.com
  HostName github.com
  IdentityFile ~/.ssh/deploy_key_sub2
  HostKeyAlias github.com
  IdentitiesOnly yes
```
