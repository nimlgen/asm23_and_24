fw is `240417_85_00_00`

## intro

The interrupt table consists of the first two jumps:

```
CODE:0000  02 43 1a  LJMP  reset
CODE:0003  02 0e 5b  LJMP  int0
```

`reset` is executed once at power‑on. 
`int0` is invoked for every endpoint request.

---

## map

```
0x0000–0x5FFF  24 kB XRAM – Global controller state (temporal data, next SCSI offset, etc.).
               Offsets vary by firmware version; these notes refer to 240417_85_00_00.
0x6000–0x6FFF   4 kB Unused (zero‑filled, read‑only)
0x7000–0x7FFF   4 kB XRAM – SPI‑flash controller read/write buffer
0x8000–0x8FFF   4 kB XRAM – USB data‑OUT buffer; SCSI 0xE4 copies data here and triggers a transfer
0x9000–0x93FF   USB MMIO peripherals
0x9400–0x97FF          Mirror of 0x9000–0x93FF
0x9800–0x9BFF          Mirror of 0x9000–0x93FF
0x9C00–0x9DFF          Mirror of 0x9000–0x91FF
0x9E00–0x9FFF   512 B XRAM – USB control‑transfer buffer
0xA000–0xAFFF   4 kB XRAM – PCIe DMA window @ 0x0082_0000 (NVMe I/O SQ)
0xB000–0xB1FF   512 B XRAM – PCIe DMA window @ 0x0080_0000 (NVMe Admin SQ)
0xB200–0xB7FF   PCI MMIO (TLP engine, hard‑wired NVMe doorbell window)
0xB800–0xBFFF   2 kB XRAM
0xC000–0xCFFF   MMIO peripherals (UART, flash controller, timers, …)
0xD000–0xD3FF   1 kB XRAM
0xD400–0xD7FF          Mirror of 0xD000–0xD3FF
0xD800–0xDFFF   2 kB XRAM – SCSI status buffers (served via status endpoint)
0xE000–0xE2FF          Mirror of 0xD800–0xDAFF
0xE300–0xE7FF   MMIO peripherals
0xE800–0xE9FF   512 B XRAM
0xEA00–0xEBFF          Mirror of 0xE800–0xE9FF
0xEC00–0xEDFF          Mirror of 0xE800–0xE9FF
0xEE00–0xEFFF          Mirror of 0xE800–0xE9FF
0xF000–0xFFFF   4 kB XRAM – PCIe DMA window @ 0x0020_0000 (generic NVMe data buffer).
               The controller maps only 4 kB, but firmware uses 512 kB
               (addresses 0x0020_0000–0x0028_0000, step 0x4000).
```

---

## `int0`

Every USB‑endpoint request triggers the `int0` handler.
For nvme mode:
* Endpoints for both SCSI read and write fall through to `LAB_CODE_10e0`.
* SCSI‑command processing is entered at `LAB_CODE_112e`.

---

## intermediate return codes after scsi commands (proceed in `fill_scsi_to_usb_transport` and some layer above)

0x05 - Do nothing (release endpoints)
0x01 - Complete with a 0x8000‑byte read into the endpoint
0x09 - SCSI read 
0x0A - SCSI write

---

## write patch

1. Change the return code to 0x05 to exit and free all endpoints after the write completes.
2. Restore the list of free buffers (`0x0170–0x0174`).
3. Reset `0xCE6E` to accept the next write command.
4. Reset `0xCE40` to clear the DMA‑write offset.

---

## read path: 0x8000 vs dma read

The regular data in (0x81) endpoint does **not** reach `LAB_CODE_10e0` but is handled in `LAB_CODE_0fc6`. For these cases (and for flash reads), registers hold:
```
0x9093 = 0x08
0x9094 = 0x02
```
The largest transfer size observed is 0x400 bytes for this method.

! With the return code at `0x29ad` patched from `0x9` to `0x5`, the execution paths still differ between nvme and nvme. and this might be a reason based on what read-locations are chosen from. In this case non-nvme controller reads trash data from old buffer. The nvme controller seems to setup endpoints correctly (?). I tried to find some global-config regs to be set, but no luck.

## global state for writes

