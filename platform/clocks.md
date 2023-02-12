# Clocks

> Note: this document contains primarily information that I deemed useful while trying to add some things to the mainline driver. As such, it does not currently cover the things already defined in the mainline driver. For that, consult the source code. It's very well documented so it should be decently easy to read through.

**TODO**: KHUBAON_CLK_MGR_REG_PWRMGR? There's a pwrmgr clock??? Figure out if we need this one.

**TODO**:  arch/arm/plat-kona/clock.c line 3684: investigate.

TODO: figure out what's the exact deal with the AUTO_GATE flag (is it equivalent to a "hardware-only" gate in mainline?)

## CCUs

Clocks are split into CCUs (Clock Control Units). There are 10 CCUs for java:

* KPROC (or just PROC)
* ROOT
* KHUB
* KHUBAON
* KPM (master)
* KPS (slave)
* MM
* MM2 (java only)
* BMDM
* DSP

### Initialization

The main init function is `__ccu_clk_init`, called from `__clk_init`. It does the following:

* Enables the PI for the clock if the `CCU_ACCESS_ENABLE` flag is set. This only seems to happen for MM and MM2 slots.
* "Initailize CCU context save buf if CCU state save parameters are defined for this CCU." (`ccu_init_state_save_buf`)
* Calls the init function for the clock. (See `ccu_clk_init` below.)
* Does something to keep write access enabled if the `CCU_KEEP_UNLOCKED` flag is set. This seems to only happen for the root CCU.
* "Keep CCU intr disabled by default": `ccu_int_enable` (-> `ccu_clk_int_enable`) false for `ACT_INT`, `TGT_INT`

Downstream defines `gen_ccu_clk_ops` which contains references to init functions. The init function for CCUs is `ccu_clk_init`. A walkthrough of this function:

* Enable write access.
* Stop policy engine.
* Enable all policy masks by default (same as upstream `policy_init`).
* Initialize the voltage table (`ccu_set_voltage` -> `ccu_clk_set_voltage`).
* Initialize the peripheral voltage data (`ccu_set_peri_voltage` -> `ccu_clk_set_peri_voltage`)
* Initialize the frequency policy (`ccu_set_freq_policy` -> `ccu_clk_set_freq_policy`).
* Set ATL and AC.
* Disable write access.

### Policy engine

Allows for synchronous and asynchronous requests. Required to make changes to the trigger policy, voltage tables and frequency policy. See mainline comments in `__ccu_policy_engine_start`.

### Trigger policy

No idea what it does. Mainline initializes this in policy_init.

### Voltage tables

**TODO**: This information might be shared with the pwrmgr???

Here's an example voltage table from downstream (as defined in mach `pm_params.h`):

```c
#define PROC_CCU_FREQ_POLICY_TBL    \
        ARRAY_LIST(PROC_CCU_FREQ_ID_ECO, PROC_CCU_FREQ_ID_ECO,\
```

The voltage IDs are as follows:

```c
#define PROC_CCU_FREQ_ID_XTAL       0
#define PROC_CCU_FREQ_ID_52MHZ      1
#define PROC_CCU_FREQ_ID_156MHZ     2
#define PROC_CCU_FREQ_ID_ECO        4
#define PROC_CCU_FREQ_ID_NRML       6
#define PROC_CCU_FREQ_ID_TURBO      6
#define PROC_CCU_FREQ_ID_SUPER_TURBO    7
```

MM/MM2 voltage tables/values are different and use different offsets.

### Frequency policy

**NOT** the same thing as the trigger policy. At least I don't think it is. Either way, it's enabled separately.

### ATL and AC

Also mentioned in the Power Manager. ATL means "Automatic/Active(?) Target Load", AC means "auto-copy".

## PLL clocks

PLL clocks are *completely* different from the remaining clocks. They're primarily used for the PLL1 and PLL2 clocks for the ARM core clock.

A PLL clock can have multiple *channels*. These are a separate clock type in downstream.

### Common terms

Taken from the `clk-iproc-armpll.c` driver in mainline, I think this description should also apply here:

```
/*
 * The output frequency of the ARM PLL is calculated based on the ARM PLL
 * divider values:
 *   pdiv = ARM PLL pre-divider
 *   ndiv = ARM PLL multiplier
 *   mdiv = ARM PLL post divider
 *
 * The frequency is calculated by:
 *   ((ndiv * parent clock rate) / pdiv) / mdiv
 */
```

