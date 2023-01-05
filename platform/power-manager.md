
* Base address: `0x35010000`

## Revisions

There exist two revisions of the Power Manager: REV0 (capri) and REV2 (hawaii, java). Here is a list of known differences:

* A second set of voltage controllers is introduced in REV02.
	* To go with the second VO, a second sequencer command bank is introduced.
* A new i2c command, `PWRMGR_I2C_APB_READ_OFFSET`, is introduced in REV02. This register contains i2c data and i2c isr register... whatever those are.

## Components

* Some kind of interrupt controller?
* Power Island (mostly in separate pi_mgr driver)
* Sequencer (might be related to those sequences mentioned in PI driver)
* Some kind of PMU, or connection to the PMU
* Some kind of voltage controller?
* Event handler

## Initialization

The initialization is first called during early init, through a platform-specific function called `hawaii_pwr_mgr_init`, located in the mach- folders.

First, this function initializes some initialization parameters for the pwrmgr from `pm_params.c` through `pm_params_init`. These are stored in the `pwrmgr_init_param` struct in the aforementioned file.

Then, it uses a set of dummy pointers defined in `pwr_mgr` to initialize the power manager with:

    /**
     * Load the dummy sequencer during power manager initialization
     * Actual sequencer will be loaded during pmu i2c driver
     * initialisation
     */

At this point, `pwr_mgr_init` is called. This function is defined in `plat-kona/pwr_mgr.c`, and marks the first time we enter the full platform driver.

The passed dummy struct we created earlier is a `pwr_mgr_info` struct. `pwr_mgr.info` is set to this value. This indicates to all further functions that the power manager has been initialized. `pwr_mgr.sem_locked` is set to `false`.

