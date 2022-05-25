# A7/A8(X) quirks, in random order
### Based on Apple's XNU with some hints from m1n1 chicken bits code

### Register LUT
```
ARM64_REG_CYC_OVRD - S3_5_C15_C5_0
        CYC_OVRD_OK2PWRDN_FORCE_UP                              (2UL << 24)
        CYC_OVRD_OK2PWRDN_FORCE_DOWN                            (3UL << 24)
        CYC_OVRD_DISABLE_WFI_RETENTION                          BIT(0)

// Cyclone, Typhoon, Twister
ARM64_REG_CYC_CFG - S3_5_C15_C4_0
        CYC_CFG_DEEP_SLEEP                                      BIT(24)
        CYC_CFG_SKIP_INIT                                       BIT(30)

ARM64_REG_HID0 - S3_0_C15_C0_0
        HID0_DISABLE_LOOP_BUFFER                                BIT(20)
        HID0_IC_PREF_LIMIT_ONE_BRN                              BIT(25)
        HID0_DISABLE_PMULL_FUSE                                 BIT(33)
        HID0_IC_PREFETCH_DEPTH_SHIFT                            60
        HID0_IC_PREFETCH_DEPTH_MASK                             (7UL << HID0_IC_PREFETCH_DEPTH_SHIFT)

ARM64_REG_HID1 - S3_0_C15_C1_0
        HID1_DISABLE_CMP_BR_FUSION                              BIT(14)
        HID1_RCC_FORCE_ALL_IEX_L3_CLOCKS_ON                     BIT(23)
        HID1_RCC_DISABLE_STALL_INACTIVE_IEX_CTL                 BIT(24)
        HID1_DISABLE_LSP_FLUSH_WITH_CONTEXT_SWITCH              BIT(25)
        HID1_DISABLE_AES_FUSE_ACROSS_GROUP                      BIT(44)
        HID1_ENABLE_BR_KILL_LIMIT                               BIT(60)

ARM64_REG_HID2 - S3_0_C15_C2_0
        HID2_DISABLE_MMU_MTLB_PREFETCH                          BIT(13)

ARM64_REG_HID3 - S3_0_C15_C3_0
        HID3_DISABLE_COLOR_OPTION                               BIT(2)
        HID3_DISABLE_DC_ZVA_CMD_ONLY                            BIT(25)
        HID3_DISABLE_XMON_SNP_EVICT_TRIGGER_L2_STARVATION_MODE  BIT(54)

ARM64_REG_HID4 - S3_0_C15_C4_0
        HID4_DISABLE_DC_MVA                                     BIT(11)
        HID4_DISABLE_SPEC_LAUNCH_READ                           BIT(33)
        HID4_FORCE_NS_ORD_LD_REQ_NO_OLDER_LOAD                  BIT(39)
        HID4_DISABLE_DC_SW_L2_OPS                               BIT(44)

ARM64_REG_HID5 - S3_0_C15_C5_0
        HID5_DISABLE_HWP_LOAD                                   BIT(44)
        HID5_DISABLE_HWP_STORE                                  BIT(45)
        HID5_CRD_EDB_SNP_RESERVED_MASK                          (3UL << 14)
        HID5_CRD_EDB_SNP_RESERVED_VALUE                         (2UL << 14)
        HID5_ENABLE_DN_FIFO_READ_STALL                          BIT(54)
        HID5_DISABLE_FULL_LINE_WRITE                            BIT(57)

ARM64_REG_HID6 - S3_0_C15_C6_0
        HID6_DISABLE_CLOCK_DIV_GATING                           BIT(55)

ARM64_REG_HID7 - S3_0_C15_C7_0
        HID7_DISABLE_CROSS_PICK2                                BIT(7)
        HID7_DISABLE_NEX_FAST_FMUL                              BIT(10)

ARM64_REG_HID8 - S3_0_C15_C8_0
        HID8_DATASET_ID0_VALUE                                  (0xFUL << 4)
        HID8_DATASET_ID1_VALUE                                  (0xFUL << 8)
        HID8_DATASET_ID2_VALUE                                  (0xFUL << 56)
        HID8_DATASET_ID3_VALUE                                  (0xFUL << 60)
        HID8_WAKE_FORCE_STRICT_ORDER                            BIT(35)

ARM64_REG_HID9 - S3_0_C15_C9_0
        # Nothing :thinking:

ARM64_REG_HID10 - S3_0_C15_C10_0
        HID10_DISABLE_HWP_GUPS                                  BIT(0)

// Cyclone, Typhoon, Twister
ARM64_REG_HID11 - S3_0_C15_C13_0
        HID11_DISABLE_X64_NT_LAUNCH_OPTION                      BIT(1)
        HID11_DISABLE_FILL_C1_BUB_OPTION                        BIT(7)
        HID11_DISABLE_FAST_DRAIN_OPTION                         BIT(23)
```


## Initialization common to all Apple targets
```
ARM64_REG_HID4 |= HID4_DISABLE_DC_MVA | HID4_DISABLE_DC_SW_L2_OPS
```

## Cyclone/Typhoon-Specific initialization
### Disable LSP flush with context switch to work around bug in LSP that can cause Cyclone to wedge when CONTEXTIDR is written.
```
ARM64_REG_HID0 |= HID0_DISABLE_LOOP_BUFFER
ARM64_REG_HID1 |= HID1_DISABLE_STALL_INACTIVE_IEX_CTL

// This quirk applies only to A7
ARM64_REG_HID1 |= HID1_DISABLE_LSP_FLUSH_WITH_CONTEXT_SWITCH

ARM64_REG_HID3 |= HID3_DISABLE_XMON_SNP_EVICT_TRIGGER_L2_STARVATION_MODE
ARM64_REG_HID5 &= ~HID5_DISABLE_HWP_LOAD
ARM64_REG_HID5 &= ~HID5_DISABLE_HWP_STORE

// Change the default memcache data set ID from 0 to 15 for all agents
ARM64_REG_HID8 |= (HID8_DATASET_ID0_VALUE | HID8_DATASET_ID1_VALUE)
```

## WFI is giga broken on A7/A8. Disable it so we don't have to hack on Linux. With this setting the cores don't completely turn off, they are only clock-gated instead.
```
ARM64_REG_CYC_OVRD |= CYC_OVRD_OK2PWRDN_FORCE_UP
```

## If you want to hack your kernel, here's the 'proper' WFI setup
### First disable MMU prefetch:
```
ARM64_REG_HID2 |= HID2_DISABLE_MMU_MTLB_PREFETCH

# Make sure everything went through and the core will wake up next time around..
dsb sy
isb sy
```

### And then enable deep sleep & send OK to power down:
```
ARM64_REG_CYC_CFG = CYC_CFG_DEEP_SLEEP

ARM64_REG_CYC_OVRD |= CYC_OVRD_OK2PWRDN_FORCE_DOWN

dsb sy
isb sy

// XNU calls dsb sy once more, hmm..

wfi
```

### ..aaand it's time to return from WFI:
```
ARM64_REG_HID2 &= ~HID2_DISABLE_MMU_MTLB_PREFETCH

dsb sy
isb sy
```