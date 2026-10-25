# AudShim port: Dell Latitude 7320 Detachable (Tiger Lake)

Branch `dell7320-tgl`. Stage 1 = internal speakers. Stage 2 (microphone on link 3) comes next.

## Hardware compared with the XPS 13 Plus 9320

| | XPS 13 Plus 9320 | Latitude 7320 Detachable |
|---|---|---|
| Audio DSP | 8086:51CA (RPL-P) | **8086:A0C8 (TGL-LP)**, same `tgl_chip_info` in Linux |
| SOF firmware | `sof-rpl.ri` 2.2.0, ABI 3.22.1, IPC3 | **`sof-tgl.ri` 2.2.0, ABI 3.22.1, IPC3** |
| Topology | `sof-rpl-rt1316-l12-rt714-l0.tplg` | **`sof-tgl-rt711-l0-rt1316-l1-mono-rt714-l3.tplg`** |
| Speaker PCM | pcm 2 "Speaker", pipeline 2 | same |
| Amps | 2x RT1316, links 1 + 2, 'L,L' / 'R,R' | **1x RT1316, link 1, 'L,R'** (UCM rt1316.conf, 1 amp; confirmed with amixer) |
| Mic | RT714 link 0, PDI 2, DMIC3/4 | **RT714 link 3, PDI 2, DMIC3/4** (ALH stream 55) |
| Link 0 | RT714 | **RT711 jack codec** (left idle) |
| Link 2 | RT1316 | **empty** (DSDT template; bus clashes + clock-stop failure in Linux) |

## Stage 1 changes

- `AudShim.cpp`: `AUD_PCI_ID` = 0xA0C88086; `AUD_SPK_LINK_FIRST/LAST` = 1/1, `AUD_SPK_LINK_MASK` = 0x2,
  `AUD_SPK_RXSEL` = 0 ('L,R'), `AUD_EMPTY_LINK` = 2. All speaker loops, the bank switch, the RT1316 access check
  and the M5 register allowlist use them. Link 2 is not powered or touched. All safety caps are unchanged.
- The microphone code still looks for the RT714 on link 0. On this machine it finds none there (link 0 holds the
  RT711), so capture stays disabled in stage 1 ("no RT714 ... microphones unavailable" in the log).
- `Info.plist`: IOPCIMatch 0xa0c88086.
- `audctl.c`: `AUD_*` config block at the top (PCI id, 1 amp on link 1, 'L,R', mic link 3, default file names);
  speaker sanity check and `gen-header` emit `kDrvDai[1]`; mic sanity expects link 3 PDI 2.

## Build and test

Files: use the `sof-tgl.ri` and the `.tplg` that Linux loaded on this machine (`/lib/firmware/intel/sof/` and
`/lib/firmware/intel/sof-tplg/`), or the same names from sof-bin (`sof/intel-signed/sof-tgl.ri`).

```sh
mkdir -p ~/xe/build/audio/fw/use tools/build
cp PATH/sof-tgl.ri PATH/sof-tgl-rt711-l0-rt1316-l1-mono-rt714-l3.tplg ~/xe/build/audio/fw/use/
clang -O2 -Wall -Wextra -o tools/build/audctl tools/audctl.c -framework IOKit -framework CoreFoundation
tools/build/audctl tplg ~/xe/build/audio/fw/use/sof-tgl-rt711-l0-rt1316-l1-mono-rt714-l3.tplg
tools/build/audctl gen-header tools/AudShim/AudShimData.h
zsh tools/audio/install-audio.sh build
sudo diskutil mount disk0s1 && zsh tools/audio/install-audio.sh kext    # then reboot
tools/build/audctl info
tools/build/audctl drv test        # 3 quiet tones; expect "DRV TEST RESULT: SUCCESS"
log show --last 5m --predicate 'eventMessage CONTAINS "AudShim"' | tail -60
```

Start with low volume. Boot-arg `-audshimoff` disables the kext if anything goes wrong.
