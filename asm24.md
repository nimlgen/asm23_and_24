AS_USB4_240129_85_00_00.bin

## SCSI parsing

CODE:393c -- mb_scsi_cmd_parser: parses scsi commands, including read and writes.
CODE:2e01 -- mb_on_scsi_read

## TLP
`FUN_CODE_ad95` -- +/- looks the same as 23.

## Sending scsi write to nvme
diff 0xa600 0x0 0x1
diff 0xa604 0x0 0x1
diff 0xa61a 0x0 0x20
diff 0xa628 0x0 0xeb
diff 0xa629 0x0 0xea
diff 0xa630 0x0 0x7

diff 0xb251 0x1 0x19 -- nvme sq doorbell
diff 0xb252 0x18 0x19 -- nvme cq head
diff 0xb351 0x1 0x19 -- mirror
diff 0xb352 0x18 0x19 -- mirror

bar0 addr is 0xd00000 (sets 0xd0 in bar0)
memspaces:
0xd00000 + 0x1008 8 -- req
0xd00000 + 0x100c 8 -- comp

usb.pcie_mem_req(0xd00000 + 0x1008, value=0x2, size=4)
usb.pcie_mem_req(0xd00000 + 0x100c, value=0x2, size=4)
print(usb.read(0xb200 + 0x51, 1))
print(usb.read(0xb200 + 0x52, 1))

## trash notes:
(DAT_EXTMEM_9000 & 1) == 0 -- old msb?

trampolines:
#0xe12b -- 0xb | 0xe4ed
#0xef42 -- 0xe | 
#0xe632 -- 


scsi cmd

r7:
0x8a - 0x0
0x28 - 3
0x88 - 0x2
0x2a - 1
DAT_EXTMEM_0a83 -- scsi cmd (r7)

mb_process_scsi_cmds

DAT_EXTMEM_07ea = 1; -- is write


DAT_INTMEM_3e = 


# usb ctrls?
DAT_INTMEM_6a -- 0xb or 0x1

  <!-- DAT_EXTMEM_9093 = 8;
  DAT_EXTMEM_9094 = 2; -->

DAT_INTMEM_6f -



close_to_scsi_parse: called with arg, arg (r7) put into DAT_EXTMEM_0a7d.



# user queues
intmem3b -- next queue entry (?)
intmem3d -- some size (?)
intmem3e -- wtf?

# regs????
hmm 8000 0x5ea
hmm 8000 0x60c
hmm d000 0xa80
hmm 8000 0xab0
hmm 8000 0x9008
hmm 8000 0x9408
hmm 8000 0x9808
hmm 8000 0x9c08
hmm f000 0xc004
hmm 8000 0xc534
hmm 8000 0xc8a2
hmm 8000 0xc9a2
hmm 8000 0xcec4
hmm 8000 0xcfc4
hmm 8000 0xd014
hmm 8000 0xd414
