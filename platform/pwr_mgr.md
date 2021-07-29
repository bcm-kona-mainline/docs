# Kona Power Manager (pwr_mgr)

**Important:** this page is still very work-in-progress, and may contain false information.

The Kona power manager driver handles various power management-related tasks. There exist two revisions of the power manager; one presumably for pre-Hawaii platforms and one for Hawaii + post-Hawaii platforms[^1]. 

The power manager has an I2C interface which is used by the PMU[^2]. There's also an I2C sequencer that's initialized by the power manager, which appears to be the backend for this interface.

## Initialization sequence

Initialization is started in ``arch/arm/mach-(hawaii/java)/pwr_mgr.c``, where the ``hawaii_pwr_mgr_init`` functions performs a few preparations before the power manager and the sequencer can be initialized. After that, it calls the ``pwr_mgr_init`` function from ``arch/arm/plat-kona/pwr_mgr.c``. Once it's done, it starts initializing the clocks.

## Communication with the PMU

### I2C sequencer

The I2C sequencer is initialized in ``pwr_mgr.c:pwr_mgr_init_sequencer``.

## Optional features

### CONFIG_KONA_I2C_SEQUENCER_LOG

Presumably enables a log for the I2C sequencer. See ``pwr_mgr.c:pwr_mgr_seq_log_buf_init`` and ``pwr_mgr.c:pwr_mgr_seq_log_buf_put``.

[^1]: as per [line 2301 of pwr_mgr.c]( https://github.com/knuxdroid/android_kernel_samsung_baffinlite/blob/39a5251f3ffd996a5826d101b93a98362768fc7e/arch/arm/plat-kona/pwr_mgr.c#L2301)
[^2]: see the [bcmpmu59xxx-i2c.c](https://github.com/knuxdroid/android_kernel_samsung_baffinlite/blob/cm-12.1/drivers/mfd/bcmpmu59xxx-i2c.c) driver.
