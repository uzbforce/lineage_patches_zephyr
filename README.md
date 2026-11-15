# xiaomi_zephyr_cusrom_patches

Source patches needed to build a custom ROM (LineageOS 23.2) for the
**Redmi Note 17 4G** — codename `zephyr`, SM6225-AF (`bengal`).

The port is four things. This repo holds the one that cannot live in the
others: changes to **ROM source projects themselves**.

| Repo | Holds |
|---|---|
| [android_device_xiaomi_zephyr](https://github.com/uzbforce/android_device_xiaomi_zephyr) | device tree: `BoardConfig.mk`, product makefiles, `rootdir/`, `configs/`, `sepolicy/`, `overlay/`, `tools/`, the local manifest for the platform tree |
| [android_vendor_xiaomi_zephyr](https://github.com/uzbforce/android_vendor_xiaomi_zephyr) | the blobs taken out of the stock firmware |
| **xiaomi_zephyr_cusrom_patches** (this repo) | patches against ROM source projects, and the helpers that make them |
| SM6225 platform repos | the platform HAL tree LineageOS 23.2 does not ship (`hardware/qcom-caf/common`, `hardware/qcom-caf/sm6225/*`, `device/qcom/sepolicy_vndr/sm6225`), mirrored from the `rn124g-development` forks into the `uzbforce` org and pinned by `device/xiaomi/zephyr/local_manifests/zephyr.xml` |

## Layout

`patches/` mirrors the ROM checkout layout exactly:

```
patches/hardware/qcom-caf/common/0001-some-fix.patch
patches/build/make/0001-another-fix.patch
```

so the target of every patch is unambiguous — it applies in
`$ANDROID_BUILD_TOP/<that path>`, and nothing declares a mapping that can drift
out of sync with the checkout.

## Applying

```bash
./apply.sh                          # uses $ANDROID_BUILD_TOP, else $PWD
./apply.sh /path/to/lineage/checkout
```

Patches are applied in filename order. Each one is validated with
`git apply --check` first, patches already present in the tree are skipped, and
the first failure stops the run and names the file. Nothing is committed — the
changes stay unstaged so they can be reviewed, and `git -C <project> diff`
still works.

## Capturing the change you just made

```bash
./capture.sh hardware/qcom-caf/common "fix the pickup for bengal"
```

diffs that project against its HEAD, writes
`patches/hardware/qcom-caf/common/0001-fix-the-pickup-for-bengal.patch`, and
leaves the working tree alone. Untracked files are part of the change, so they
are marked intent-to-add for the duration of the run and appear in the patch as
`new file mode` entries; the marker is removed afterwards and only those exact
paths are touched, so anything you had staged stays staged. Use the same command
to re-capture after editing an already captured patch (a new number is added, so
delete the old file).

Four projects had accumulated a multi-file series. They now carry one current
patch per project — `frameworks/av`, `packages/apps/Settings`, `system/sepolicy`
and `kernel/xiaomi/zephyr` — plus a separate kernel patch for the ignored vendor
DTS tree. Applying the current series reproduces the tree the previous series
built, without replaying its history.

## What belongs here

Only changes to ROM source projects. Anything device-specific — `BoardConfig`,
props, HAL lists, fstab, overlays, sepolicy — belongs in the device tree, and
blobs belong in the vendor repo. The test is simple: if the change would have to
be made again after a `repo sync` because Lineage owns the file, it belongs
here.

## What is in here

| Patch | Fixes |
|---|---|
| `build/soong` | the build UI announces "Starting rbe..." before the bootstrap command and names rbe in its shutdown failure, although this host builds locally; the messages now say ninja and soong |
| `device/lineage/sepolicy` | its public types (`iso9660`, `udf`, `vendor_persist_nfc_prop`, `usb_uvc_config_prop`, `xtra_control_prop`) are listed as new objects for API 202404 only, so `treble_sepolicy_tests_202504` reported all five as added without a compatibility mapping |
| `device/qcom/sepolicy_vndr/sm6225` | `BOARD_*_PREBUILT_DIR` → `BOARD_*_SEPOLICY_PREBUILT_DIRS`, which `build/make` errors on in Android 16; the Project Tango entry in `vendor/common/seapp_contexts` assigns `domain=untrusted_app`, a coredomain, which `checkseapp` now rejects outright; and the vendor `genfscon` for `/class/typec` clashes with the platform's `sysfs_typec` at board API 202604, which `secilc` rejects as a conflicting genfscon |
| `frameworks/av` | the codec2 blobs call `GraphicBufferSource::getHGraphicBufferProducer()`, which Android 16 only exports when its flag is on, so `vendor.qti.media.c2@1.0-service` failed to link; the accessor is exported in both flag variants. Separately, `OmxInfoBuilder` fetched `IOmxStore` with a blocking `getService()`, which spins forever because the stock OMX HAL is 32-bit and this product is 64-bit-only, stalling `MediaCodecList` in every client so all players sat in BUFFERING; it now asks for the stub and reports `NO_INIT`, leaving the C2 builders to enumerate. `CCodec::CreateInputSurface()` also asked that HIDL-only store for a store-provided surface, a call that never returns, so `screenrecord` and every encoder that wanted an input surface died on the CCodec watchdog; the `debug.stagefright.c2inputsurface=-1` local `AidlGraphicBufferSource` path is now tested before the store call |
| `frameworks/base` | `StorageManagerService` enumerated codecs on its handler thread while looking for transcoding candidates, and the vendor Codec2 HAL never starts, so `getCodecList()` never returned and every message queued behind it - boot completed, user unlock, volume mount - stayed unprocessed; transcoding runs on a thread of its own now, and the emulated volume is re-mounted 15 s after boot in case MediaProvider won the race. Separately the stock `org.codeaurora.ims` and `QtiTelephony` blobs call `android.telephony.TelephonyBaseUtilsStub.isMiuiRom()`, a class AOSP does not have, so the IMS service died in `ImsService.setup()` and no IMS was ever registered; the stub reports `false`. SystemUI had no IMS icon at all - no `volte` slot in `config_statusBarIcons` and no drawable - so VoLTE and VoWiFi calls worked while the status bar showed nothing; `ImsIconController` drives a new `volte` slot from `ImsMmTelManager` registration callbacks and picks the VoLTE or VoWiFi icon from the transport that is actually registered. |
| `hardware/interfaces` | the AIDL Wi-Fi HAL forwarded the framework's multi-STA "primary connection" hint to the vendor HAL unconditionally, and the QCA driver on sm6225 answers the underlying vendor command with `-ENODEV` when a single STA iface is active, which made the vendor HAL tear itself down in the middle of `createStaIface` so Wi-Fi never came up; the hint is skipped while only one STA iface exists |
| `hardware/lineage/compat` | the stock `qccsyshal@1.2` blobs import protobuf 3.9 entry points protobuf 4.25 no longer exports - `Message::CopyWithSourceCheck`, `MessageLite::ParseFromString(const std::string&)` and the `internal::AssignDescriptors` overload that takes `google::protobuf::Metadata` - so the vendor HAL could not link at all; a shim module defines the old mangled names and forwards to `CopyFrom`/`ParseFromArray`, with the descriptor parking left a no-op. The same module defines `DynamicDisplayInfo`/`StaticDisplayInfo` out of line too, because the vendor `libgui_vendor` still imports those symbols while the platform moved them into headers with inline bodies |
| `hardware/lineage/interfaces` | the AIDL fingerprint HAL now drives the vendor FOD `extCmd` from `onPointerDown`/`onPointerUp`, which arms the stock 2.1 `fingerprint.goodix_fod.so` capture window; without it, enrollment timed out after 60 s. On the P19A panel it also sets the local white 1000-nit HBM mode on press, releases it on pointer-up, and has a 5 s watchdog so the panel cannot stay in HBM |
| `hardware/qcom-caf/common` | the platform namespaces do not import `hardware/google/pixel/kernel_headers`, so `qti_kernel_headers` is unknown to the display trees; and `nqnfcinfo` is built from source although zephyr has no NQ NFC and no `linux/nfc/nfcinfo.h`, while the stock binary is already a blob |
| `hardware/qcom-caf/sm6225/data-ipa-cfg-mgr` | `ipacm` puts `qti_kernel_headers` on its `header_libs`, and its own namespace only imports the platform one; soong imports are not transitive, so it has to name the kernel-headers namespace too |
| `hardware/qcom-caf/thermal` | `android.hardware.thermal-service.qti` needs the kernel-headers namespace; and the SKIN ramp read `xo-therm`, the crystal/PMIC zone that idles at ~39.5 C with the screen on, so thermalservice reported `MODERATE` from boot and camera apps refused to record video - the whole ramp is shifted up ~5 C so idle is `NONE` |
| `hardware/qcom-caf/wlan` | the same unsupported command reached through the legacy HAL instead: firmware rejects `CONCURRENT_STA_PRIMARY` with `-ENODEV` and the netlink failure is fatal, so the legacy path also reports success without sending it - the chip has no concurrent STA support (`dbs=0`, `mimo=0`) |
| `vendor/qcom/opensource/dataservices` | same for `librmnetctl` |
| `hardware/qcom-caf/sm6225/audio/primary-hal` | `libqcompostprocbundle` needs `<sound/audio_effects.h>` from the audio techpack; the kernel drop has none, so the module's build rule is disabled and the stock library in the vendor image is used |
| `packages/apps/Dialer` | Telecom can report a Bluetooth call-audio route while the SCO link is still pending, and `ProximitySensor` trusted that stale route, so in-call ear detection never armed even though audio was still on the earpiece. It now follows `AudioManager.getCommunicationDevice()` and re-evaluates when the communication device changes, leaving Bluetooth, speaker and wired routing untouched |
| `packages/apps/Settings` | `EnableVerboseVendorLoggingPreferenceController.isAvailable()` called `ServiceManager.waitForDeclaredService()` for `android.hardware.dumpstate.IDumpstateDevice/default`, which blocks until the service actually registers. zephyr's VINTF manifest declares it but nothing on the device registers it, and the call runs on the main thread from `DashboardFragment.onCreatePreferences()`, before the first frame - so Developer options never drew and input dispatch timed out into `ANR in com.android.settings`. A non-blocking `getService()` lookup replaces it, so the preference reports itself unavailable instead of hanging |
| `system/libbase` | the prebuilt health and sensors HALs import the pre-`string_view` `WriteStringToFd(const std::string&, borrowed_fd)` and `Trim(const std::string&)`, which the template rewrite dropped; both are kept as asm-labelled legacy symbols so the vendor namespace still resolves them |
| `system/sepolicy` | `treble_labeling_tests` aborts when `treble_labeling_violators` is absent from the binary policy, which happens when no vendor domain claims the exemption and the compiler drops the empty attribute; an absent attribute now means an empty set |
| `tools/extract-utils` | symlink modules take their soong name from the path, and `vendor/bin/[` is not an identifier soong accepts |
| `vendor/lineage` | `lineage_generator` hands an unknown `$(VAR)` back for a later expansion, but soong expands the generator command during analysis; a device tree with a prebuilt kernel never defines the kernel variables, so analysis stopped on them, and an undefined vendor variable now expands to nothing |
| `vendor/qcom/opensource/power` | the QTI power rc declared no `interface aidl` line, so init answered system_server's `ctl.interface_start` for `android.hardware.power.IPower/default` with `PROP_ERROR_HANDLE_CONTROL_MESSAGE` and the watchdog killed the blocked system_server; the service now declares the interface |
| `vendor/qcom/opensource/vibrator` | `leds-qpnp-vibrator-ldo` is a module, so the transient-trigger attributes do not exist at post-fs-data, and every HAL start rewrote `trigger`, which made `led_trigger_set()` recreate `activate`/`duration`/`state` as `root:root 0644`. The system-uid HAL could not open them and silently fell back to an input FF device zephyr does not have. The HAL now selects the trigger only when the attributes are missing, and the rc selects the trigger, chowns the four attributes and restarts the service once boot has completed. Requires the device tree's `sysfs_leds` labels |
| `kernel/xiaomi/zephyr` | the exported `linux/sched/types.h` defines `struct sched_param`, which bionic defines as well, so a module that includes both fails with `redefinition of 'sched_param'`; the patch puts it inside `#ifdef __KERNEL__`, and `headers_install` unifdefs it away. The tree also has to build `techpack/` into the kernel, enable the GKI fragments the display and charger modules need (`DRM_MSM` without `DRM_MSM_DP`, `ANDROID_VENDOR_HOOKS`, `LX_CP_POLICY_MODULE`, `-Wno-*` for pre-clang-21 vendor code), and guard the `sc853x` chargepump registration, which otherwise fails to link without the charge-pump policy module. A second patch adds the ignored 5,291-file `arch/arm64/boot/dts/vendor` tree, which the DTC include-prefix symlink resolves to |
| `lineage-sdk` | `LineageGlobalActions.getUserActionsArray()` throws when the `lineageglobalactions` binder service is not registered yet, and SystemUI builds its global actions inside `KeyguardService.onCreate`, so systemui crashed at every boot; a missing service now yields the local list |

The kernel is not a `repo` project: it is hand-cloned into `kernel/xiaomi/zephyr`
as described in the device tree README. Its patch applies to that checkout like
any other, and `apply.sh` skips it with `SKIP (no such project in this checkout)`
when the clone is not there yet.

Populated as the build needs it — see the *Known gaps* section of the
device tree README for the areas most likely to need more.
