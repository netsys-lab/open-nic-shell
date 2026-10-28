# Scitra plugin for OpenNIC Shell (Alveo U55C)

This plugin offloads SCION-IP translation and checksum calculation for
[Scitra-TUN](https://github.com/lschulz/scion-cpp/tree/main/scitra) into the
250 MHz user box of OpenNIC Shell. The P4 code is developed in
[netsys-lab/scitra-alveo](https://github.com/netsys-lab/scitra-alveo) (`p4/`).

## Requirements

* Vivado 2023.1
* Vitis Networking P4 (SDNet P4)
* UltraScale+ CMAC license (free)
* `hcam_base` and `hcam_advanced` licenses

Evaluation licenses are sufficient.

## Building

The design targets the Alveo U55C with one CMAC port and one physical function.

```bash
cd script
vivado -mode batch -source build.tcl -tclargs -board au55c \
       -num_cmac_port 1 -num_phys_func 1 \
       -user_plugin $(realpath ../plugin/scion_p4) \
       -impl 1 -post_impl 1 \
       -impl_strategy Performance_BalanceSLRs
```

A full build takes 4-5 hours. The results end up in
`build/au55c/open_nic_shell/open_nic_shell.runs/impl_1/`:

* `open_nic_shell.bit`: bitstream for JTAG programming
* `open_nic_shell.mcs`: image for the configuration flash
* `open_nic_shell_timing_summary_routed.rpt`: check that WNS and WHS are >= 0

To rebuild an existing project, add `-rebuild 1`.

`-impl_strategy Performance_BalanceSLRs` is required. With the default strategy the
design misses timing: all four P4 blocks and the CMAC end up in SLR1, and the
parser of the ingress translator fails by about 70 ps. The `-impl_strategy` option
is an addition to `build.tcl` in this branch; without it the shell's default
strategy is used.

## Programming the card

Program `open_nic_shell.mcs` into the configuration flash, then cold reboot the
host (power off, not just a warm reboot). A cold reboot is required because the
PCIe BAR2 size differs from the stock shell (see below), and the host only picks up
the new BAR size on enumeration.

Use the modified driver from
[scitra-alveo/open-nic-driver](https://github.com/netsys-lab/scitra-alveo/tree/main/open-nic-driver).

## Design overview

```
RX:  CMAC -> ingress classifier (P4) -> ingress checksum calculator
          -> ingress translator (P4) -> FIFO 0 -> arbiter -> QDMA C2H

TX:  QDMA H2C -> egress translator (P4) -> AXI-Stream switch
        M0 (forward):  -> egress checksum calculator -> egress checksum (P4) -> CMAC
        M1 (route back to host): -> FIFO 1 -> arbiter -> QDMA C2H
```

* `tuser_size` is carried through the whole pipeline. QDMA uses it as the C2H
  descriptor length, so packets that are routed back to the host keep the length
  set by the egress translator.
* The switch routes on the egress translator's `axis_tdest` metadata field.
* The checksum calculators compute the raw one's-complement sum of the payload
  (from the L4 header to the end of the packet) in network byte order. The P4
  blocks add the pseudo header and invert the result.
* The plugin drives the TX sideband to the CMAC packet adapter
  (`tuser_dst = 0x0040`, i.e. CMAC port 0).

### Files

| File | Content |
|---|---|
| `p2p_250mhz.sv` | Top level of the 250 MHz box: P4 blocks, FIFOs, switch, metadata wiring |
| `ingress_checksum_calculator.sv` | Payload checksum on the RX path |
| `calculator_UDP_chksm_egress.sv` | Payload checksum on the TX path |
| `axi_stream_arbiter.sv` | Merges RX traffic and routed-back TX traffic into QDMA C2H |
| `*.p4` | The four P4 programs (generated, see below) |
| `build_box_250mhz.tcl` | Creates the P4 IPs, FIFOs and switch, reads the RTL |
| `box_250mhz/` | Plugin instantiation, AXI-Lite address map and crossbar |
| `box_322mhz/`, `p2p_322mhz.sv`, `build_box_322mhz.tcl` | Unchanged from the stock `p2p` plugin |

### Metadata layout

The Vitis Net P4 metadata ports are packed structs: the first field of the P4
struct sits in the most significant bits.

| Block | Width | Layout |
|---|---|---|
| Ingress classifier | 33 | `tuser_size[32:17] is_scion[16] hop_fields[15:10] payload_offset[9:0]` |
| Ingress translator | 39 | `tuser_size[38:23] is_scion[22] hop_fields[21:16] payload_chksum[15:0]` |
| Egress translator | 59 | `axis_tdest[58] tuser_size[57:42] payload_offset[41:32] l4_src[31:16] l4_dst[15:0]` |
| Egress checksum | 16 | `payload_chksum[15:0]` |

If the metadata structs in the P4 code change, the slices in `p2p_250mhz.sv` have to
be updated to match.

## Address map

Changes to the stock shell (`src/system_config/`, `src/qdma_subsystem/`):

* Box 0 (250 MHz) spans `0x100000`-`0x4FFFFF`, box 1 (322 MHz) moves to `0x500000`.
* PCIe BAR2 grows from 4 MB to 8 MB on all physical functions.

Inside box 0 (`box_250mhz/box_250mhz_axi_crossbar.tcl`), offsets relative to
`0x100000`:

| Offset | Size | Block |
|---|---|---|
| `0x000000` | 512 KB | Egress translator |
| `0x080000` | 256 KB | Ingress classifier |
| `0x0C0000` | 128 KB | Ingress translator |
| `0x0E0000` | 4 KB | Unused (dummy slave) |

## Updating the P4 code

The `.p4` files in this folder are copies of the generated files in
`scitra-alveo/p4/build`. After changing the P4 code there, run `make -C p4` in
`scitra-alveo` and copy the four files here:

```bash
cp <scitra-alveo>/p4/build/{ingress-classifier,ingress-translator,egress-translator,egress-checksum}.p4 \
   plugin/scion_p4/
```

Then rebuild, and check the metadata layout above if any metadata struct changed.
