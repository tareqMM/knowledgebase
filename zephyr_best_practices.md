## **Best Zephyr Practices For Better Code Quality**

### **Device Tree Best Practices**

- **Pull-ups on I2C Buses**:
  - If external pull-ups are too weak, enable internal pull-ups in the device tree.

  ```plaintext
  i2c0_default {
      group1 {
          psels = <NRF_PSEL(TWIM_SDA, 1, 5)>,
                  <NRF_PSEL(TWIM_SCL, 1, 7)>;
          bias-pull-up;
      };
  }
  ```

- **Reset Pin Activation**:
  - For NRF chips, the reset pin should be activated in the device tree if needed as follows:

  ```plaintext
  &uicr {
      gpio-as-nreset;
  }
  ```

- **Naming Conventions**:
  - Follow Zephyr's naming convention, especially for ‘compatible’.

    - **Example**:

    ```plaintext
    selfhold {
        compatible = "manufacturer,hardware_drivername";
    }
    ```

- **Indentation**:
  - Pay special attention to indentation, particularly in device tree files.

    ```plaintext
    <&adc 6>;
      <&adc 7>;
    ```

- **Flash Partitions**:
  - Note that all the information regarding partitions defined here will be ignored by Nordic SDK, as soon as a second image (i.e., mcuboot) is enabled.
  - Nordic has its own approach to define partitions. In case you want to know more about the approach from Nordic, you can have a look at this documentation: https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/scripts/partition_manager/partition_manager.html

  ```plaintext
  &flash0 {
      partitions {
          compatible = "fixed-partitions";
          #address-cells = <1>;
          #size-cells = <1>;
          ...
      };
  }
  ```

**Node label**:
- When using labels in the device tree, prefer `DT_NODELABEL()` over hardcoded paths when possible for clarity.

- **Vendor Information**:
  - Don’t forget to add vendor information at the end of `consteed.yaml`.

---

### **Implementation Best Practices**

