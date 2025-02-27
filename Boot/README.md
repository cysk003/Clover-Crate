# Boot
![Boot](https://user-images.githubusercontent.com/76865553/136703418-a28fed86-1f46-4519-80ad-671e96b89141.jpeg)

<details>
<summary><strong>TABLE of CONTENTS</strong> (click to reveal)</summary>

- [Issues](#issues)
- [Arguments](#arguments)
  - [Debugging](#debugging)
  - [GPU-specific boot arguments](#gpu-specific-boot-arguments)
  - [Network-specific boot arguments](#network-specific-boot-arguments)
  - [Other useful boot arguments](#other-useful-boot-arguments)
- [Boot-args and device properties provided by kexts](#boot-args-and-device-properties-provided-by-kexts)
  - [Lilu.kext](#lilukext)
  - [Whatevergreen.kext](#whatevergreenkext)
  - [AppleALC](#applealc)
- [Custom Logo](#custom-logo)
- [Debug](#debug)
  - [Enabling the Preeboot.log](#enabling-the-preebootlog)
- [Default Boot Volume](#default-boot-volume)
- [Default Loader](#default-loader)
- [DisableCloverHotkeys](#disablecloverhotkeys)
- [Fast](#fast)
- [HibernationFixup](#hibernationfixup)
- [Legacy](#legacy)
- [NeverDoRecovery](#neverdorecovery)
- [NeverHibernate](#neverhibernate)
- [NoEarlyProgress](#noearlyprogress)
- [RtcHibernateAware](#rtchibernateaware)
- [SignatureFixup](#signaturefixup)
- [SkipHibernateTimeout](#skiphibernatetimeout)
- [StrictHibernate](#stricthibernate)
- [Timeout](#timeout)
- [XMPDetection](#xmpdetection)
- [Adding Clover entry to the BIOS Boot menu](#adding-clover-entry-to-the-bios-boot-menu)
- [Links](#links)

</details>

## Issues

- Apparently, Clover cannot handle _combined_ boot-args. I noticed that Firefox kept crashing in Sequoia on my Intel Ivy Bridge Laptop. The reason was that the `revpatch=f16c` boot-arg provided by `RestrictEvent.kext` that prohibits core graphics from crashing by disabling f16c instruction set was not working because I combined it with other revpatch arguments into a long string (`revpatch=sbvmm,memtab,f16c`). After splitting the string into three single boot-args, it worked again.

## Arguments
The following tables contain boot arguments that are passed over to `boot.efi`, which in return passes them down to the system kernel. See Apple's documentation for a list in the `com.apple.Boot.plist`. Some commonly used ones are:

### Debugging
|Boot-arg|Description|
|:------:|-----------|
**`-v`**|_V_erbose Mode. Replaces the progress bar with a terminal output with a bootlog which helps resolving issues. Combine with `debug=0x100` and `keepsyms=1`
**`-f`**|_F_orce-rebuild kext cache on boot.
**`-s`**|_S_ingle User Mode. This mode will start the terminal mode, which can be used to repair your system. Should be disabled with a Quirk since you can use it to bypass the Admin account password.
**`-x`**|Safe Mode. Boots macOS with a minimal set of system extensions and features. It can also check your startup disk to find and fix errors like running First Aid in Disk Utility. Can be triggered from OC boot menu by holding a key combination if `PollAppleHotkeys` is enabled.
**`debug=0x100`**|Disables macOS'es watchdog. Prevents the machine from restarting on a kernel panic. That way you can hopefully glean some useful info and follow the breadcrumbs to get past the issues.
**`keepsyms=1`**|Companion setting to `debug=0x100` that tells the OS to also print the symbols on a kernel panic. That can give some more helpful insight as to what's causing the panic itself.
**`dart=0`**|Disables VT-x/VT-d. Nowadays, `DisableIOMapper` Quirk is used instead.
**`cpus=1`** | Limits the number of CPU cores to 1. Helpful in cases where macOS won't boot or install otherwise.
**`npci=0x2000`** / **`npci=0x3000`** | Disables PCI debugging related to `kIOPCIConfiguratorPFM64`. Alternatively, use `npci=0x3000` which also disables debugging of `gIOPCITunnelledKey`. Required when stuck at `PCI Start Configuration` as there are IRQ conflicts related to your PCI lanes. **Not needed if `Above4GDecoding` can be enabled in BIOS**
**`-no_compat_check`** | Disables macOS compatibility checks. Allows installing and booting macOS with unsupported SMBIOS/board-ids. Downside: you can't install system updates if this boot-arg is active. But this restriction can be worked around by adding `RestrictEvents.kext` and boot-arg `revpatch=sbvmm` ([requires macOS 11.3 or newer](https://github.com/5T33Z0/OC-Little-Translated/tree/main/S_System_Updates)).

### GPU-specific boot arguments
For more iGPU and dGPU-related boot args see the Whatevergreen topic.

|Boot-arg|Description|
|:------:|-----------|
**`agdpmod=pikera`**|Disables Board-ID checks on AMD Navi GPUs (RX 5000 & 6000 series). Without this you'll get a black screen. Don't use on Polaris or Vega Cards.
**`igfxonln=1`**|Forces all displays online. Resolves screen wake issues after quitting sleep mode in macOS 10.15.4 and newer when using Coffee and Comet Lake's Intel UHD 630.
**`-igfxvesa`** |Disables graphics acceleration in favor of software rendering. Useful if iGPU and dGPU are incompatible or if you are using an NVIDIA GeForce Card and the WebDrivers are outdated after updating macOS, so the display won't turn on during boot.
**`-wegnoegpu`**|Disables all GPUs but the integrated graphics on Intel CPU. Use if GPU is incompatible with macOS. Doesn't work all the time.
**`nvda_drv=1`**|Enable Web Drivers for NVIDIA Graphics Cards (supported up to macOS High Sierra only).
**`nv_disable=1`**|Disables NVIDIA GPUs (***don't*** combine this with `nvda_drv=1`)
**`unfairgva=x`** (x = number from 1 to 7 )| Replaces `shikigva` in macOS 11+ to address issues with [**DRM**](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.Chart.md) on AMD cards. It's a bitmask with 3 bits (1, 2 and 4) which can be combined to enable/select different features as [explained here](https://www.insanelymac.com/forum/topic/351752-amd-gpu-unfairgva-drm-sidecar-featureunlock-and-gb5-compute-help/)

### Network-specific boot arguments
|Boot-arg|Description|
|:------:|-----------|
**`dk.e1000=0`**|Prohibits `com.apple.DriverKit-AppleEthernetE1000` (Apple's DEXT driver) from attaching to the Intel I225-V Ethernet controller used on higher end Comet Lake boards, causing Apple's I225 kext driver to load instead. This boot argument is optional on most boards as they are compatible with the DEXT driver. However, it may be required on Gigabyte and several other boards, which can only use the kext driver, as the DEXT driver causes hangs. You don't need this if your board didn't ship with the I225-V NIC.

### Other useful boot arguments
|Boot-arg|Description|
|:------:|-----------|
**`alcid=1`**|For selecting a layout-id for AppleALC, whereas the numerical value specifies the layout-id. See [supported codecs](https://github.com/acidanthera/applealc/wiki/supported-codecs) to figure out which layout to use for your specific system.
**`amfi_get_out_of_my_way=1`**|Combined wit disabled SIP, this disables Apple Mobile File Integrity. AMFI is a macOS kernel module enforcing code-signing validation and library validation which strengthens security. Even after disabling these services, AMFI is still checking the signatures of every app that is run and will cause non-Apple apps to crash when they touch extra-sensitive areas of the system. There's also a [kext](https://github.com/osy/AMFIExemption) which does this on a per-app-basis.
**`-force_uni_control`**|Force-enables the Universal Control service in macOS Monterey 12.3+.

## Boot-args and device properties provided by kexts

### Lilu.kext
Assorted Lilu boot-args. Remember that Lilu acts as a patch engine providing functionality for other kexts in the hackintosh universe, so you got to be aware of that if you use any of these commands!

|Boot-arg|Description|
|:-------:|-----------|
**`-liluoff`**| Disables Lilu.
**`-lilubeta`**| Enables Lilu on unsupported macOS versions (macOS 12 and below are supported by default).
**`-lilubetaall`**| Enables Lilu and *all* loaded plugins on unsupported macOS (use _very_ carefully).
**`-liluforce`**| Enables Lilu regardless of the mode, OS, installer, or recovery.
**`liludelay=1000`**| Adds a 1 second (1000 ms) delay after each print for troubleshooting.
**`lilucpu=N`**|to let Lilu and plugins assume Nth CPUInfo::CpuGeneration.

### Whatevergreen.kext 
Listed below you'll find a small but useful assortment of WEG's boot args for everything graphics-related. Check the [complete list](https://github.com/acidanthera/WhateverGreen/blob/master/README.md#boot-arguments) to find many, many more.

|Boot-arg|Description|
|:------:|-----------|
**`-wegoff`**| Disables WhateverGreen.
**`-wegbeta`**| Enables WhateverGreen on unsupported OS versions.
**`-wegswitchgpu`**|Disables th iGPU if a discrete GPU is detected (or use `switch-to-external-gpu` property to iGPU)
**`-wegnoegpu`**|Disables all discrete GPUs (or add `disable-gpu` property to each GFX0).
**`-wegnoigpu`**|Disables internal GPU (or add `disable-gpu` property to iGPU)
**`agdpmod=pikera`**| Replaces `board-id` with `board-ix`. Disables Board-ID checks on AMD Navi GPUs (RX 5000 & 6000 series). Without this, you’ll get a black screen. Don’t use on Polaris or Vega cards.
**`agdpmod=vit9696`**|Disables check for `board-id` (or add `agdpmod` property to external GPU).
**`applbkl=0`**| Boot argument (and `applbkl` property) to disable `AppleBacklight.kext` patches for the iGPU. In case of custom AppleBacklight profile, read [this](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.OldPlugins.en.md)
**`gfxrst=1`**|Prefers drawing the Apple logo at the 2nd boot stage instead of framebuffer copying. Makes the transition between the progress bar and the desktop/login screen smoother if an external monitor is attached.
**`ngfxgl=1`**|Disables Metal support on NVIDIA cards (or use `disable-metal` property)
**`igfxgl=1`**|boot argument (and `disable-metal` property) to disable Metal support on Intel.
**`igfxmetal=1`**|boot argument (and `enable-metal` property) to force enable Metal support on Intel for offline rendering.
**`-igfxvesa`**|Disable Intel Graphics acceleration in favor of software rendering (aka VESA mode). Useful when installing never macOS lacking graphics drivers for legacy hardware.
**`-igfxnohdmi`**| boot argument (and `disable-hdmi-patches` property) to disable DP to HDMI conversion patches for digital sound.
**`-cdfon`**| Boot-arg (and `enable-hdmi20` property) to enable HDMI 2.0 patches.
**`-igfxhdmidivs`**| boot argument (and `enable-hdmi-dividers-fix` property) to fix the infinite loop on establishing Intel HDMI connections with a higher pixel clock rate on SKL, KBL and CFL platforms.
**`-igfxlspcon`**|boot argument (and `enable-lspcon-support` property) to enable the driver support for onboard LSPCON chips. [Read the manual](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.IntelHD.en.md#lspcon-driver-support-to-enable-displayport-to-hdmi-20-output-on-igpu)
**`igfxonln=1`**| boot argument (and `force-online` device property) to force online status on all displays.
**`-igfxdvmt`**| boot argument (and `enable-dvmt-calc-fix` property) to fix the kernel panic caused by an incorrectly calculated amount of DVMT pre-allocated memory on Intel ICL platforms.
**`-igfxblr`**| boot argument (and `enable-backlight-registers-fix` property) to fix backlight registers on KBL, CFL and ICL platforms.
**`-igfxbls`**| boot argument (and `enable-backlight-smoother` property) to make brightness transitions smoother on IVB+ platforms. [Read the manual](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.IntelHD.en.md#customize-the-behavior-of-the-backlight-smoother-to-improve-your-experience)
**`applbkl=3`**| boot argument (and `applbkl` property) to enable PWM backlight control of AMD Radeon RX 5000 series graphic cards [read here.](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.Radeon.en.md)
**`-igfxblr`**| Boot argument (and enable-backlight-registers-fix property) to fix backlight registers on KBL, CFL and ICL platforms.

### AppleALC
Boot-args for your favorite audio-enabler kext. All the Lilu boot arguments affect AppleALC as well.

|Boot-arg|Description|
|:------:|-----------|
**`alcid=layout`**| To select a layout-id, for example alcid=1
**`-alcoff`**| Disables AppleALC (Bootmode `-x` and `-s` will also disable it)
**`-alcbeta`**| Enables AppleALC on unsupported systems (usually unreleased or old ones)
**`alcverbs=1`**| Enables alc-verb support (also alc-verbs device property)

If you right-click anywhere in the "Arguments" list, you will find many more boot-args not covered here.

## Custom Logo
Enables the drawing of the custom boot logo.

- `YES`– Uses the default boot style, Apple.
- `NO` – Disables custom boot logo.
- `Apple` – Use the default gray on gray Apple logo.
- `Alternate` – Use the alternate white on black Apple logo.
- `Theme` – Use the theme boot screen for entry type - NOT IMPLEMENTED.
- `None` – Use no logo only background color, gray if not specified by custom entry.

> [!NOTE]
> As of r5140.1 "Apple" and "Alternate" don't show a progress bar. Feels like these are just a wallpaper covering the bootscreen.

## Debug
If set to `true`, a debug log will be created on next boot. This will slow down the boot time significantly (up to 10 minutes) but allows you figure out what the problem is since each step will be documented by writing a `debug.log` to disk/flash drive which will be saved line for line. So even if the system hangs and you have to perform a hard reset, you won't lose the log. It will be located under `/EFI/CLOVER/misc/debug.log` and collectively records all logs for all boots, as long as the `Debug` feature is enabled.

### Enabling the Preeboot.log
Alternatively, you could use the the `preboot.log` instead which is not as in-depth but still very useful for identifying configuration issues and hardware conflicts. To enable it, press the `F2` key in the Clover Boot Menu GUI. The `preboot.log` will be saved in under `EFI/CLOVER/misc/`. However, the file's log entries end once macOS takes over control of the system. But it is still a useful resource for resolving issues that occur prior to macOS being started – and it's way faster than using the `Debug` option.

> [!WARNING]
> If you delete `EFI/CLOVER/misc` from the folder structure then the `Debug` features will no longer work!

## Default Boot Volume
Default Boot Volume is used to specify which entry is the default boot entry in the Clover GUI. It can be set to:

- `LastBootedVolume`: The last booted volume will be set as default one in Clover GUI.
- Volume Name: The name of the volume. E.g. `Macintosh`.
- GUID - Globally Unique ID of the volume shown in Clover's boot, preboot or debug log. E.g. `57272A5A-7EFE-4404-9CDA-C33761D0DB3C`.
- Part of Device Path - Also shown in Clover's logs. E.g. HD(1,GPT,57272A5A-7EFE-4404-9CDA-C33761D0DB3C,0x800,0xFF000).

OS X Startup Disk can be used to reboot into another volume, but for the following reboot DefaultVolume will be used again.

## Default Loader
In addition to `DefaultVolume`, the path to the default boot loader of an Operating System can be specified here by entering the path to the boot loader or a unique portion of the file name. This can be useful for Volumes which contain multiple boot loaders, so the desired boot loader is used by default when said volume is selected.

**IMPORTANT**: You cannot use the `Default Loader` section to set Clover or OpenCore as default Boot Loaders, since both are *Boot Managers*, not *Boot Loaders!* Boot Managers choose which Boot Loader to select. They don't actually boot the system. That's the job of Boot Loaders. Examples for *Boot Loaders* are:

- `boot.efi` for macOS
- `bootmgfw.efi` for Windows
- `grubx64.efi` for Linux

## DisableCloverHotkeys
Disables all hotkeys in the bootloader menu. The list of all hotkeys can be found by pressing F1 in the bootloader menu.

## Fast
`Fast` is similar to setting `Timeout` to `0` but there are some differences:

- Skips the bootmenu GUI completely &rarr; No user interaction with the boot menu to change the OS or any settings is possible
- Boots directly into the default/last used volume &rarr; no chance to switch the OS
- Does not search for the best video mode 

:warning: Use with caution! Should only be used if macOS is the only OS on your machine.

## HibernationFixup
Fixes hibernation on systems without native NVRAM support. To be used for [**Hibernation modes**](https://github.com/5T33Z0/OC-Little-Translated/tree/main/04_Fixing_Sleep_and_Wake_Issues/Changing_Hibernation_Modes) 25 and 3 with `Lilu.kext` and [**`HibernationFixup.kext`**](https://github.com/acidanthera/HibernationFixup).

## Legacy
Legacy Boot. Necessary for running older versions of Windows and Linux. Depends on the hardware and the BIOS, so several algorithms have been implemented. 

The options are:

- `LegacyBiosDefault`: for those UEFI BIOSes that contain a LegacyBios protocol.
- `PBRtest`, `PBR`, `PBRsata` - variants of the PBR boot algorithm.

In general, it has not been possible to achieve unconditional legacy boot operation. It is easier and better to forget about legacy systems and use UEFI versions of OSes. The oldest of them is Windows 7-64, and I personally see no reason to stick with Windows XP. Does anyone still use a 32-bit only processor? Well, good luck then!

## NeverDoRecovery
With FileVault2 technology, it is now possible to use a hotkey, but to do so, you have to override the assignments already made in Clover.

## NeverHibernate
Disables the hibernation state detection.

## NoEarlyProgress
Removes tooltips before loading the loader interface, e.g. "Welcome to Clover".

## RtcHibernateAware
The key to safely work with the RTC during hibernation. Only required in macOS 10.13.

## SignatureFixup
 When hibernating, the system leaves a signature in the image, which is then checked by `boot.efi`. We wanted to fix it with this key. Probably for nothing. It's recommended to leave it disabled.
 
## SkipHibernateTimeout
Disables the Hibernation Timeout.

## StrictHibernate
This only works if you have working NVRAM. It is compatible with FileVault2 technology, where the old method does not work.

## Timeout
Sets the timeout in seconds (0-30 s), before the boot process continues loading the default volume. The timer is disabled if a user input occurs before the timer ends.

- `-1`: sets the timeout to infinite. Nothing happens until a user input happens.
- `Fast`: Skips the GUI and boots from the default volume instantly, so no user interaction is possible – so no chance to correct something in case of an error.

## XMPDetection
Enables (if set to `0`) or disables XMP detection. By default it's disabled (set to `-1`). Usually, enabling this feature is only necessary if the XMP Profile of the RAM is not detected. In addition to simply enabling/disabling XMP detection, you can can also choose between Profile 1 (`1`) or Profile 2 (`2`) if your RAM modules support different memory profiles. Perhaps in the future the profiles will be used for other purposes.

## Adding Clover entry to the BIOS Boot menu
If for some reason the entry for the partition containing the Clover bootloader is missing from your BIOS boot menu, you can let Clover create one. But this has to be done in from within the Clover boot menu because this affects UEFI BIOS variables working prior to Clover being started, so it can't be set up in the config.plist.

In order to do so, do the following:

1. In the Clover boot menu, select "Clover Boot Options":</br>![screenshot1](https://user-images.githubusercontent.com/76865553/159431070-103960ad-90b8-4a1a-b86c-7127c9bfac2d.png)
2. In the next screen, you will find the PCI path pointing to the `CLOVERX64.efi` bootloader file. Clicking or hitting Enter on "Add Clover boot options for all entries" will generate a boot entry in the BIOS boot menu:</br>![screenshot2](https://user-images.githubusercontent.com/76865553/159431126-4afa3874-d322-4cf2-b18e-942e8e76b86d.png)

## Links
- [**How to make Clover the default Boot Manager**](https://www.insanelymac.com/forum/topic/326494-how-to-make-clover-default-bootloader-after-installing-windows-on-uefi/)
