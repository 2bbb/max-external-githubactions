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
          echo "$DEPLOY_KEY" | base64 -D > ~/.ssh/deploy_key  # macOS (BSD). Linux: base64 -d
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
