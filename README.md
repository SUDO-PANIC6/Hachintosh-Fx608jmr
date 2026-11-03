# Hackintosh: ASUS TUF Gaming F16 (FX608JMR) — macOS 15.8.1 Sequoia, OpenCore 1.0.8

OpenCore EFI for the **ASUS TUF Gaming F16 FX608JMR** (i7-14650HX, RTX 5060 Laptop GPU, 
Intel UHD, HM770, Intel AX211 Wi-Fi) running **macOS 15.8.1 Sequoia**, with the 
**NullMoth** NVIDIA driver for the RTX 5060.

> **Disclaimer.** Use everything here at your own risk. I am **not responsible for any data 
> corruption, data loss, bricked firmware, or any other damage**. Back up everything before you 
> start. Hackintoshing breaks Apple's license terms and is unsupported by Apple.
> This project is not affiliated with Apple, ASUS, Intel, NVIDIA, NullMoth or Acidanthera.

## Hardware

| Part | Model |
|---|---|
| Laptop | ASUS TUF Gaming F16 FX608JMR |
| CPU | Intel Core i7-14650HX (Raptor Lake-HX) |
| dGPU | NVIDIA GeForce RTX 5060 Laptop (GB206M) |
| iGPU | Intel UHD (not supported by macOS, unused) |
| Chipset | Intel HM770 |
| Wi-Fi / BT | Intel AX211 (CNVi) |

## What works

- USB: all ports, 2.0 and 3.0, Type-A and Type-C
- Wi-Fi (via itlwm + HeliPort)
- Camera
- Microphone
- Audio
- Trackpad
- Keyboard
- Ethernet (Realtek RTL8111 via Mieze's kext)
- HDMI output (wired to the dGPU): works, both when plugged in before boot and when hot-plugged after the desktop loads
- Battery status (SMCBattery)
- CPU temperatures (VirtualSMC / SMCProcessor)
- NVIDIA RTX 5060 Laptop GPU (NullMoth driver)
- GPU utilization readout (HWMonitorSMC2)

## Not tested / not working

- Intel iGPU: unsupported by macOS on Raptor Lake. It is not used, the dGPU drives the display.
- Sleep/wake: **does not work**. The screen goes black, the fans and keyboard backlight stay on, and the laptop never wakes (hard reset needed). Disable sleep with `sudo pmset -a disablesleep 1`.
- Not tested yet: Bluetooth, DisplayPort / USB-C video output, 
  fan readings, GPU temperature readout. Fill this list in as you test.
- Power limits (PL1/PL2) and fan control cannot be read or changed from macOS.
  Use the BIOS or Windows tools.

## 1. BIOS settings

Before changing anything, **if BitLocker is on in Windows, back up your BitLocker recovery key 
first** (Settings → Privacy & security → Device encryption, or your Microsoft account). 
Disabling Secure Boot can trigger a BitLocker recovery prompt, and without the key you lose 
access to your Windows data.

Then set:

| Setting | Value |
|---|---|
| Secure Boot | **Disabled** |
| CSM / Legacy | Disabled |
| Hyper-Threading | **Off** |
| E-cores (Active Efficient Cores) | **0** |
| VT-d | **Off** |
| Intel Virtualization (VT-x) | On |
| Above 4G Decoding | On |
| GPU mode (MUX) | **dGPU only** (discrete GPU) |

Resizable BAR and any other BAR options: follow the NullMoth README for your driver version.

## 2. Get macOS 15.8.1 (offline, full installer image)

I installed from a **full offline installer image (a complete macOS 15.8.1 `.dmg`)**, 
**not** the online recovery process (`macrecovery.py`). The online method downloads a small 
recovery image and then fetches macOS from Apple during install. It needs a working network 
connection inside the installer and may serve a different version than the one you want, since 
Apple decides what it offers. The offline method has the whole installer on the USB stick, so 
it doesn't depend on that.

1. Get the full **macOS 15.8.1 Sequoia** installer image. Check that the version is exactly 
   **15.8.1**.  
   `<name the tool or source you used to get the full .dmg>`
2. Write the image to a USB stick (16 GB or larger is safest). Format the stick with a 
   **GUID partition map**.
3. Make sure the stick has a **FAT32 EFI partition** and copy this repo's `EFI` folder to its 
   root.
4. Boot from the stick through the BIOS boot menu, choose the OpenCore entry, then the macOS 
   installer.

If you'd rather use the online method, run `macrecovery.py` from OpenCorePkg's 
`Utilities/macrecovery` and check the version it downloads. The result can change as Apple 
updates its servers.

## 3. Make your own SMBIOS

The `config.plist` in this repo has **blank** identity fields. Generate your own with 
GenSMBIOS before booting:

- Model: `iMacPro1,1`
- Fill in `SystemSerialNumber`, `MLB`, `SystemUUID` and `ROM` under 
  `PlatformInfo → Generic`.

Never reuse another person's serial. Never post yours.

## 4. NVIDIA driver (NullMoth)

Download the RTX 5060 driver from the NullMoth website and follow their README. The driver files 
are **not** included here. 

The config already contains the settings it needs:

- `csr-active-config` = `430A0000`
- `AMFIPass.kext` enabled, plus the AMFI boot-args
- `IONDRVSupport` kext block
- `SecureBootModel` = `Disabled`
- boot-args `nvfb=1 nvaccel=1 nvfbheads=4 gfxrst=4`

> **Security note.** These settings turn off parts of macOS's security (SIP, AMFI). Don't use 
> this setup for anything sensitive.

## 5. Config highlights

- **CPU:** Cpuid1Data/Cpuid1Mask spoof for Raptor Lake, `ProvideCurrentCpuInfo`, 
  `AppleCpuPmCfgLock` and `AppleXcpmCfgLock`, `DisableIoMapper`, RestrictEvents 
  (`revcpu=1 revcpuname="Intel Core i7-14650HX"` shows the real CPU name in About This Mac).
- **ACPI:** `SSDT-EC`, `SSDT-PLUG-ALT`, `SSDT-RTCAWAC`, `SSDT-USBX`, made with SSDTTime from my 
  own laptop's dump. `SSDT-EC` is laptop-specific. If your model is different, 
  generate your own with SSDTTime.
- **Kexts:** Lilu, VirtualSMC (+ SMCProcessor, SMCBattery), WhateverGreen, AppleALC, 
  RestrictEvents, AMFIPass, NVMeFix, USBToolBox + UTBMap, VoodooPS2 / VoodooI2C for the trackpad, 
  itlwm for Wi-Fi.
- **Audio:** `alcid=11` (ALC256). Try other layout IDs if yours differs.

## 6. Wi-Fi

Use **itlwm** with the **HeliPort** menu-bar app to connect. Native AirportItlwm needs extra 
patching on Sequoia, so it isn't used here.

## 7. Monitoring

For CPU and fan sensors, use a VirtualSMC-based app such as **HWMonitorSMC2**, **Stats** or 
iStat Menus. Add `SMCProcessor.kext` for CPU temperatures. Compare readings against Windows 
HWiNFO the first time, since the chip is new.

## Credits

Acidanthera (OpenCorePkg, Lilu, VirtualSMC, WhateverGreen, AppleALC, RestrictEvents), the 
Dortania guides, SSDTTime (CorpNewt), ProperTree and GenSMBIOS (CorpNewt), VoodooPS2 / VoodooI2C, 
itlwm / HeliPort, the Realtek RTL8111 kext by Mieze, and NullMoth for the RTX 5060 driver.
