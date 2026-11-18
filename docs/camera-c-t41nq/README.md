# Physical T41NQ / GC5603 upstream handoff — 2026-11-18

This is measured bring-up evidence from the owner's Jienuo camera C, not a generic firmware image or a claim that every T41NQ board is supported. Retail model/revision is not established well enough to publish a universal builder target. Related: [Majestic #376](https://github.com/OpenIPC/majestic/issues/376), [sensor selection #55](https://github.com/OpenIPC/openingenic/pull/55), [tuning #2892](https://github.com/OpenIPC/firmware/pull/2892), [NOR correction #262](https://github.com/OpenIPC/linux/pull/262).

## What now runs on physical hardware

A new native OpenIPC-family Linux 4.4.94 kernel and rebuilt native T41 modules/userspace are installed on NOR, with persistent SquashFS/JFFS2, the owner's existing key-only SSH identity and a separate old-kernel recovery bank. No UART was needed for this migration. This supersedes the earlier reports limited to native userspace on the old kernel. It is still a device-specific integration, not a stock nightly boot.

- SoC T41NQ, 128 MiB RAM, GC5603 at I2C0 `0x31`, CSI0/MCLK0; chip ID reads `0x56/0x03` at `0x03f0/0x03f1`.
- Native configuration uses **Marmot**, `sensorResetPin: 18`, `sensorPowerDownPin: -1`, `sensorI2cBus: 0`, `sensorMclk: 0`. This is the tested combination; do not turn it into a family-wide pin default. The earlier Tomcat/default-power-down combination did not register this sensor; a measured 27 MHz MCLK by itself was insufficient to establish correct setup.
- Memory split: `mem=74M@0x0 rmem=45M@0x4A00000 nmem=9M@0x7700000`.
- Load AVPU before registering the sensor; native `tx-isp-t41.ko isp_clk=300000000`, `direct_mode=0` (default), `isp_memopt=1` (default). An `isp_memopt=0` trial did not repair the image fault and was reverted. Run mdev/hotplug for frame-source nodes created during registration.
- Native audio module uses `spk_gpio=-1`; Majestic owns active-high speaker GPIO63. IR-cut GPIO50 is night/filter-out and GPIO49 is day/filter-in, bounded pulse with both low afterward. Do not simultaneously load an actuator driver that owns those pins.
- Wi-Fi: USB RTL8733BU `0bda:b733`; board power GPIO61, reset off/on before loading the matching driver. Keep this board-specific. WPA client, libnl dependencies and DHCP/DNS/time startup all need to be packaged, not just the kernel module.
- Ethernet is functional at 100 Mbps/full duplex. Wi-Fi-only key-pinned SSH and both AV streams were checked with Ethernet actually administratively down, then Ethernet was restored. Link-down route handling prevents same-subnet Ethernet from blackholing Wi-Fi. This is bounded path validation, not a claim of lossless physical cable failover.

Kernel provenance: OpenIPC/linux `ingenic-t41` source plus the exact SFC delta in #262. The built base commit `43cd7dbaf0ada9681eba0e867b7b43744472e9b9` has an identical tree to upstream `0d32657940518857e66129975389642a09498fe4`. The Marmot uImage is 1,917,906 bytes, SHA256 `97fbd27c196452ade4647bafd7b3e6ef68942f5e71dd21a7bdac31122aeabe86`. Native SDK module build input: OpenIPC/openingenic generic-GC5603 branch `f183cb39fa7126405decbd7b4f3f101fa7f8c369`. The sensor then received the explicitly described board experiment below. Majestic banner: T41 `master+2a4f463`. No T40/OpenIMP ABI bridge is used in this native installation.

## Important visual regression: selection/build success is not enough

The source-built GC5603 H20221117a module with the packaged Zeratul IQ file (`c7075ff33ba3ab562919134a934c0d18bacfd583506a3f28edcec31d3a8d7f58`) produced right-half blue horizontal corruption on this board. Both H.264 streams decoded successfully, including the bad image; the fault was reproduced in independent host captures, not just a browser view. Earlier successful Zeratul comparisons used the factory sensor module and **do not** validate this new source-driver/IQ pairing.

Sequential controlled-reboot experiments at unchanged 2880x1620:

| Driver / IQ combination | Physical visual result |
|---|---|
| Unmodified native sensor table + Zeratul IQ | Right-half stripes/blue corruption |
| Same with `isp_memopt=0` | Still bad; reverted |
| Native driver API/code, but exact ordered factory mode-0 register sequence + Zeratul IQ | Heavy stripes removed; purple/green split remained |
| Same factory-sequence native driver + camera's original IQ | Continuous scene without the former split or stripe fault, main and sub, also after another reboot |

This proves a working **combination** on this camera, not which individual register/IQ field caused the fault, and not that the sensor-table change alone is necessary with original IQ. The missing unmodified-table/original-IQ comparison has not been performed. Do not replace a family-wide default on this evidence alone.