"frac_div" is the fraction of a **frac**tional **div**ider
"pdiv" is the **p**re-**div**ider
"ndiv" is the multiplier
"ndiv_int" is the **div**ider **int**eger
"xtal" is a static value of 26 MHz; probably linked to `ref_crystal` clock

### PLL clock operations

#### Initialization

* Enable CCU write access
* If the clock is auto-gated, set idle powerdown ON, else set it OFF (`pll_ctrl_offset`, `idle_pwrdwn_sw_ovrride_mask`)
* If the clock has a desense (`->desense`):
	* If flags contain `PLL_OFFSET_EN`, get the `pll_offset_offset` and set `OFFSET_MODE_MASK` on/off depending on SW mode; then call `__pll_set_desense_offset` with `des->def_delta` as the offset
	* Otherwise, clear the `pll_offset_offset` register and set it to `PLL_OFFSET_MODE_MASK`
* Disable CCU write access

#### Enabling

* Enable CCU write access
* If the idle powerdown sw override bit is set, return (clock is autogated)
* Unset the `pwrdwn_mask` in the `pwrdwn_offset`
* Set the `soft_post_resetb_mask` bit to the `soft_post_resetb_offset`
* Set the `pll_clk->soft_post_resetb_mask | pll_clk->soft_resetb_mask` bits to the `soft_resetb_offset`
* Wait until `pll_lock` bit is not set at the `pll_lock_offset` (max. 1000 retries)
* Disable CCU write access

#### Disabling
* Enable CCU write access
* If the idle powerdown sw override bit is set, return (clock is autogated)
* Set the `pwrdwn_mask` in the `pwrdwn_offset`
* Disable CCU write access

#### Setting desense offset
* Enable CCU write access, enable CCU power island if `CCU_ACCESS_ENABLE` flag is set
* If `PLL_OFFSET_EN` not present in desense flags, return early
* Get the current rate with `__pll_clk_get_rate` 
* Calculate the offset rate by adding the provided offset to the current rate
* Calculate the new rate by calling `compute_pll_vco_div` with the offset value, NULL, `&ndiv_off` and `&nfrac_off`
* If `abs` of `new_rate - off_rate` `> 100`, rate is not supported - bail out
* Get `nfrac` value by checking `ndiv_frac_offset` masked by `_mask` shifted by `_shift`
* Get `ndiv` value by checking `ndiv_pdiv_offset` masked by `int_mask` shifted by `int_shift`
* Get value from `CCU_REG_ADDR(ccu_clk, des->pll_offset_offset)`, and:
	* If `PLL_OFFSET_NDIV` flag is set - unset `PLL_OFFSET_NDIV_MASK` bit, set the new `ndiv_off` shifted by `PLL_OFFSET_NDIV_SHIFT` instead
		* Else if ndiv != ndiv_off error out
	* If `PLL_OFFSET_NDIV_FRAC` flag is set - unset `PLL_OFFSET_NDIV_F_MASK` bit, set the new `nfrac_off` shifted by `PLL_OFFSET_NDIV_F_SHIFT` instead
		* Else if nfrac != nfrac_off error out
	* Write the value back to the offset
* Disable CCU write access

#### Setting clock rate
* Calculate the new rate by calling `compute_pll_vco_div` with the provided new rate, `&pdiv, &ndiv_int, &nfrac`
*  If `abs` of `new_rate - off_rate` `> 100`, rate is not supported - bail out
* Enable CCU write access
* If `(pll_clk->cfg_ctrl_info && pll_clk->cfg_ctrl_info->thold_count)`:
	* Loop over all vco_thold members until we reach the largest one or the new rate is smaller than the vco_thold
		* When we do, set pll_cfg_ctrl to `cfg_ctrl->pll_config_value[inx]` << `ctrl_shift` & `ctrl_mask`
* Write nfrac to `ndiv_frac_offset`
* Write nint and pdiv to `ndiv_pdiv_offset`
* Set soft_post_resetb_mask bit at `soft_post_resetb_offset`
* Set soft_resetb mask bit at `soft_resetb_offset`
* Loop for lock bit if PLL is autogated or enabled
* Disable CCU write access

#### Getting clock rate
* If clock is fixed-rate, return clk->rate
* Get nfrac from `ndiv_frac_offset` (& mask >> shift)
* Get pdiv, ndiv_int from `ndiv_pdiv_offset`
	* If either are empty, get the `_max` value
