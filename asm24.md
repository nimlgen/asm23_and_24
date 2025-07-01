240417_85_00_00

## intro

Int table is first 2 jmps:
CODE:0000  02 43 1a      LJMP       reset
CODE:0003  02 0e 5b      LJMP       int0

reset is called once.
int0 is called for every endpoint request.

#### map

```
0x0000-0x5FFF: 24 kB XRAM: Contains global internal state of the controller. Includes data like: temporal data, next scsi offset, etc. Offsets vary from version to version, so 240417_85_00_00 is used.
0x6000-0x6FFF: 4 kB of unused address space (zero-filled, read-only)
0x7000-0x7FFF: 4 kB XRAM (SPI flash controller read/write buffer)
0x8000-0x8FFF: 4 kB XRAM: usb data out buffer, 0xe4 special command copies data into this buffer and fires transfer.
0x9000-0x93FF: USB MMIO peripherals
0x9400-0x97FF: Mirror of MMIO 0x9000-0x93FF
0x9800-0x9BFF: Mirror of MMIO 0x9000-0x93FF
0x9C00-0x9DFF: Mirror of MMIO 0x9000-0x91FF
0x9E00-0x9FFF: 512 B XRAM (USB control transfer buffer)
0xA000-0xAFFF: 4 kB XRAM, PCIe DMA address: 0x00820000 (NVMe I/O Submission Queue)
0xB000-0xB1FF: 512 B XRAM, PCIe DMA address: 0x00800000 (NVMe Admin Submission Queue)
0xB200-0xB7FF: PCI MMIO peripherals (tlp, hardwired nvme doorbell window)
0xB800-0xBFFF: 2 kB XRAM
0xC000-0xCFFF: MMIO peripherals (UART, flash controller, timers, etc.)
0xD000-0xD3FF: 1 kB XRAM
0xD400-0xD7FF: Mirror of XRAM 0xD000-0xD3FF
0xD800-0xDFFF: 2 kB XRAM: scsi status buffers (status endpoint transfers this)
0xE000-0xE2FF: Mirror of XRAM 0xD800-0xDAFF
0xE300-0xE7FF: MMIO peripherals
0xE800-0xE9FF: 512 B XRAM
0xEA00-0xEBFF: Mirror of XRAM 0xE800-0xE9FF
0xEC00-0xEDFF: Mirror of XRAM 0xE800-0xE9FF
0xEE00-0xEFFF: Mirror of XRAM 0xE800-0xE9FF
0xF000-0xFFFF: 4 kB XRAM, PCIe DMA address: 0x200000 (NVMe generic data buffer). The window which is mapped on the controller. The real dma size is 512kb (based on the addresses passed to nvme: 0x200000 - 0x280000. step is 0x4000)
```

#### int0
every request endpoint hits the int handler.
in nvme mode:
1) endpoints for both scsi read and write falls to LAB_CODE_10e0
2) the scsi command processing is called from LAB_CODE_112e
3) with patched 0x9 -> 0x5 in 0x29ad, paths are still different for nvme/no-nvme. (should be state difference)

#### scsi return codes
* 0x5 - do nothing (releases endpoints)
* 0x1 - completes with 0x8000 read into the endpoint
* 0x9 - scsi read
* 0xa - scsi write

#### scsi write patch
1) change the return code to 5 (save to exit and free all endpoints, since write has been issued).
2) restore the list free buffers (0x170-0x174 range).
3) reset 0xce6e to be able to accept next write commands.
4) reset 0xce40 to clear dma write offset.

#### read path 0x8000 vs scsi read difference
The regular (data_in 0x81) endpoint does not fall to LAB_CODE_10e0 part. And is proceed in LAB_CODE_0fc6 sections. For this case (and flash read) 0x9093=8 and 0x9094=2. The max output size I could get working is 0x400.

#### usb buffers allocation
it seems there are 3 paths: <0x400, <0x4000, >=0x4000. I could get buffer colliding writing (0x400) and reading (0x400), but writing (>0x400) and reading (0x400) will never see the original data.
Also there is a gate in the current write path: which clears 0xce40. 0xce40 is responsible into what offset dma-window the write will go into.

## TLP
TLP is limited to 4bytes payload, can't get working larger payloads. (usb.py:pcie_request)

## SCSI
CODE:393c -- mb_scsi_cmd_parser: parses scsi commands, including read and writes.
CODE:2e01 -- mb_on_scsi_read

#### Custom commands

These are some custom scsi commands:
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

e5 write 1 byte into mem
e4 read (up to 0xff bytes) from mem

e1 write config:
`cdb = struct.pack('>BBB12x', 0xe1, 0x50, 0x0)` -- write config
`cdb = struct.pack('>BBB12x', 0xe1, 0x50, 0x1)` -- write config 2

e3 write fw
`cdb = struct.pack('>BBI', 0xe3, 0x50, len(part1))` -- lower part
`cdb = struct.pack('>BBI', 0xe3, 0xd0, len(part2))` -- higher part

e8 reset
`cdb = struct.pack('>BB13x', 0xe8, 0x51)` -- reset controller

## Sending scsi write to nvme

writes into the ring:

diff 0xa600 0x0 0x1
diff 0xa604 0x0 0x1
diff 0xa61a 0x0 0x20
diff 0xa628 0x0 0xeb
diff 0xa629 0x0 0xea
diff 0xa630 0x0 0x7

update doorbell:

diff 0xb251 0x1 0x19 -- nvme sq doorbell
diff 0xb252 0x18 0x19 -- nvme cq head

bar0 addr is 0xd00000 (sets 0xd0 in bar0)
memspaces:
0xd00000 + 0x1008 8 -- req
0xd00000 + 0x100c 8 -- comp

usb.pcie_mem_req(0xd00000 + 0x1008, value=0x2, size=4)
usb.pcie_mem_req(0xd00000 + 0x100c, value=0x2, size=4)
print(usb.read(0xb200 + 0x51, 1))
print(usb.read(0xb200 + 0x52, 1))
