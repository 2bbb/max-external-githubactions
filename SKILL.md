# max-external-githubactions

Max/MSP external (min-api) プロジェクト用 GitHub Actions ワークフロー。

## 使い方

プロジェクトの `.github/workflows/build.yml` に以下を配置:

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

## 前提

- `deps/min-api/` に min-api が submodule として配置されていること
- `cmake/bbb_external.cmake` で `bbb_add_external()` が定義されていること
- macOS 専用 external は `bbb_add_external(MACOS_ONLY ...)` でガードすること

## 生成物

- macOS: `externals/*.mxo` (universal binary: x86_64 + arm64)
- Windows: `externals/*.mxe64` (x64)

各プラットフォームの artifact として GitHub Actions からダウンロード可能。

## 注意点

### submodule は public リポジトリにすること

`submodules: recursive` で CI 上にチェックアウトするため、
依存する submodule リポジトリはすべて public に設定すること。
private のまま使う場合は deploy key の設定が必要。

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
