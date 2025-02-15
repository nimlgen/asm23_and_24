AS_PCIE_201012_91_00_00.bin
it's 8051.

## SCSI parsing

CODE:6be0 -- mb_scsi_cmd_parser: parses scsi commands, including read and writes.
CODE:0914 -- mb_on_scsi_read
CODE:6649 -- mb_on_scsi_write

Tables which looks like that to jump into cmds:
```
       CODE:79d4 7a 8f           addr       mb_reset_e1_cmd_entry
       CODE:79d6 e1              ??         E1h
       CODE:79d7 7a 98           addr       mb_reset_e2_cmd_entry
       CODE:79d9 e2              ??         E2h
       CODE:79da 7a a1           addr       mb_reset_e3_cmd_entry
       CODE:79dc e3              ??         E3h
       CODE:79dd 7a aa           addr       mb_reset_e4_cmd_entry
       CODE:79df e4              ??         E4h
       CODE:79e0 7a b3           addr       mb_reset_e5_cmd_entry
       CODE:79e2 e5              ??         E5h
       CODE:79e3 7a 7d           addr       LAB_CODE_7a7d
       CODE:79e5 e6              ??         E6h
       CODE:79e6 7a bc           addr       mb_reset_e8_cmd_entry
       CODE:79e8 e8              ??         E8h
```

### mb_on_scsi_write_cmd:

Different write scsi commands parsing (seems they support 2 of them write10 and write16):
TODO: they should do writes here (nvme write command is byte 01)

## Custom scsi commands
CODE:a5f1 -- mb_read_e4_cmd
CODE:b6b3 -- mb_write_e5_cmd

## TLP
`FUN_CODE_8b25` -- looks excatly the way we send tlp with scsi. We use custom scsi commands which just memwrite (`mb_write_e5_cmd`).


When cfg tlp is called:
0x5ef-0x5f2 - device address
0x5f3 - first byte enable

read_pci_cfg_16b_into_addr_r6r7 -- read 16b of cfg and put into 0x8000


mem tlp:
0x5f9-0x5fc - mem address

### pci

0xd00000 or 0xe00000 look to be tha bars.
0x800000/0x808000 or 0x900000/0x908000 -- nvme admin queues.

admin req queue ptr is in 0x50e-0x510...
admin cq_doorbell: 0x511-0x513 (?)
0x51a???

FUN_CODE_8f16 -- hmmmm


CODE:c437 -- looks it returns the bar address based

internal:
0x5e8 -- something device related? it's used to get correct bar address?


### user queues creation path

Prepreqs:
0) bar setup... see addressed above
1) tlp to init admin queues and enable nvme...

UserQueue Creation:
We need NVME_ACMD_CREATE_SQ=1, NVME_ACMD_CREATE_CQ=2.

Let's look at NVME_ACMD_CREATE_SQ, they are similar (all submits look the same):
It's `CODE:8f16`.

* Started with a call to `CODE:a55e` which sets up address for the command. and zeroes the cmd.
* Fill the command inside the main function: 0x820000 or 0x920000 seems to be the queues addresses.
* Call to `CODE:ba61`
    * Sets doorbell `DAT_INTMEM_6c + 1U & 3`
    * Calls to `CODE:c90a`.
        * Sets some pci regs: DAT_EXTMEM_b296 = 4; DAT_EXTMEM_b251 = param_1 (the value of a doorbell???); DAT_EXTMEM_b254 = 1(??); wait_pci; DAT_EXTMEM_b296 = 4;
    * Calls to `CODE:9450` which looks like a loop waiting for a completion.
