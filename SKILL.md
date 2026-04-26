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