- **Comment Style**:
  - Use `/* ... */` style comments instead of `//` comments, as per the C89 standard.
  - *Reference*: [Zephyr C Guidelines](https://docs.zephyrproject.org/latest/contribute/guidelines.html)

- **Include Zephyr Kernel**:
  - Prefer `#include <zephyr/kernel.h>` over `#include <zephyr/sys/util.h>` for a more Zephyr-aligned style.

- **Opaque Types**:
  - Avoid inspecting the content of opaque types (e.g., `timepoint.tick`).
  - *Reference*: [Zephyr sys_clock.h](https://github.com/zephyrproject-rtos/zephyr/blob/main/include/zephyr/sys_clock.h#L212)

- **Use CONTAINER_OF**:

    ```c
    struct net_time_timer {
        struct k_timer timer;

        /**
         * The currently programmed expiry time in terms of global time - required
         * to adjust the timer after a synchronization event and calculate
         * error-accumulation free counter periods.
         */
        net_time_t current_global_expiry_time_ns;

        /**
         * Period measured in global time for error-accumulation free calculation of
         * counter periods.
         */
        net_time_t period_ns;

        /**
         * Function to be executed upon timer expiry. This function must be initialized before
         * starting the timer. It runs in ISR context and it is immutable after timer
         * initialization.
         */
        void (*expiry_fn)(struct net_time_timer *timer, net_time_t time);
  };
    ```

  - **Good Example**: Use the `CONTAINER_OF()` macro to safely retrieve the parent structure.

    ```c
    static void system_timer_expiry_fn(struct k_timer *system_timer) {
        struct net_time_timer *timer = CONTAINER_OF(system_timer, struct net_time_timer, timer);

        if (timer->expiry_fn != NULL) {
            timer->expiry_fn(timer, timer->current_global_expiry_time_ns);
        } else {
            LOG_WRN("timer->expiry_fn is NULL for timer %p", timer);
        }

        schedule_next_timeout(timer);
    }
    ```

- **Bad Example**:
  - Manually casting a pointer to get the parent structure could break if the structure layout changes.

    ```c
    struct net_time_timer *timer = (struct net_time_timer *)system_timer;
    ```

- **System Initialization**:
  - Understand the parameters in `SYS_INIT()`, especially when using `PRE_KERNEL_1` and `CONFIG_KERNEL_INIT_PRIORITY_DEVICE`. This ensures proper initialization order:
    - The GPIO driver is initialized at `PRE_KERNEL_1` with priority `CONFIG_GPIO_INIT_PRIORITY`, so ensure dependencies are met.
    - `PRE_KERNEL_1` is suitable when kernel services aren't needed yet, allowing early setup.
    Have a look at [Initialization levels](https://docs.zephyrproject.org/apidoc/latest/group__sys__init.html)

- **Do Not Forget to Add the File `zephyr/module.yaml`**:
  - Typically, you always need this file to set at least `board_root`; otherwise, Zephyr will not find your board files. It also allows Zephyr to identify the project as a module. For further details, please check [Zephyr Modules Documentation](https://docs.zephyrproject.org/latest/develop/modules.html).

  ```plaintext
  build:
    kconfig: Kconfig
    cmake: .
    settings:
      board_root: .
      dts_root: .
  ```

- **Canceling Work**:
  - Prefer using `k_work_cancel_sync()` instead of `k_work_cancel` to ensure that any potentially ongoing work is completed before cancellation. 
  - *Note*: This change will affect whether a function is safe to call from an ISR. If `k_work_cancel_sync()` is used, the function can no longer be marked with `@funcprops \isr_ok`. Make sure to adjust the documentation accordingly.

  **Good Example**:

  ```c
  int sample_srv_stop_sampling(void)
  {
      int ret;
      struct k_work_sync work_sync;
      bool was_pending;

      ret = clk_srv_timer_stop(&timer);
      if (ret == -EINVAL) {
          LOG_ERR("Failed to stop the timer!");
          return ret;
      }

      was_pending = k_work_cancel_sync(&adc_all_ch_sequence_work, &work_sync);
      if (was_pending) {
          LOG_DBG("adc_all_ch_sequence_work was pending and has been canceled.");
      } else {
          LOG_DBG("adc_all_ch_sequence_work was not pending or already completed.");
      }
  }
  ```

  **Bad Example**:

  ```c
  was_pending = k_work_cancel(&adc_all_ch_sequence_work);
  ```

- **Use `DT_FOREACH_PROP_ELEM()`**:
  - Simplify device tree element retrieval using `DT_FOREACH_PROP_ELEM()`.

  **Bad Example**:

  ```c
  static const struct gpio_dt_spec multiplexer_gpios[] = {
      GPIO_DT_SPEC_GET_BY_IDX(MULTIPLEXER_CTRL_NODE, gpios, 0), /* ref-en-gpios */
      GPIO_DT_SPEC_GET_BY_IDX(MULTIPLEXER_CTRL_NODE, gpios, 1), /* meander-en-gpios */
      GPIO_DT_SPEC_GET_BY_IDX(MULTIPLEXER_CTRL_NODE, gpios, 2), /* ntc-en-gpios */
      GPIO_DT_SPEC_GET_BY_IDX(MULTIPLEXER_CTRL_NODE, gpios, 3)  /* bat-en-gpios */
  };
  ```

  **Good Example**:

  ```c
  #define MULTIPLEXER_CTRL_NODE DT_NODELABEL(multiplexer_control)
  #define MUX_GPIO_SPEC_GET(node_id, prop, idx) GPIO_DT_SPEC_GET_BY_IDX(node_id, prop, idx),
  static const struct gpio_dt_spec multiplexer_gpios[] = {
      DT_FOREACH_PROP_ELEM(MULTIPLEXER_CTRL_NODE, gpios, MUX_GPIO_SPEC_GET)
  };
  ```

- **Sensor Trigger Structure**:
  - Avoid allocating the `sensor_trigger` structure on the stack. Refer to the documentation for `sensor_trigger_set()` for proper usage.

  **Good Example**:

  ```c
  static const struct sensor_trigger trig = {
      .type = SENSOR_TRIG_DATA_READY,
      .chan = SENSOR_CHAN_ACCEL_XYZ,
  };
  ```

- **Include Path Management**:
  - Manage include paths properly by using `target_include_directories()` in `CMakeLists.txt`.

  **Bad Include**:

  ```c
  #include "../../drivers/sensor/lsm6dsm/lsm6dsm.h"
  ```

  **Better CMake Configuration**:

  ```cmake
  target_include_directories(app PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/..)
  ```

  **Improved Include**:

  ```c
  #include "drivers/sensor/lsm6dsm/lsm6dsm.h"
  ```

- **Avoid Duplicate Settings**:
  - Avoid having duplicate settings in different configuration files (e.g., `consteed_defconfig`, `.conf` files).
  - *Note*: `CONFIG_I2C` and `CONFIG_SPI` should be part of `prj.conf`, as other applications (like mcuboot) may not use I2C or SPI.
  - Similarly, serial and console should only be included if required in production.

- **Function Documentation**:
  - If a function can be called from an ISR (Interrupt Service Routine), add `@funcprops \isr_ok` in the documentation.
  - **Good Example**:

    ```c
    /**
     * @brief Start a drift-compensated timer with a specified period as measured
     * by the global clock.
     *
     * @details The timer's resolution follows the underlying local clock's
     * resolution. Timeouts will be rounded up to the next local clock tick. The
     * timer implementation ensures that rounding errors cannot accumulate over
     * time.
     *
     * Timer start and stop are not synchronized. It is the caller's responsibility
     * to protect against races. Timer memory is owned by the client. Calling
     * clk_srv_timer_start() transfers exclusive access to the timer to the clock
     * service.
     *
     * @funcprops \isr_ok
     *
     * @param timer The timer to start.
     * @param period The period for the timer in net_time_t format (drift
     * compensated, nanoseconds).
     *
     * @return 0 on success, negative error code on failure.
     */
    int clk_srv_timer_start(struct net_time_timer *timer, net_time_t period);
    ```
  - Note that `@param[in]`/`@param[out]`/`@param[in,out]` is only relevant for pointer arguments.
---

### **Logging and Debugging**

- **Log Module Flexibility**:
  - Avoid specifying a log level in `LOG_MODULE_REGISTER()` unless necessary. This allows users to set log levels globally using `LOG_DEFAULT_LEVEL`.

    ```c
    LOG_MODULE_REGISTER(selfhold_control);  // Omitting LOG_LEVEL_INF for flexibility
    ```

- **Log Messages**:
  - Log at the end of timing-critical functions to avoid unnecessary delays.
  - Add the name of the device being logged for better clarity.
    - **Example**:

      ```c
      LOG_ERR("Failed to configure %s pin %d (%d)", clock_sync_gpio_dt_spec.port->name, clock_sync_gpio_dt_spec.pin, err);
      ```

  - Avoid odd line breaks in log messages.

- **Printf Format**:
  - Use `%" PRId64 "` instead of `%lld` for printing 64-bit integers.
    - **Example**:

      ```c
      LOG_DBG("sync_signal_net_time: %" PRId64 " ns", net_time);
      ```

---

### **Miscellaneous**

- **Debouncing Input Pins**:
  - Always debounce input pins and buttons. See the implementation in `charging_srv` and `button_srv` from the `consteed` project.

- **Modifying Zephyr Sensor Drivers**:
  - To add private channels to a Zephyr sensor driver, modify two files:
    1. `include/zephyr/drivers/sensor/lsm6dsm.h` (create if it does not exist)
    This is a "private" include file of the driver. This file should never be included by any other component.
    This needs to go in include/zephyr/drivers/sensor/lsm6dsm.h inside this repository.
    ```c
    target_include_directories(app PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/../include)
    // Then you can include it like:
    #include <zephyr/drivers/sensor/lsm6dsm.h>
    ```
    2. `drivers/sensor/st/lsm6dsm/lsm6dsm.c`

  - Refer to this [commit example](https://github.com/GCX-EMBEDDED/continental-consteed-application/pull/74/commits/c082592b628984bece51dc74062d037c6c7e8831#diff-102561cf74a1dfec9b018f1901c271104d22147390323f787a239e0733ba6f40).

  - **Enum Usage Example**:

    ```c
    enum sensor_channel_lsm6dsm {
        SENSOR_CHAN_LSM6DSM_ACCEL_XYZ_RAW = SENSOR_CHAN_PRIV_START,
        SENSOR_CHAN_LSM6DSM_GYRO_XYZ_RAW,
    };
    ```

  - Add documentation similar to the example in the [Zephyr TMAG5273 documentation](https://github.com/zephyrproject-rtos/zephyr/blob/main/include/zephyr/drivers/sensor/tmag5273.h#L20-L22).

  - **Casting Example**:
    You need also to cast the new enums to avoid build and static analysis Warnings 
    ```c
    ret = sensor_channel_get(imu_dev, (int)SENSOR_CHAN_LSM6DSM_ACCEL_XYZ_RAW, data.accel);
    // or
    ret = sensor_channel_get(imu_dev, (enum sensor_channel)SENSOR_CHAN_LSM6DSM_ACCEL_XYZ_RAW, data.accel);
    ```
