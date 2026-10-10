# OpenCore EFI — Lenovo IdeaCentre K450 (10120) / Core i7-4770 (Haswell)

[Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/) の **Desktop Haswell** の手順に沿って作成した EFI 一式です。
対象 macOS は **12 Monterey**(Haswell の内蔵 GPU が公式に動く最後のバージョン)。OpenCore **1.0.8 (DEBUG)**。

## リポジトリ直下の中身

| パス | 内容 |
| --- | --- |
| `EFI/` | USB(FAT32 / GPT)のルートにそのままコピーするフォルダ |
| `.github/workflows/download-macos-recovery.yml` | macrecovery.py で macOS 12 のリカバリを取得する GitHub Actions |
| Releases → `recovery-macos12-monterey` | Actions が作る `com.apple.recovery.boot.zip`(BaseSystem.dmg は 100MB 超のため Git に置けない) |

## USB の作り方(Windows / ガイド「Making the installer in Windows」)

1. USB を GPT + FAT32 でフォーマット(Disk Management か Rufus)。
2. Actions タブ → **Download macOS 12 (Monterey) recovery** を実行(初回は自動で走ります)。完了後、Releases の `com.apple.recovery.boot.zip` を展開し、**`com.apple.recovery.boot` フォルダを USB のルート**に置く(`BaseSystem.dmg` と `BaseSystem.chunklist` の両方が入っていること)。
3. このリポジトリの **`EFI` フォルダを USB のルート**に置く。
4. BIOS を下記のとおり設定して USB から起動し、ガイドの [Installation Process](https://dortania.github.io/OpenCore-Install-Guide/installation/installation-process.html) に進む。

## BIOS 設定(ガイド「Intel BIOS settings」)

- **Disable**: Fast Boot / Secure Boot / CSM / Serial・Parallel Port / Thunderbolt / Intel SGX / Intel Platform Trust / **CFG Lock**(項目が無ければ `AppleXcpmCfgLock` で対処済み)/ VT-d(有効でも `DisableIoMapper` で対処済み)
- **Enable**: VT-x / Above 4G Decoding / Hyper-Threading / Execute Disable Bit / EHCI・XHCI Hand-off / OS type: Windows 8.1/10 UEFI Mode / DVMT Pre-Allocated 64MB 以上 / SATA Mode: AHCI
- 補足(ガイド外): 内蔵 GPU を有効にし、**モニターはマザーボード側の映像出力**につなぐ(GTX 660 は macOS 12 で非対応のため映像出力に使えません)。

## 設定内容(config.plist)

| 項目 | 値 | 根拠 |
| --- | --- | --- |
| ACPI | `SSDT-PLUG-DRTNIA.aml` / `SSDT-EC-DESKTOP.aml` | Haswell デスクトップ。Dortania の prebuilt を使用 |
| Drivers | `HfsPlus.efi` / `OpenRuntime.efi` / `ResetNvramEntry.efi` | ガイド「Adding The Base OpenCore Files」 |
| Tools | `OpenShell.efi` | 同上 |
| Kexts(読み込み順) | Lilu → VirtualSMC → SMCProcessor → SMCSuperIO → WhateverGreen → AppleALC → IntelMausi → RealtekRTL8111 | LAN チップ不明のため Intel / Realtek 両方を搭載 |
| `AAPL,ig-platform-id` | `0300220D`(+ framebuffer-patch-enable / stolenmem / fbmem) | iGPU でディスプレイを駆動する場合の Haswell デスクトップ値 |
| Kernel Quirks | `AppleXcpmCfgLock`=YES、`DisableIoMapper`=YES、`PanicNoKextDump`=YES、`PowerTimeoutKernelPanic`=YES、`XhciPortLimit`=**NO** | ガイドの Haswell 表。`XhciPortLimit` は macOS 11.3 以降で無効化との指示のため NO |
| UEFI Quirks | `IgnoreInvalidFlexRatio`=YES、`RequestBootVarRouting`=YES | 同上 |
| Misc | `HideAuxiliary`=YES、`AppleDebug`/`ApplePanic`/`DisableWatchDog`=YES、`Target`=67、`ScanPolicy`=0、`SecureBootModel`=Default、`Vault`=Optional | 同上 |
| boot-args | `-v debug=0x100 keepsyms=1 alcid=1` | `alcid=1` はガイドの例。音が出ない場合は AppleALC の対応コーデック表で別の値に変更 |
| SMBIOS | `iMac16,2` | Monterey 向けの表「Haswell デスクトップ(iGPU のみ)」 |
| NVRAM | `csr-active-config`=`00000000`(SIP 有効)、`run-efi-updater`=No、`prev-lang:kbd` 空(初回に言語選択が出る) | ガイドの推奨 |

## 注意点

- **GTX 660 は無効化していません**(指定どおり)。Kepler は macOS 12 で非対応のため VESA ドライバで動作し、ガイドによるとスリープなどが不安定になる場合があります。問題が出たら、ガイドの「Disabling GPU」に従い boot-args に `-wegnoegpu` を追加してください。
- **SMBIOS のシリアルは生成済み**です。Apple の [Check Coverage](https://checkcoverage.apple.com) に入力して「Unable to check coverage」となる(= 無効なシリアルである)ことを確認してください(オフラインでは確認できません)。有効な番号だった場合は GenSMBIOS などで再生成してください。`ROM` は `112233000000` の仮値です。
- **USB マッピングは未実施**です。`XhciPortLimit` を NO にしているため、ガイドどおり [USBToolBox](https://github.com/USBToolBox/tool) で Windows 上からマッピングし、`USBToolBox.kext` と `UTBMap.kext` を `EFI/OC/Kexts` に追加することを推奨します。
- **DSDT は未取得**のため SSDT は prebuilt を使用しています。起動できない場合は [SSDTTime](https://github.com/corpnewt/SSDTTime) で K450 実機の DSDT から SSDT-PLUG / SSDT-EC を作り直してください。
- インストール完了後は、ガイドに従い OpenCore を **RELEASE** 版に入れ替え(`-v` や `debug=0x100` も外す)てください。
- Wi-Fi(内蔵)と Bluetooth は使わない前提で、関連 kext は入れていません。
