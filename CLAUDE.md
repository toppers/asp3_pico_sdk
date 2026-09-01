# CLAUDE.md — asp3_pico_sdk

TOPPERS/ASP3 Core を Raspberry Pi Pico SDK と協調動作させる **SDK統合リポジトリ**。
純カーネル（`asp3_core`）を submodule 参照し、pico 固有部だけを本リポジトリに持つ。

> 本ファイルは別マシン（**PICO2 実機が接続された PC**）で作業する Claude 向けの作業指示書。
> 設計の全体像・経緯は submodule 側 `asp3/asp3_core/docs/dev/pico-sdk-integration.md` が正本。

---

## 0. リポジトリ構成

```
asp3_pico_sdk/
├── asp3/                            ← ASP3移植部（協調ヘルパ＋カーネル submodule＋SDK target）
│   ├── asp3_pico_sdk.cmake          ← 協調ヘルパ（PICO_PLATFORM→ASP3_TARGET/ASP3_TARGET_DIR・irq_* の --wrap）
│   ├── asp3_core/                   ← submodule（純カーネル＋全アーキ/チップ依存部 arch/）※public
│   └── target/                      ← SDK ターゲット依存部（チップ arch は asp3_core を共用）
│       ├── pico2_arm_sdk_gcc/       ← ARM-S（chip.cmake を asp3_core から直接 include）
│       └── pico2_riscv_sdk_gcc/     ← RISC-V（拡張toolchain(newlib)で実機 task1+RX 確認済）
└── sample1/                         ← アプリ（ARM/RISC-V 共通・timer_check 同梱）
```

> sample1 は ISA で分けず **PICO_PLATFORM で切替**（VS Code Pico 拡張のボード切替／CLI は -DPICO_PLATFORM）。
> ソース・cfg は ISA 非依存（差は `asp3_pico_sdk.cmake` の分岐と main の割込み禁止命令 `#if __ARM_ARCH` のみ）。
> arch（共通＋チップ）は ARM/RISC-V とも asp3_core 側を共用：SDK 側の arch 重複は全廃。
> SDK 固有なのは target 依存部（serial=pico-sdk stdio・timer・kernel_impl）のみ。
> **ツールチェーン**：ARM/RISC-V とも Pico SDK 拡張同梱（RISC-V は拡張の RISC-V toolchain＝newlib）を前提。

- ビルドは `asp3_core` の正準 CMakeLists を **ライブラリ専用モード**（`ASP3_LIBRARY_ONLY=ON`）で
  `add_subdirectory` し、最終 ELF は本リポジトリ側でリンクする（fork CMake は廃止済み）。
- 共存の要：pico-sdk の割込み登録API（`irq_*`/`exception_*` 8関数）を **リンカ `--wrap` で
  ASP3 管理へ誘導**＋`PICO_RUNTIME_SKIP_INIT_*` で SDK のベクタ/IRQ優先度初期化を抑止
  （ASP3 が NVIC を掌握）。実装は `asp3_pico_sdk.cmake` の `asp3_set_pico_sdk_options()`。

## 1. ⚠️ 禁則（作業前に必読）

1. **`asp3/asp3_core/`（submodule）配下を直接編集しない**。カーネル本体は上流 ASP3 追従領域。
   変更が必要なら asp3_core リポジトリ側で行い、その `AGENTS.md` の規約（`kernel/`・`include/`・
   `library/` 編集禁止、変更は `target/`・`syssvc/`・新規ファイルに限定）に従う。
   本リポジトリでの作業は **SDK 側ファイル（`asp3/target/`・`asp3/asp3_pico_sdk.cmake`・`sample1/`）**
   に閉じるのが原則（arch はすべて asp3_core 側）。
2. **カーネル内で動的メモリ確保を使わない**（`malloc`/`new` 等禁止。静的生成のみ。ASP3 安全設計方針）。

## 2. 取得・ビルド・実機確認（基本フロー）

```bash
# 取得（asp3_core は public なので匿名で submodule 取得可）
git clone --recurse-submodules https://github.com/toppers/asp3_pico_sdk.git
cd asp3_pico_sdk
# 既存clone: git submodule update --init --recursive

export PICO_SDK_PATH=/path/to/pico-sdk          # 2.1.1 / 2.2.0
cd sample1

# 推奨：VS Code Pico 拡張で sample1/ を開き、ボード/プラットフォームを選んで Build/Debug．
#       ARM↔RISC-V の切替は拡張の「Switch Platform」で行う（ツールチェーンも拡張同梱を使用）．

# コマンドライン（ISA ごとに別ビルドディレクトリ。同一 build/ では ISA 切替不可）：
# ARM-S（pico2 既定。-D 省略時も arm-s）
cmake -S . -B build_arm   -DPICO_PLATFORM=rp2350-arm-s -DPICO_BOARD=pico2 && cmake --build build_arm -j
# RISC-V / Hazard3（拡張同梱の RISC-V toolchain＝newlib が PATH/PICO_TOOLCHAIN_PATH にある前提）
cmake -S . -B build_riscv -DPICO_PLATFORM=rp2350-riscv -DPICO_BOARD=pico2 && cmake --build build_riscv -j
# 生成物: build_*/sample1_pico_sdk.uf2 / .elf / .bin
```