Next, the sequencer is disabled (`pwr_mgr_pm_i2c_enable(false)` - [[#I2C_ENABLE_OFFSET - 0x4100]]) and initialized with the provided data (`pwr_mgr_init_sequencer(info);`).

`pwr_mgr_init_sequencer`:
* writes the I2C commands from `info->i2c_cmds` using `pwr_mgr_pm_i2c_cmd_write`
* writes the data from `info->i2c_var_data` using `pwr_mgr_pm_i2c_var_data_write`
* writes command pointers for each volt ??? from `info->i2c_cmd_ptr` using `pwr_mgr_set_v0x_specific_i2c_cmd_ptr`

Back to `pwr_mgr_init`, it sets `pwr_mgr.i2c_seq_trg = 0` and calls the following:
* `init_completion(&pwr_mgr.i2c_seq_done);` - this sets up a [completion handler](https://embetronicx.com/tutorials/linux/device-drivers/completion-in-linux/) (`include/linux/completion.h`)
* `INIT_WORK(&pwr_mgr.pwrmgr_work, pwr_mgr_work_handler);` - this sets up a [work queue](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch04s08.html) (`kernel/workqueue.c`).

Then, it runs `pwr_mgr_mask_intr(PWRMGR_INTR_ALL, true);` to mask all interrupts by default.

Lastly, it requests an IRQ using `request_irq`, using `info->pwrmgr_intr` as the interrupt number, with the handler set to `pwr_mgr_irq_handler` ([ref](https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-part-13-interrupt-example-program-in-linux-kernel/)).

This is the end of the `pwr_mgr_init` function.

We return to the `mach-{}/pwr_mgr.c` file.

The next things that happen:

* `hawaii_pi_mgr_init` is called. This is described in the PI section.
* Some erratum stuff happens. Check later.
* A wakeup override is set for the MM (multimedia) CCU (core clock, handles stuff like camera, DSI, GPU) using `pwr_mgr_pi_set_wakeup_override(PI_MGR_PI_ID_MM, false /*clear */);`. This might be related to the workarounds in `pwr_mgr_init_sequencer`?
* `pwr_mgr_event_clear_events` is called on all events.
* `pwr_mgr_event_set` sets `SOFTWARE_2_EVENT` and `SOFTWARE_0_EVENT` to 1.

Then some pin control stuff happens:

```c
    pwr_mgr_set_pc_sw_override(PC0, false, 0);
    pwr_mgr_set_pc_clkreq_override(PC0, true, 1);

    /* make sure that PC1, PC2 software override
     * is cleared so that sequencer can toggle these
     * pins during retention/off/wakeup
     */
    pwr_mgr_set_pc_sw_override(PC1, false, 0);
    pwr_mgr_set_pc_sw_override(PC2, false, 0);
    pwr_mgr_set_pc_sw_override(PC3, false, 0);
    pwr_mgr_set_pc_clkreq_override(PC1, false, 0);
    pwr_mgr_set_pc_clkreq_override(PC2, false, 0);
```

Then we call `pwr_mgr_pm_i2c_enable(true);`.

Then, the event table is initialized:

```c
    /*Init event table */
    for (i = 0; i < ARRAY_SIZE(__event_table); i++) {
        pwr_mgr_event_trg_enable(__event_table[i].event_id,
                     __event_table[i].trig_type);

        cfg.policy = __event_table[i].policy_modem;
        pwr_mgr_event_set_pi_policy(__event_table[i].event_id,
                        PI_MGR_PI_ID_MODEM, &cfg);

        cfg.policy = __event_table[i].policy_arm;
        pwr_mgr_event_set_pi_policy(__event_table[i].event_id,
                        PI_MGR_PI_ID_ARM_CORE, &cfg);

        cfg.policy = __event_table[i].policy_arm_sub;
        pwr_mgr_event_set_pi_policy(__event_table[i].event_id,
                        PI_MGR_PI_ID_ARM_SUB_SYSTEM, &cfg);

        cfg.policy = __event_table[i].policy_aon;
        pwr_mgr_event_set_pi_policy(__event_table[i].event_id,
                        PI_MGR_PI_ID_HUB_AON, &cfg);

        cfg.policy = __event_table[i].policy_hub;
        pwr_mgr_event_set_pi_policy(__event_table[i].event_id,
                        PI_MGR_PI_ID_HUB_SWITCHABLE, &cfg);

        cfg.policy = __event_table[i].policy_mm;
        pwr_mgr_event_set_pi_policy(__event_table[i].event_id,
                        PI_MGR_PI_ID_MM, &cfg);

```

The following sets up some kind of event triggers... not that they seem to do much that the others don't. Only happens on java. Might be related to the 4 CPU cores as opposed to hawaii's 2?

```c
    /*Map core timer interrupts to COMMON_TIMER_3_EVENT and
    *COMMON_TIMER_4_EVENT events.
    *Set timer_wu_event_sel to 0x3 to map both slave and core timer
    *interrupts to these events*/
    reg_val = readl(KONA_CHIPREG_VA + CHIPREG_PERIPH_SPARE_CONTROL1_OFFSET);
    reg_val |= 0x3 <<
        CHIPREG_PERIPH_SPARE_CONTROL1_TIMER_WU_EVENT_SEL_SHIFT;
    writel(reg_val,

```

Then we init all PIs using `pi_init` on the PI recieved with `pi_mgr_get` on every ID.

Finally, we call `__clock_init()`. Then clock setup happens for CPU speed (only used on Hawaii chips with lower clocks.)

This is the end of the `hawaii_pwr_mgr_init` function.

There are two more late init functions:

* `hawaii_pwr_mgr_late_init` which runs `hawaii_pwr_mgr_delayed_init`, which in turn runs `pi_init_state` on every PI;
* `mach_init_sequencer` which initializes the sequencer with real values and "on return PWRMGR I2C block will be enabled".

## Events

Events are called on specific actions in the system. When they occur, they set the PI policies. There are multiple types of trigger mechanisms for events.

A full list of events is present at `arch/arm/mach-java/include/mach/pwr_mgr.h`. Not all of them trigger policy changes; the ones that do are defined in `arch/arm/mach-java/pwr_mgr.c`.

## Sequencer

There is code for both a HW sequencer and a SW sequencer. As evidenced by the comments, they cannot both run at once or timeouts will happen.

Downstream driver defines multiple erratas for these. In our case:
* CONFIG_KONA_PWRMGR_SWSEQ_FAKE_TRG_ERRATUM=y
* CONFIG_KONA_PWRMGR_SWSEQ_RETRY_WORKAROUND=y

This sequencer can control voltage requests and PC pin status.

### Sequencer commands

The sequencer has 2 command banks, each of which stores 64 commands. Each command consists of a 4-bit command number and an 8-bit value.

REV02 has 2 command banks, REV00 only has one.

| Address | Name | Description (as found in downstream kernel) | Used? |
|---------|------|-------------|--------|
| 0x0 | REG_ADDR | sets the next address for read/write data | Y |
| 0x1 | REG_DATA | executes a write to I2C controller via APB | F
| 0x2 | I2C_DATA | data to be written to PMU via I2C | F
| 0x3 | I2C_VAR | data returned for voltage lookup (payload is the index to table) | F
| 0x4 | WAIT_I2C_RETX | wait for retry from I2C control register | N
| 0x5 | WAIT_I2C_STAT | wait for good status (loop until good) | N
| 0x6 | WAIT_TIMER | wait for the number of clocks in the payload | Y
| 0x7 | END | stop and wait for new voltage request change | Y
| 0x8 | SET_PC_PINS | pc pins are set based on value and mask | Y
| 0x9 | SET_UPPER_DATA | sets the data in the upper byte of apb data bus | N
| 0xA | READ_FIFO | read i2c fifo | F
| 0xB | SET_READ_DATA | copy I2C read data to PWRMGR register | F
| 0xE | JUMP_VOLTAGE | jump to address based on current voltage request | Y
| 0xF | JUMP | jump to address defined in payload | F
(Y - yes in dummy, F - yes in full table, N - not at all)

## Power Island

The Power Islands are power domains managed by the Power Manager.

### What makes a Power Island

There are 6 [[#Power Islands]] as defined in the code. Each power island has a **policy**, and has a controllable frequency and latency(?).

### DFS and QoS

ref: Documentation/Broadcom/Kona_PM_DFS_and_QOS_API.txt in downstream kernel

DFS APIs are used to control the **frequency**, or to be exact, **OPP mode** (see [[#OPP types]]), of a specific PI. QoS APIs are used to control the **maximum wake-up latency**.

Both of these APIs work through a system of "requests"; clients can make requests by using `add_request` (this creates a "node" which is usually stored somewhere in the client's driver), change their value using `request_update` or remove the request by using `request_remove`.

### Initialization

Initialization begins in `mach-{}/pi_mgr.c`., where `hawaii_pi_mgr_init` is called. This does some errata stuff, runs `pi_mgr_init` and finally calls `pi_mgr_register` on every PI. This file defines the default settings for each PI.

`pi_mgr_init` is defined in `arch/arm/plat-kona/pi_mgr.c`. It does two things:
* Initializes memory for the pi_mgr data: `memset(&pi_mgr, 0, sizeof(pi_mgr));`
* Sets `pi_mgr.init = 1;`.

Most of the actual PI initialization is actually done in the power manager initialization sequence.

### Policies
From `arch/arm/mach-java/include/mach/pwr_mgr.h`:

```c
#define PM_OFF      0  /* Shutdown policy */
#define PM_RET      1  /* Retention policy */
#define PM_ECO      4  /* Economy */
#define PM_DFS      5  /* also RUN_POLICY; see [[#DFS and QoS]] */
#define PM_WKP      7  /* wakeup? */
```

`pwr_mgr_pi_set_wakeup_override` needs to be called before policies can be changed.

### Raw data

#### Nodes

* DFS (Dynamic Frequency Scaling?) and QOS (Quality of Service?)
* Both have some kind of requests (add_request, update_request in plat-kona/pi_mgr.c)

#### OPP types

:src: arch/arm/mach-java/include/mach/pi_mgr.h

OPP stands for "operating frequency". See [[#DFS and QoS]].

* 0 - XTAL
* 1 - ECONOMY
* 2 - NORMAL
* 3 - TURBO
* 4 - SUPER_TURBO

#### Power Islands

:src: arch/arm/mach-java/include/mach/pi_mgr.h

* MM
* HUB_SWITCHABLE
* HUB_AON
* ARM_CORE
* ARM_SUB_SYSTEM
* MODEM

Each has 3 states: ACTIVE, RETENTION, and SHUTDOWN; ARM cores have ACTIVE, SUSPEND, RETENTION and DORMANT.


## Raw data

### I2C value offsets

#### INTR_STATUS - 0x40A0

#### INTR_MASK - 0x40A4

#### EVENT_BANK1_OFFSET - 0x40A8

#### I2C_ENABLE_OFFSET - 0x4100

Writing 0x0000001 enables the sequencer, 0x00000000 disables it. See `pwr_mgr_pm_i2c_enable`.

#### I2C_CMD_BANK0_OFFSET - 0x4104

#### I2C_CMD_BANK1_OFFSET - 0x4280

#### I2C_SW_CMD_CTRL - 0x41B8

`PWRMGR_I2C_REQ_TRG_MASK`  - enables/disables software sequencer

#### I2C_APB_READ

**REV02 only**. 0x41CC, mask 0xFF, shift 0.
