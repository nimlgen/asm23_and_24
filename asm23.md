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
```
  if (cEXTMEM0002 == '*') {
    pbVar5 = (byte *)CONCAT11(DAT_EXTMEM_0007 - (((0xe9 < DAT_EXTMEM_0008) << 7) >> 7),
                              DAT_EXTMEM_0008 + 0x16);
    DAT_INTMEM_4f = *pbVar5;
    DAT_INTMEM_50 = pbVar5[1];
  }
  else {
    pbVar5 = (byte *)CONCAT11(DAT_EXTMEM_0007 - (((0xe4 < DAT_EXTMEM_0008) << 7) >> 7),
                              DAT_EXTMEM_0008 + 0x1b);
    DAT_INTMEM_4f = *pbVar5;
    DAT_INTMEM_50 = pbVar5[1];
  }
```

TODO: they should do writes here (nvme write command is byte 01)

## Custom scsi commands
CODE:a5f1 -- mb_read_e4_cmd
CODE:b6b3 -- mb_write_e5_cmd

## TLP sender (the function we use)
`FUN_CODE_8b25` -- looks excatly the way we send tlp with scsi. We use custom scsi commands which just memwrite (`mb_write_e5_cmd`).
