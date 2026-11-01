# Recover monitor audio after framebuffer connection changes

On the tested system, both monitor audio engines initially registered with
AppleHDA and appeared in CoreAudio. They disappeared after login, near the
NVIDIA framebuffer's first-frame connection-change event for a rotated display.
This timing does not establish that rotation is the only possible trigger.

## Change

`triggerNvidiaFramebufferEvents` previously required all three conditions:

```cpp
fb && !sink && gNotifiedFb[i] != fb
```

The last condition prevented the same framebuffer from being announced again
after its audio sink was removed. The retry build uses:

```cpp
fb && !sink
```

Controller identity, readiness, framebuffer existence, and missing-sink checks
remain in place. The existing `gfxMatchedHandler` and `readyToSendFBMsg` hooks
can now send another `framebufferEventGate` event 2 for that framebuffer.
There is no new timer or polling loop. Recovery still depends on an existing
callback running again; this does not guarantee recovery from every teardown.

## Confirmed test

One user confirmed working monitor audio after installing this build and
restarting on November 1, 2026:

| Component | Observed configuration |
| --- | --- |
| macOS | Sequoia 15.8.2, build 24H232, x86_64 |
| GPU | NVIDIA GeForce RTX 5070 |
| NVIDIA audio PCI ID | `10DE:2F80` |
| Lilu | 1.7.3 |
| Displays | ASUS VG258QM and ASUS VG249; one rotated 270 degrees |

After the restart, `debug.nmhda` recorded successful initial sink creation and
later successful reannouncements for both pin/converter pairs (4/8 and 5/9).
CoreAudio listed both monitor outputs at 48 kHz, with one selected as the
default output. The user reported that audio worked.

Sleep/wake, repeated hot-plug cycles, additional GPUs, and macOS Tahoe have
not been validated for this change. Existing private AppleHDA offsets and
function signatures are unchanged. The retry build retains upstream's bundle
version metadata; its filename and SHA-256 distinguish it from the original.

Other configurations may need a modified patch. You may need an AI coding
assistant to help adapt it using your own logs and the matching driver source.
Review and test any changes on the target machine with a verified backup;
AI assistance alone does not establish compatibility.

## Download and install

The tested bundle is in
[`Release/NMHDA-framebuffer-retry.zip`](../Release/NMHDA-framebuffer-retry.zip).
Check the archive against
[`Release/NMHDA-framebuffer-retry.sha256`](../Release/NMHDA-framebuffer-retry.sha256):

```sh
cd Release
shasum -a 256 -c NMHDA-framebuffer-retry.sha256
```

The executable inside the extracted `NMHDA.kext` has SHA-256:

```text
8e42ae319b0de0e7399536ee57b55b45d85e9e3e807217f41441f48e7e4d3237
```

1. Identify the EFI that actually boots OpenCore. Back up its complete `EFI`
   folder and the current `NMHDA.kext` before making changes.
2. Extract the retry archive and replace `EFI/OC/Kexts/NMHDA.kext` with the
   extracted bundle. Follow the prerequisites in the main README.
3. Confirm that OpenCore's `Kernel -> Add` enables Lilu before NMHDA. An
   existing NMHDA entry with the same bundle and executable paths can be reused.
4. Unmount the EFI and restart. Wait at least 30 seconds after login, then test
   each monitor output and inspect `sysctl debug.nmhda`.

To roll back, restore your backed-up `NMHDA.kext` to that same EFI and restart.
Keep a boot environment that can access the EFI if macOS cannot start. This
driver also supports the `-nmhdaoff` boot argument to disable its plugin.

## Build the same source

The change is based on upstream commit
`558dc644996fdeb05ff4f80b4e112527d384b9fe` from
[`emirercan3459/nullmoth-nvidia-audio`](https://github.com/emirercan3459/nullmoth-nvidia-audio).
The tested compiler was Apple clang 17.0.0 (`clang-1700.0.13.5`).

From a checkout containing this fix, with Apple's Command Line Tools installed:

```sh
git clone https://github.com/acidanthera/Lilu.git Lilu_repo
git -C Lilu_repo checkout 0249c3a918c68a43891350f35fded615b1575776
git clone https://github.com/acidanthera/MacKernelSDK.git MacKernelSDK
git -C MacKernelSDK checkout 3f750085caa17ec3a7880f11c11bf4f48cd6a164
LILU_DIR="$PWD/Lilu_repo/Lilu" \
MAC_KERNEL_SDK_DIR="$PWD/MacKernelSDK" \
bash ./build.sh
```

If those dependency directories already exist, reuse them and check out the
recorded commits instead of cloning again. Explicit dependency paths also let
the existing build script run repeatedly. The script writes its normal
`Release/NMHDA.kext` and versioned zip; the separately named retry archive in
this branch preserves the exact tested bundle.

Validation completed: original and patched source built with the same tools;
the bundle is a Mach-O x86_64 kext; its plist validates; kernel start/stop and
`kmod_info` symbols are present; undefined symbols match the original release;
disassembly confirms removal of the prior-notification comparison; and the
installed executable matched the SHA-256 above.

The source diff can be submitted upstream independently of the binary archive.
Original project credits and copyright notices are retained.