* `frac_div = 1 + (pll_clk->ndiv_frac_mask >> pll_clk->ndiv_frac_shift);`
* Return output of compute_pll_vco_rate

#### Compute PLL VCO rate

* Get `xtal` with `clock_get_xtal()`
* Calculate initial rate using the formula `(ndiv_int * frac_div + nfrac) * xtal`
* Call `do_div` on the initial rate / `pdiv * frac_div` (this divides `a` by `b` and saves the result inline to `a` (returns the remains, but this is unused here))
* Return the new initial rate

Comment in code:
```
4400     /*
4401        vco_rate = 26Mhz*(ndiv_int + ndiv_frac/frac_div)/pdiv
4402        = 26(*ndiv_int*frac_div + ndiv_frac)/(pdiv*frac_div)
4403      */
```

#### Compute PLL VCO div

* Set `max_ndiv` to `ndiv_int_max`
* Set `frac_div` to 1 + frac_mask >> frac_shift
* Set `_ndiv_int` to `rate / xtal` (comment: `pdiv = 1`)
	* If it's larger than `max_ndiv`, set it to `max_ndiv`
* Set temporary fraction to `((u64)rate - (u64)_ndiv_int * xtal) * frac_div`
* Divide temporary fraction by `xtal`
* Save temporary fraction to `_nfrac`, set to &= `frac_div - 1`
* Calculate the temp rate using `compute_pll_vco_rate`
	* If this temp rate != rate, loop over every `_nfrac` smaller than `frac_div` until `temp_rate` is larger than `rate`, increasing `_nfrac` by 1 on every loop
		* If `temp_rate > rate`, set `temp` to the rate calculated using `compute_pll_vco_rate` with `_nfrac - 1`
		* `if (abs(temp_rate - rate) > abs(rate - temp))`, decrease nfrac by 1 and break
* Calculate the new rate using `compute_pll_vco_rate`
* If `_ndiv_int` == max_ndiv, set `_ndiv_int` to 0
* If they are set (evaluate to true), save `ndiv_int`, `nfrac` and `pdiv` to the pointers provided by the function call
* Return the new rate

### PLL channel operations

#### Initialization
Nothing.

#### Enabling
* Enable CCU write access
* Unset `out_en_mask` bit at `pll_enableb_offset`
* Set `load_en_mask` bit at `pll_load_ch_en_offset`
* Disable CCU write access

#### Disabling
* Enable CCU write access
* Set `out_en_mask` bit at `pll_enableb_offset`
* Disable CCU write access

#### Getting rate
* Get config reg value by reading `pll_chnl_clk->ccu_clk, pll_chnl_clk->cfg_reg_offset`
* Get mdiv value by getting the mdiv bit from the config reg value (mdiv_mask >> mdiv_shift)
	* If mdiv == 0, mdiv = mdiv_max
* Get rate of parent PLL clock
* Return rate of parent PLL clock / mdiv

#### Setting rate
* Get rate of parent PLL clock as `vco_rate`
* Get mdiv by dividing `vco_rate` by the desired rate
	* If mdiv == 0, mdiv++
	* If abs(rate - vco_rate / mdiv) > abs(rate - vco_rate / (mdiv + 1)), mdiv++
	* If mdiv > mdiv_max, mdiv = mdiv_max
* If `(abs(rate - vco_rate / mdiv) > 100)`, bail out (invalid rate)
* Enable CCU write access
* Write mdiv to `pll_chnl_clk->cfg_reg_offset`, with `mdiv_mask` << `mdiv_shift`
* Unset load_en_mask bit at `pll_chnl_clk->pll_load_ch_en_offset`

### Core clock operations

Core clocks technically have an enable bit and gating bit, but they're never set. There are functions for *getting* them, but not *setting* them. There are no explicit enable or disable functions. There's an init function but all it does is add the clock to some clock list somewhere, and does a sanity-check to ensure ccu_clk, pll_clk and num_chnls are all set.

#### Initialization
Nothing.

#### Getting rate
* Get frequency ID (ccu_get_freq_policy(core_clk->ccu_clk, core_clk->active_policy))
* If the frequency ID is in `->pre_def_freq`, return that item from the struct
* Otherwise, get the PLL channel = `freq_id - core_clk->num_pre_def_freq` and call `__pll_chnl_clk_get_rate(&core_clk->pll_chnl_clk[pll_chnl]->clk);`

