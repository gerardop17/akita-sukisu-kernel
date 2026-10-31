# Pixel 8a (akita) SukiSU Ultra + SUSFS kernel — 6.1.145 / latest SukiSU

Google Pixel 8a (akita / Tensor G3) 向けの **GKI 6.1.145** カーネルに
**SukiSU Ultra (最新)** と **SUSFS** を組み込んで GitHub Actions でビルドする構成。

[frost-1256/android-kernels](https://github.com/frost-1256/android-kernels) の
`akita_sukisu` 構成（現在端末で使用中の `akita-sukisu-5` のビルド元）を
このリポジトリに移植し、SukiSU / SUSFS の固定コミットを最新に更新したもの。
ローカルでのダウンロードは不要で、カーネルソース・ツールチェーン等の
ダウンロードからビルド・Release 作成まで **すべて Actions 上で自動** で行われる。

## 対象 / ビルド構成

| 項目 | 値 |
|---|---|
| 端末 | Pixel 8a (`akita`)、現在使用中であることを確認済み |
| ROM | Android 16 系 (stock / GKI 2.0)、fingerprint `BP4A.260205.001` で確認 |
| カーネル | ACK `android14-6.1-2025-09` = **6.1.145-android14-11**（現在使用中と同一） |
| root | SukiSU Ultra `builtin` ブランチ、`b16e899a555e9b96aa1225f049cd7274820ca5b8`（2026-10-29 時点の先頭）に固定 |
| hiding | susfs4ksu (ShirkNeko) `gki-android14-6.1`、`ee155a1ad4219804a7c54cb39dd4fa1b5c2a3fa4`（2026-10-29 時点の先頭）に固定 |
| KPM | off（従来通り） |
| 出力 | `boot.img` / `boot-avb.img` / AnyKernel3 zip / `Image` / `.config` / `build-info.txt` |

> **注意:** これは GKI（共通カーネル）のみを差し替える構成。
> Pixel 8a の `vendor_dlkm` / `system_dlkm` / `vendor_kernel_boot` / `dtbo` は
> ストックのまま使う。

### 2025-09 と 2026-09 はどちらがよいか

- `android14-6.1-2025-09` = 6.1.145。**現在端末で実際に動いているカーネルと同一**で、
  変更点が SukiSU / SUSFS の更新だけに絞れる。まずはこちらが安全（採用）。
- `android14-6.1-2026-09` = 6.1.176 級。新しい LTS 修正・セキュリティ修正が入る一方、
  端末の vendor モジュールとの組み合わせでは未検証で、起動しない可能性が上がる。
  試す場合は別途 Run workflow で `kernel_ref` を上書きして実験用にビルドし、
  6.1.145 版を復旧用に残しておくこと。

## ビルド

- **自動:** `akita_sukisu/**` またはこのワークフローへの push で自動起動し、
  Release（タグ `akita-sukisu-<run_number>`）まで作成される。
- **手動:** GitHub の **Actions → Build Pixel 8a (akita) SukiSU Ultra + SUSFS → Run workflow**。

主な入力（既定値が今回の構成）:

- `kernel_ref`: `android14-6.1-2025-09`（6.1.145）
- `sukisu_ref`: **`builtin` のまま**。SUSFS は `builtin` (manual hook) 前提で、`main` (kprobe) では噛み合わない
- `sukisu_commit` / `susfs_commit`: 上記の最新コミットに固定。空欄にしても同じ固定が使われる。
  将来さらに新しい先頭で組みたい場合は、その時点の先頭 SHA を入力する
- `kpm`: 通常は off
- `make_release`: on なら Release に添付

## 書き込み

前提: bootloader unlock 済み。**必ず現在の boot をバックアップ**する。

```sh
fastboot devices
fastboot flash boot akita-sukisu-susfs-<ver>-boot.img
fastboot reboot
```

起動確認:

```sh
adb shell uname -a
adb shell su -c id
```

SukiSU Ultra の Manager は最新の APK（v4.3.0 以降）をインストールする。
カーネル側のバージョンが上がっているため、旧 Manager (v4.2.1) のままだと
一部機能が表示・動作しない場合がある。

## 復旧

ストックの `boot.img` を同じスロットに焼き戻す。

```sh
fastboot flash boot stock-boot.img
fastboot reboot
```

## 仕組み / 注意点

- `build.sh` はカーネルソース (ACK)、AOSP clang、SukiSU Ultra、SUSFS、
  mkbootimg、AnyKernel3 をすべて自動ダウンロードする。
- SUSFS は `50_add_susfs_in_gki-android14-6.1.patch` を適用（6.1 系向けの
  適用調整ロジック含む）。`android/abi_gki_protected_exports_*` を削除する
  （Wi-Fi などの vendor モジュール対策）。
- AVB footer は AOSP test key。unlocked bootloader 前提で、ロック状態では
  検証を通らない。