Factory H20230720a mode 0 has 137 ordered register records including the terminator, versus 135 in the native source. `gc5603-register-comparison.json` lists the 16 differing final register values. The 124-byte MIPI bus structures were byte-identical, and both windows are 2880x1620 at 25/1. `gc5603-factory-mode0-experiment.patch` is the exact laboratory source delta: native API/driver code preserved, only the mode-0 table and a version marker changed. It is **factory-derived observed register data**, not represented as a public SDK release or a generic production patch. The ordered factory table's 548-byte SHA256 is `65d3f3ea4d566069ada8980c6406a3255b757ac700a15e15f12664cc1b5732f5`.

The original IQ is 254032 bytes, SHA256 `4a0f449761633a93fca82070a7829643e436d264e8b8a08ecef8821bcf87ca1d`. It is not uploaded: it came from product firmware, and the SDK source/provenance route should be agreed before packaging a replacement. Please identify a publicly released matching sensor/IQ pair, or review a board-specific variant rather than silently changing every GC5603. The two IQs share the `1.00`/`t41` envelope but about 105 KB differ; matching size/header does not prove visual compatibility.

## Validation and limits

After correction and reboot: main H.264 2880x1620 (configured 25 fps), sub H.264 640x360 (configured 5 fps), independently decoded and visually inspected. Configured rate is not a measured sustained-FPS assertion. Nonzero PCMA 8 kHz samples decoded with each stream; intelligible speech and latest-native physical speaker audibility were not independently reconfirmed. Supported `/play_audio` returns 200 and GPIO63 activates then returns low; that is not an acoustic success claim. Separate IR-cut pulses on 50 and 49 return both low and restore the original state; owner previously confirmed the optical polarity.

Full-resolution JPEG was temporarily enabled through the supported API, captured, decoded and visually inspected, then the owner's original disabled setting restored. The first immediate JPEG after enabling was effectively black; a later warmed frame showed the scene. Counting a decoded black image as a passing scene check would be incorrect. Main/sub/audio remained functional after restoration.

[Sanitized kernel/driver excerpts](native-boot-excerpts.txt) were captured over SSH from the new running kernel. They retain the SDK's CPU1 reset diagnostic and DT warnings rather than claiming warning-free boot. They are selected excerpts, not a stock UART log. Owner IP/MAC, passwords, keys, unique board IDs and private scene/audio files are not included.

## Flash/upgrade boundary

16 MiB GD25Q128C, JEDEC `c84018`; kernel table names this entry GD25Q127C. Stock environment occupies `0x3c000–0x3ffff`. The old driver advertised a 32 KiB erase unit, overlapping bootloader code if that environment were erased generically. #262 supplies the bounded 4 KiB driver correction and physical guard/readback evidence.

Stock bootargs described 256 KiB boot, 1600 KiB kernel, 3584 KiB root, 10368 KiB ipc, remainder appfs. The current **custom**, not recommended universal, layout is:

| Region | Offset | Size |
|---|---:|---:|
| Preserved boot code, RO | 0x000000 | 240 KiB |
| Environment | 0x03c000 | 16 KiB |
| Independent old-kernel recovery bank, RO | 0x040000 | 4 MiB |
| New native kernel, RO | 0x440000 | 2 MiB |
| New native SquashFS root, RO | 0x640000 | 7 MiB |
| Persistent JFFS2 overlay | 0xd40000 | 2.75 MiB |

Before a trial, U-Boot stores the old recovery route, then arms the hardware watchdog. Native boot commits itself only after persistent key-only SSH, writable overlay and local media/network health are up. A deliberately uncommissioned native boot actually fell back to old-kernel pinned SSH; subsequent commissioned native reboots passed. Qualified environment/tail cut-point and forward/reverse transaction models plus native 4 KiB scratch writes were checked separately. This is not a proof against every hardware/power failure or an upstream-ready universal updater.

The boot prefix and recovery kernel/root remain byte-identical to the pre-upgrade reference. One pre-reserved 4 KiB recovery diagnostic sector contains the expected CDG1 log. No recovery-kernel/rootdata is replaced by that log. Full corrected flash and pre-upgrade backups are private.

A WebUI message offering a generic `t41nq`/NOR nightly is **not** qualification for this layout, Marmot choice, sensor/IQ pairing or persistent management/recovery startup. Do not turn that match into an automatic upgrade recommendation. Generic nightly installation, generic fw_setenv/sysupgrade and HTTPS browser talkback are not validated here.

## What would make this maintainable upstream

1. Review/merge the bounded NOR erase change separately from board layout or environment transaction policy.
2. Keep #55's selection/build validation distinct from physical visual qualification; review the observed sensor-table differences and find an SDK-provenanced matching IQ pair.
3. Add a measured board profile in the appropriate builder/device layer: Marmot, the pins above, memory split, matching RTL8733BU and complete network dependencies. Do not put these pins into a shared T41 default.
4. Qualify an explicit migration/upgrade contract that preserves recovery and SSH for this board. The owner-specific experimental layout is evidence, not a proposed generic target layout.
5. Keep the native T41 Majestic path; the historical ABI bridge is no longer needed for this installation. Retain the known cutover and acoustic-validation limits reported in #376.