#### Setting rate
* Set `vco_rate` to the rate of the PLL clock
* Set `div` to the vco rate divided by the desired rate
* If clock has `UPDATE_LPJ` flag:
	* Save first CPU's `loops_per_jiffy` value to `l_p_j_reg`
	* Set `l_p_j_ref_freq` to `clk->ops->get_rate(clk) / 1000;`
* If `div < 2` or `rate * div != vco_rate`, set the PLL clock rate to the desired rate * 2, then recalculate vco_rate and div
* Set last channel clock rate to the desired rate
* If this returns succesfully and `UPDATE_LPJ` flag is set:
	* For each online CPU, set the `loops_per_jiffy` value to `core_clk_freq_scale(l_p_j_ref, l_p_j_ref_freq, rate / 1000)`
	* Set global(?) loops_per_jiffy variable to `core_clk_freq_scale(l_p_j_ref, l_p_j_ref_freq, rate / 1000)`

#### core_clk_freq_scale

> "old * mult / div" calculation for large values (32-bit-arch safe)

Takes values `unsigned long old, u_int div, u_int mult`.
Effectively the same as `(old * mult) / div`.

```c
static inline unsigned long core_clk_freq_scale(unsigned long old, u_int div,
                        u_int mult)
{
#if BITS_PER_LONG == 32

    u64 result = ((u64)old) * ((u64)mult);
    do_div(result, div);
    return (unsigned long)result;

#elif BITS_PER_LONG == 64

    unsigned long result = old * ((u64)mult);
    result /= div;
    return result;

#endif

```

### ARM PLL bringup

The A9_PLL_CLK is used as the CPU frequency clock. It's initialized in `mach_config_arm_pll`, which is called in `pwr_mgr` init right after `__clock_init()`. The function, on the BCM23550, goes as follows:

* Get the A9 PLL clock
* Switch the KPROC CCU frequency policy to `PROC_CCU_FREQ_ID_ECO, PM_WKP`
* Set the A9 PLL clock rate to the "turbo rate" basec on the provided "turbo val"
* Switch the KPROC CCU frequency policy to `PROC_CCU_FREQ_ID_SUPER_TURBO, PM_WKP`

## Frequency tables and voltage tables

### KHUB
#### Frequency
```c
static u32 khub_clk_freq_list0[] = DEFINE_ARRAY_ARGS(26000000,26000000,26000000,26000000);
static u32 khub_clk_freq_list1[] = DEFINE_ARRAY_ARGS(52000000,52000000,52000000,52000000);
static u32 khub_clk_freq_list2[] = DEFINE_ARRAY_ARGS(104000000,104000000,52000000,52000000);
static u32 khub_clk_freq_list3[] = DEFINE_ARRAY_ARGS(156000000,156000000,78000000,78000000);
static u32 khub_clk_freq_list4[] = DEFINE_ARRAY_ARGS(156000000,156000000,78000000,78000000);
static u32 khub_clk_freq_list5[] = DEFINE_ARRAY_ARGS(208000000,104000000,104000000,104000000);
static u32 khub_clk_freq_list6[] = DEFINE_ARRAY_ARGS(208000000,104000000,104000000,104000000);
```
#### Voltage
```c
#define PROC_CCU_FREQ_VOLT_TBL  \
        ARRAY_LIST(VLT_ID_A9_ECO, VLT_ID_A9_ECO,\
            VLT_ID_A9_ECO, VLT_ID_A9_ECO, VLT_ID_A9_TURBO,\
        VLT_ID_A9_NORMAL, VLT_ID_A9_TURBO, VLT_ID_A9_SUPER_TURBO)
```
#### Frequency policy
```c
#define HUB_CCU_FREQ_POLICY_TBL \
        ARRAY_LIST(HUB_CCU_FREQ_ID_ECO, HUB_CCU_FREQ_ID_ECO,\
		HUB_CCU_FREQ_ID_NRML, HUB_CCU_FREQ_ID_NRML)
```