This is the minimal state in the global config that should be set to allow write path.

```python
self.exec_ops([WriteOp(0x54b, b'\x20'), WriteOp(0x54e, b'\x04'), WriteOp(0x5a8, b'\x02'), WriteOp(0x5f8, b'\x04'),
      WriteOp(0x7ec, b'\x01\x00\x00\x00'), WriteOp(0xc422, b'\x02')])
```

---

## on-controller usb buffers

Firmware appears to follow three code paths, depending on requested size:

* `< 0x400`
* `< 0x4000`
* `>= 0x4000`

Buffer collisions were reproduced when **writing 0x400** bytes and **reading 0x400** bytes. Writing **>0x400** and reading **0x400** does **not** expose the original data.

The current write path clears `0xCE40`, which selects the DMA‑window offset for the write.

---

## TLP

TLP transfers are limited to a 4‑byte payload; larger payloads fail (`usb.py:pcie_request`). Same as asm23.

---

## SCSI Implementation

* `CODE:393C` — `mb_scsi_cmd_parser`: parses SCSI commands (read, write, etc.).
* `CODE:2E01` — `mb_on_scsi_read`

### custom commands

```
CODE:3948  39 b9         addr       LAB_CODE_39b9
CODE:394a  e0            ??         E0h
CODE:394b  39 be         addr       LAB_CODE_39be
CODE:394d  e1            ??         E1h
CODE:394e  39 c3         addr       LAB_CODE_39c3
CODE:3950  e2            ??         E2h
CODE:3951  39 c8         addr       LAB_CODE_39c8
CODE:3953  e3            ??         E3h
CODE:3954  39 cd         addr       jumper_e4
CODE:3956  e4            ??         E4h
CODE:3957  39 d2         addr       jumper_e5
CODE:3959  e5            ??         E5h
CODE:395a  39 b4         addr       jumper_e6
CODE:395c  e6            ??         E6h
CODE:395d  39 d7         addr       LAB_CODE_39d7
CODE:395f  e8            ??         E8h
```

* **E5** — Write one byte to memory
* **E4** — Read up to 0xFF bytes from memory

##### `E1` — Write config

```python
# Write config 1
cdb = struct.pack('>BBB12x', 0xE1, 0x50, 0x00)
# Write config 2
cdb = struct.pack('>BBB12x', 0xE1, 0x50, 0x01)
```

##### `E3` — Write fw

```python
# Lower part
cdb = struct.pack('>BBI', 0xE3, 0x50, len(part1))
# Upper part
cdb = struct.pack('>BBI', 0xE3, 0xD0, len(part2))
```

##### `E8` — Reset Controller

```python
cdb = struct.pack('>BB13x', 0xE8, 0x51)  # reset
```

---

## Sending a scsi wr to nvme

### nvme ring buffer

```
diff 0xA600 0x00 0x01
diff 0xA604 0x00 0x01
diff 0xA61A 0x00 0x20
diff 0xA628 0x00 0xEB
diff 0xA629 0x00 0xEA
diff 0xA630 0x00 0x07
```

### Doorbell Update

```
diff 0xB251 0x01 0x19  # NVMe SQ doorbell
diff 0xB252 0x18 0x19  # NVMe CQ head
```

### BAR0 Addressing

BAR0 base is **0x00D0\_0000** (firmware sets 0xD0 in BAR0).

Memory spaces:

```
0xD0_0000 + 0x1008  (8 B) — request register
0xD0_0000 + 0x100C  (8 B) — completion register
```

```python
usb.pcie_mem_req(0xD00000 + 0x1008, value=0x2, size=4)
usb.pcie_mem_req(0xD00000 + 0x100C, value=0x2, size=4)
print(usb.read(0xB200 + 0x51, 1))
print(usb.read(0xB200 + 0x52, 1))
```

---

## Tools

Firmware patcher and trace‑point injection scripts are in **`patcher.py`**.
Also some interesting tracing points are commented out:
```python
# this will also contain diff for other reads inside dump_stats/reset_stats. so check the diff between scsi_read and no scsi_read.

reset_stats()
usb.scsi_read(sz, lba=0x1000 + i)
dump_stats()
```