書き込み：BOOTSEL を押しながら USB 接続 →`RPI-RP2` に `.uf2` をコピー（または
`picotool load -x build/sample1_pico_sdk.uf2`、SWD は openocd）：
- ARM-S: `openocd -f interface/cmsis-dap.cfg -f target/rp2350.cfg       -c "adapter speed 5000" -c "program build/sample1_pico_sdk.elf verify reset exit"`
- RISC-V: `openocd -f interface/cmsis-dap.cfg -f target/rp2350-riscv.cfg -c "adapter speed 5000" -c "program build/sample1_pico_sdk.elf verify reset exit"`

シリアル確認：**UART0・GP0(TX)/GP1(RX)・115200 8N1**（USB stdio は無効）。
`minicom -D /dev/ttyACM0 -b 115200`（Debugprobe）等で
`TOPPERS/ASP3 Kernel ...` → `task1 is running (NNN).` の周期出力が出れば基本動作OK。

> 前提ツール：pico-sdk **2.1.1 / 2.2.0**（両方でビルド確認済。2.2.0 は `pico_platform_common/include` の
>   追加が必要で対応済）/ Pico SDK 拡張同梱ツールチェーン（ARM=14_2_Rel1、RISC-V=拡張の RISC-V toolchain＝newlib）/
>   cmake≥3.13 / python3 / picotool。
> 既知の検証実績（**ARM-S・RISC-V とも拡張同梱ツールチェーンで PICO2 実機 task1＋シリアル RX を確認済**）：
>  - ARM-S：拡張 ARM toolchain(14_2_Rel1)＋拡張 SDK でビルド確認（システム arm-none-eabi-gcc でも可）。実機 task1＋RX 済。
>  - RISC-V：拡張 RISC-V toolchain(RISCV_RPI_2_0_0_5＝newlib)＋拡張 SDK でビルド確認、実機 task1＋RX 済
>    （`a`→`#act_tsk(1)`、`2a`→`#act_tsk(2)`）。Ubuntu システム picolibc 用の回避策は不要（全削除済）。
>  - RISC-V の RX は ASP3 ネイティブ ISR＝target_serial.cfg の CRE_ISR で受ける（pico-sdk の irq 登録は
>    mtvec 解釈非互換のため使えない）。経緯は `asp3/asp3_core/docs/dev/pico-sdk-integration.md`。
> ※ VS Code 拡張は「Switch Platform」で本ファイル先頭の `set(toolchainVersion …)` を 14_2_Rel1↔RISCV_RPI_2_0_0_5 に
>   書換え、`pico-vscode.cmake` がそれを見て RISC-V 時は PICO_PLATFORM=rp2350-riscv を自動設定する（ISA 切替の実体）。

## 検証の鉄則

- コードを変更したら **必ずビルドを通してから報告**。「動くはず」で報告しない。
- 実機確認はシリアル出力（`task1 is running`・周期）を根拠とする。
- 変更後、進捗・結果を `asp3/asp3_core/docs/dev/pico-sdk-integration.md` の「未完（後続）」に追記する
  （asp3_core は別リポジトリ。push 権限が無ければ差分を提示してユーザーに push を依頼）。

---

# タスク（完了済み・記録）

当初の依頼タスクは**すべて完了**（再実施不要）。経緯・結果は docs/ に集約済み：

| タスク | 結果 |
|---|---|
| タスク1：タイマ競合の検証と調停 | **完了**。ALARM0(ASP3) vs ALARM3(SDK既定pool) で競合なし・PICO2 実測 +6ppm。詳細とアプリ制約は [docs/timer-coexistence.md](docs/timer-coexistence.md) |
| タスク2：RISC-V（Hazard3）版の追加 | **完了**。`rp2350-riscv` 対応済み・実機 task1＋RX 確認済み（RX は ASP3 ネイティブ ISR＝mtvec 非互換のため）。設計は [docs/design.md](docs/design.md) §2 |

残課題は [docs/history.md](docs/history.md) の「残課題」を参照
（RISC-V の timer_check 定量検証・OS Awareness 対応など）。

---

## 設計・経緯の記録

- [docs/design.md](docs/design.md) — 割込み共存（ARM `--wrap`／RISC-V ネイティブISR）・ツールチェーン・pico-sdk 互換
- [docs/timer-coexistence.md](docs/timer-coexistence.md) — タイマ競合の解析・PICO2 実測・アプリ制約
- [docs/history.md](docs/history.md) — リポジトリ再編の経緯・検証マトリクス・残課題

## 参考リポジトリ

| リポジトリ | 用途 |
|---|---|
| `asp3_core`（submodule・public） | 純カーネル。`docs/dev/pico-sdk-integration.md`＝統合の正本。`docs/porting/PORTING_GUIDE.md`「外部ターゲット」節＝パス規約 |
| `asp3_pico_sdk_legacy`（archived） | 旧・カーネル同梱fork。移植元の参照点（読取専用） |
