# Kona clocks

TODO: write a nicer description for these. also, finish this past line ~1470?

## Clock types

The following clock types are available:

- CCU clock (controls a few clocks)
- Peripheral clock
- Bus clock (many peripherals also have bus clocks)
- Ref clock (in mainline these are just generic clocks)
- Core clock - ???
- PLL clock - ???
- PLL channel clock - ???
- Miscelaneous

## CCUs

CCU clocks control multiple clocks. The BCM21664 and BCM23550 have the following CCUs:

- KPROC / PROC_CCU (Processor clocks?)
- ROOT (Some core clocks)
- KHUB
- KHUBAON
- KPM (Master)
- KPS (Slave)
- MM
- MM2
- BMDM (Modem)
- DSP

## List of clocks per CCU

The category names for the clocks represent their name in the clocks.c file in downstream. Clock names without the ``clk_`` prefix are enclosed in a ``CLK_NAME()`` block in downstream.

### Root

- ``CLK_NAME(root)``
- Operations: ``&root_ccu_clk_ops`` (clock), ``&root_ccu_ops`` (CCU)

Note that most of this CCU's clocks are ref clocks, documented in the [Ref clocks](#ref-clocks) section.

#### dig_prediv

- Type: Peripheral clock
- CCU: [root](#root)
- Operations: ``&dig_prediv_peri_clk_ops``
- Source clocks: [crystal](#crystal), [pll0](#pll0), [pll1](#pll1)
- Purpose: **TODO**: what's the purpose?

Attached is the following comment:

```c
/* Dig_prediv clock: no physical clock as such, user has to make sure that
   he has enabled one of the channel clocks before set_rate for this clk.
   This is done to accomodate changing of chnl_clk freq independently */
```

#### dig_ch0

- Type: Peripheral clock
- CCU: [root](#root)
- Operations: ``&dig_ch_peri_clk_ops``
- Source clocks: [dig_prediv](#dig_prediv)

#### dig_ch1

- Type: Peripheral clock
- CCU: [root](#root)
- Operations: ``&dig_ch_peri_clk_ops``
- Source clocks: [dig_prediv](#dig_prediv)

#### dig_ch2

- Type: Peripheral clock
- CCU: [root](#root)
- Operations: ``&dig_ch_peri_clk_ops``
- Source clocks: [dig_prediv](#dig_prediv)

#### dig_ch3

- Type: Peripheral clock
- CCU: [root](#root)
- Operations: ``&dig_ch_peri_clk_ops``
- Source clocks: [dig_prediv](#dig_prediv)

### kproc

- ``CLK_NAME(kproc)``
- Operations: ``&proc_ccu_clk_ops`` (clock), ``&proc_ccu_ops`` (CCU)

Appears to control a few ARM processor-related clocks.

#### a9_pll

- Type: PLL clock
- CCU: [kproc](#kproc)
- Frequency: read dynamically
- Operations: ``&gen_pll_clk_ops``

#### a9_pll_chnl0

- Type: PLL channel clock
- CCU: [kproc](#kproc)
- PLL: [a9_pll](#a9_pll)
- Operations: ``&gen_pll_chnl_clk_ops``

#### a9_pll_chnl1

- Type: PLL channel clock
- CCU: [kproc](#kproc)
- PLL: [a9_pll](#a9_pll)
- Operations: ``&gen_pll_chnl_clk_ops``

#### arm_core

- Type: Core clock
- CCU: [kproc](#kproc)
- PLL: [a9_pll](#a9_pll)
- Operations: ``&gen_core_clk_ops``
- Frequency: In frequency table: 25, 52, 156 (twice), 312 (twice) MHz

#### arm_switch

- Type: bus clock
- CCU: [kproc](#kproc)
- Operations: ``&gen_bus_clk_ops``

#### cci

- Type: bus clock
- CCU: [kproc](#kproc)
- Operations: ``&gen_bus_clk_ops``

### Modem

- ``CLK_NAME(bmdm)``
- Operations: ``&gen_ccu_clk_ops`` (clock), ``&bmdm_ccu_ops`` (CCU)

### DSP

- ``CLK_NAME(dsp)``
- Operations: ``&gen_ccu_clk_ops`` (clock), ``&gen_ccu_ops`` (CCU)

### Ref clocks

#### crystal

- Type: ref clock
- Frequency: 26MHz
- Operations: ``&gen_ref_clk_ops``

#### clk_pll0

- Type: ref clock
- Frequency: 312MHz
- Operations: ``&gen_ref_clk_ops``
- Purpose: "624MHZ (const PLL)"

#### clk_pll1

- Type: ref clock
- Frequency: 312MHz
- Operations: ``&gen_ref_clk_ops``
- Purpose: "624MHZ (Root PLL - desensing)"

It's worth noting that both of these clocks have their frequency set to 312MHz rather than the claimed 624MHz from the comments.

#### clk_8phase_en_pll1

- Type: ref clock
- Frequency: 624MHz
- Operations: ``&en_8ph_pll1_ref_clk_ops``

There's an erratum that has something to do with this clock (``CONFIG_PLL1_8PHASE_OFF_ERRATUM``). This config option appears to be enabled on the Samsung Galaxy Grand Neo (baffinlite).

Also, this clock has its own initialization function (``en_8ph_pll1_clk_enable``).

**TODO**: perhaps this is the actual root cause of our clock failures?

#### frac_1m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 1000000
- Operations: ``&gen_ref_clk_ops``

#### ref_96m_varvdd

- Type: ref clock
- CCU: [root](#root)
- Frequency: 96000000
- Operations: ``&gen_ref_clk_ops``

Notably doesn't seem to have an enable/gating select mask?

#### ref_96m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 96000000
- Operations: ``&gen_ref_clk_ops``

#### var_96m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 96000000
- Operations: ``&gen_ref_clk_ops``

#### var_500m_varvdd

- Type: ref clock
- CCU: [root](#root)
- Frequency: 500000000
- Operations: ``&gen_ref_clk_ops``

#### ref_312m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 312000000
- Operations: ``&gen_ref_clk_ops``

#### ref_208m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 208000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### ref_156m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 156000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### ref_104m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 104000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### ref_52m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 104000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### ref_13m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 13000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### var_312m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 312000000
- Operations: ``&gen_ref_clk_ops``

#### var_500m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 500000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### var_208m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 208000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### var_156m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 156000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### var_104m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 104000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### var_52m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 52000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### var_13m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 13000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### dft_19_5m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 19500000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### ref_cx40_varcdd

- Type: ref clock
- CCU: [root](#root)
- Frequency: 153600000
- Operations: ``&gen_ref_clk_ops``

#### ref_cx40

- Type: ref clock
- CCU: [root](#root)
- Frequency: 153600000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### ref_1m

- Type: ref clock
- CCU: [root](#root)
- Frequency: 1000000
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

#### ref_32k

- Type: ref clock
- CCU: [root](#root)
- Frequency: 32768
- Operations: ``&gen_ref_clk_ops``
- **No masks given**

### Misc clocks

These are notably not under any CCUs. They have their own initialization function (``misc_clk_enable``).

#### clk_tpiu

- Type: misc clock
- Frequency: unspecified, seems to use clk_pll1?
- Depends on: [clk_pll1](#clk_pll1)? but only without erratum?
- Operations: ``&misc_clk_ops``
- Purpose: unknown, comment above it says "peri clk src list"

#### clk_pti

- Type: misc clock
- Frequency: unspecified, seems to use clk_pll1?
- Depends on: [clk_pll1](#clk_pll1)? but only without erratum?
- Operations: ``&misc_clk_ops``
- Purpose: unknown

#### xtal_dummy_clk

- Type: misc clock
- Frequency: unspecified, 26MHz?
- Operations: ``&xtal_dummy_clk_ops``
- Purpose: Workaround for a [Power Island](pi_mgr.md) quirk, as explained by this comment:

```c
/* HUB, AON, FABRIC run at 26Mhz by default, but it is not enough for
   some clocks inside the CCUs. For this purpose, we add a dummy clk
   which on enabling will raise a dfs request to these power domains.
   And, we add this clock as a dependence for those clocks that need
   the power domains to run at a higher freq. So whenever such clocks
   are enabled, dummy will be enabled and PIs will run at higher freq*/
```