### KHUBAON
#### Frequency
```c
static u32 khubaon_clk_freq_list0[] = DEFINE_ARRAY_ARGS(26000000,26000000);
static u32 khubaon_clk_freq_list1[] = DEFINE_ARRAY_ARGS(52000000,52000000);
static u32 khubaon_clk_freq_list2[] = DEFINE_ARRAY_ARGS(78000000,78000000);
static u32 khubaon_clk_freq_list3[] = DEFINE_ARRAY_ARGS(104000000,52000000);
static u32 khubaon_clk_freq_list4[] = DEFINE_ARRAY_ARGS(156000000,78000000);
```
#### Voltage
```c
/*AON is on fixed voltage domain. Voltage ids does not really matter*/
#define AON_CCU_FREQ_VOLT_TBL   \
        ARRAY_LIST(VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO,\
            VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO,\
            VLT_ID_OTHER_ECO)
```
#### Frequency policy
```c
#define AON_CCU_FREQ_POLICY_TBL \
        ARRAY_LIST(AON_CCU_FREQ_ID_ECO, AON_CCU_FREQ_ID_ECO,\
            AON_CCU_FREQ_ID_NRML, AON_CCU_FREQ_ID_NRML)
```
#### Peripheral voltage
```c
DEFINE_ARRAY_ARGS(VLT_NORMAL_PERI,VLT_HIGH_PERI)
```

### KPM
#### Frequency
```c
static u32 kpm_clk_freq_list0[] = DEFINE_ARRAY_ARGS(26000000,26000000,26000000);
static u32 kpm_clk_freq_list1[] = DEFINE_ARRAY_ARGS(52000000,52000000,26000000);
static u32 kpm_clk_freq_list2[] = DEFINE_ARRAY_ARGS(104000000,52000000,26000000);
static u32 kpm_clk_freq_list3[] = DEFINE_ARRAY_ARGS(156000000,52000000,26000000);
static u32 kpm_clk_freq_list4[] = DEFINE_ARRAY_ARGS(156000000,78000000,39000000);
static u32 kpm_clk_freq_list5[] = DEFINE_ARRAY_ARGS(208000000,104000000,52000000);
static u32 kpm_clk_freq_list6[] = DEFINE_ARRAY_ARGS(312000000,104000000,52000000);
static u32 kpm_clk_freq_list7[] = DEFINE_ARRAY_ARGS(312000000,156000000,78000000);
```
#### Voltage
```c
#define KPM_CCU_FREQ_VOLT_TBL   \
        ARRAY_LIST(VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO,\
            VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO,\
            VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO,\
            VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO)
```
#### Frequency policy
```c
#define KPM_CCU_FREQ_POLICY_TBL \
        ARRAY_LIST(KPM_CCU_FREQ_ID_ECO, KPM_CCU_FREQ_ID_ECO,\
            KPM_CCU_FREQ_ID_NRML, KPM_CCU_FREQ_ID_NRML)
```
#### Peripheral voltage
```c
DEFINE_ARRAY_ARGS(VLT_NORMAL_PERI,VLT_HIGH_PERI)
```

### KPS
#### Frequency
```c
static u32 kps_clk_freq_list0[] = DEFINE_ARRAY_ARGS(26000000,26000000,26000000,26000000,26000000);
static u32 kps_clk_freq_list1[] = DEFINE_ARRAY_ARGS(52000000,26000000,26000000,26000000,26000000);
static u32 kps_clk_freq_list2[] = DEFINE_ARRAY_ARGS(78000000,39000000,39000000,39000000,39000000);
static u32 kps_clk_freq_list3[] = DEFINE_ARRAY_ARGS(104000000,52000000,52000000,52000000,52000000);
static u32 kps_clk_freq_list4[] = DEFINE_ARRAY_ARGS(156000000,52000000,52000000,52000000,52000000);
static u32 kps_clk_freq_list5[] = DEFINE_ARRAY_ARGS(156000000,78000000,78000000,78000000,78000000);
```
#### Voltage
```c
#define KPS_CCU_FREQ_VOLT_TBL   \
        ARRAY_LIST(VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO,\
            VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO,\
            VLT_ID_OTHER_ECO, VLT_ID_OTHER_ECO)
```
#### Frequency policy
```c
#define KPS_CCU_FREQ_POLICY_TBL \
        ARRAY_LIST(KPS_CCU_FREQ_ID_ECO, KPS_CCU_FREQ_ID_ECO,\
            KPS_CCU_FREQ_ID_NRML, KPS_CCU_FREQ_ID_NRML)
```
#### Peripheral voltage
```c
DEFINE_ARRAY_ARGS(VLT_NORMAL_PERI,VLT_HIGH_PERI)
```
