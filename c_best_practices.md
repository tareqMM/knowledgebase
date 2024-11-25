## **Best C Practices For Better Code Quality**

- **Imperative Function Naming**:
  - When documenting functions, use clear action words like "Initialize", "Start", and "Stop" to describe what the function does.

- **Unused Definitions**:
  - After making code changes, review your code to remove unused `#define` statements and `#include` directives.

- **Static Initialization**:
  - Variables such as `static size_t current_sample_count = 0;` are automatically initialized to `0`. Avoid redundant initializations.

- **Global Variables and Functions**:
  - Use the `static` keyword for global variables and internal functions to limit their scope to the current file.

- **typedef**:
  - Avoid using `typedef` when declaring `struct`s to improve clarity and alignment with best practices.

- **Magic Numbers**:
  - **Bad Example**: Using magic numbers in your code without explanation.

    ```c
    int duration = period * 1000 * 1000; // convert to nanoseconds
    ```

  - **Good Example**: Use descriptive constants instead of magic numbers.

    ```c
    #define MS_TO_NS(ms) ((ms) * 1000000LL)
    int duration = MS_TO_NS(period);
    ```

  - *Note*: Use `NSEC_PER_MSEC` instead of the literal number.

- **Ignoring a Return Value**:
  - If you are ignoring a return value like `(void)k_work_reschedule(dwork, K_SECONDS(...));`, please add a comment explaining why:

    ```c
    /* Cannot fail because delay is not K_NO_WAIT. */
    (void)k_work_reschedule(dwork, K_SECONDS(next_held_evt_dur_rel));
    ```

- **Best Practice for Floats in Embedded Systems**:
  - On processors with an FPU (Floating Point Unit), most floating-point operations are efficient.
    - Addition, subtraction, multiplication, and integer-to-float conversion are generally not performance bottlenecks.
    - Avoid divisions when possible, as they are computationally expensive. For example, multiply by `0.1` instead of dividing by `10`.
    - Enable the `-Wdouble-promotion` GCC compiler warning to catch implicit promotions to double.

- **Switch Case Best Practices**:
  - Use `switch` statements instead of multiple `if-else` when handling enums. This enables the compiler flag `-Wswitch-enum` to warn if an enum case is missed.
  - Assume you have a state machine and you need to add a specific timeout value for each state:

    ```c
    enum ctrl_srv_state {
        CTRL_SRV_STATE_STARTUP,
        CTRL_SRV_STATE_UPGRADE_IN_PROGRESS,
        CTRL_SRV_STATE_CHARGING,
        CTRL_SRV_STATE_BUTTON,
        CTRL_SRV_STATE_IDLE_NOT_CONNECTED,
        CTRL_SRV_STATE_IDLE_CONNECTED,
        CTRL_SRV_STATE_CONFIG,
        CTRL_SRV_STATE_SAMPLING_PRESSURE,
    };
    ```

  - **Not very nice Example**:
    - Its not nice because you always need to update the value of the define each time you add a new state

    ```c

          #define CTRL_SRV_STATE_TOTAL_COUNT 8
          static state_timeout[CTRL_SRV_STATE_TOTAL_COUNT] = {
              [CTRL_SRV_STATE_STARTUP] = 0,
              [CTRL_SRV_STATE_UPGRADE_IN_PROGRESS]= 0,
              [CTRL_SRV_STATE_CHARGING]= 0,
              [CTRL_SRV_STATE_BUTTON]= 0,
              [CTRL_SRV_STATE_IDLE_NOT_CONNECTED]= 600,
              [CTRL_SRV_STATE_IDLE_CONNECTED]= 0,
              [CTRL_SRV_STATE_CONFIG]= 0,
              [CTRL_SRV_STATE_SAMPLING_PRESSURE]= 0,
          }
    ```

  - If you still want/need to use an array, you can avoid the #define by doing like this:

    ```c

    enum display_srv_state {
      DISPLAY_BASE_NOTHING,
      DISPLAY_BASE_CHARGING,
      DISPLAY_BASE_CONFIG,
      DISPLAY_BASE_UPGRADE_IN_PROGRESS,
      DISPLAY_BASE_MOUNTING_LOCATION,
      DISPLAY_OVERLAY_BATTERY,
      DISPLAY_OVERLAY_WARN_WIPE,
      DISPLAY_OVERLAY_WIPE,
      /*To be replaced by DISPLAY_OVERLAY_PAIRING*/
      DISPLAY_OVERLAY_SYNC_LEADER,
      DISPLAY_OVERLAY_POWERON_POWEROFF,

      /** Number of entries in @ref display_srv_state - intentionally last. */
      DISPLAY_STATE_COUNT,
    };

    ```

  - **Nice Example**:

  ```c
    static unsigned int ctrl_srv_get_state_timeout(enum ctrl_srv_state state)
        {
            switch (state) {
            case CTRL_SRV_STATE_STARTUP:
            case CTRL_SRV_STATE_UPGRADE_IN_PROGRESS:
            case CTRL_SRV_STATE_CHARGING:
            case CTRL_SRV_STATE_BUTTON:
            case CTRL_SRV_STATE_IDLE_CONNECTED:
            case CTRL_SRV_STATE_CONFIG:
            case CTRL_SRV_STATE_SAMPLING_PRESSURE:
                break;

            case CTRL_SRV_STATE_IDLE_NOT_CONNECTED:
                return ;
            }

            return 0;
            }
  ```

- **Passing parameter as a pointer**:
  - **Bad Example**:

    ```c
    void sample_agg_send_sample(struct sample_data data)
    ```

  - **Good Example**:

    ```c
    void sample_agg_send_sample(const struct sample_data *data);
    ```

- **Passing parameter as a copy**:
  - It is just one single value. Please pass by value.
  - **Bad Example**:

    ```c
    void ctrl_srv_consume_battery_level(uint8_t *battery_level)
    ```

  - **Good Example**:

    ```c
    void ctrl_srv_consume_battery_level(uint8_t battery_level)
    ```

- **Error Handling**:
  - For functions that return error codes, return the precise `errno` instead of generic values like `EINVAL`. For example:

    ```c
    err = sample_srv_start_sampling(MS_TO_NS(period_ms));
    if (err != 0) {
        shell_error(shell, "Failed to start periodic sampling! (%d)", err);
        return -EINVAL;
        /* EINVAL is "invalid argument", but the error message suggests that something other than 
         * the argument was the issue. Since you have the precise errno in err,
         * why not just return that?
         */
    }
    ```

- **Explicit Boolean Conversion**:
  - Use explicit boolean conversion for clarity:

    ```c
    static int ctrl_srv_startup(bool *button_down, bool *charger_connected)
    {
    int ret;
    ret = charger_srv_init();
    *charger_connected = !!ret;
    }
    ```

- **Function Signatures**:
  - Prefer using `void` in function signatures without parameters:
  - Without the void you can pass any or no arguments to that function without ever getting any warnings

    ```c
    int charger_srv_init(void);
    ```

- **Enum Naming Conventions**:
  - Use uppercase letters for enum values, following common coding practices.
  - Also its common to use mux_instead of multiplexer_

    ```c
    enum multiplexer_outputs {
        RFIX_OUTPUT,
        ...
    };
    ```

- **Variable Declaration**:
  - Declare variables like `int ret;` at the start of the block for better readability.
  - Declaring a variable in a switch case without scope is generally discouraged. Move this to the top of the block (function).
    **Good Example**:

    ```c
    fn(){
    const struct flash_area *settings_area;

    switch (target) {
    case CTRL_SRV_TEARDOWN_TARGET_WIPE:
    .......
    }
    }

    ```
  
    **Bad Example**:

    ```c
    fn(){

    switch (target) {
    case CTRL_SRV_TEARDOWN_TARGET_WIPE:
    const struct flash_area *settings_area;
    .......
    }
    }
    ```

    **Good Example**:

    ```c
    static void ctrl_srv_teardown(void)
    {
        ctrl_srv_deinit();

        if (data.state != CTRL_SRV_STATE_WIPE) {
            int ret = settings_save_subtree(APP_SETTINGS_SUBTREE);
            if (ret != 0) {
                LOG_ERR("Failed to save app settings! (%d)", ret);
            }
        }
    }
    ´´´

    **Bad Example**:
    ```c
    static void ctrl_srv_teardown(void)
    {
        int ret;
        ctrl_srv_deinit();

        if (data.state != CTRL_SRV_STATE_WIPE) {
            ret = settings_save_subtree(APP_SETTINGS_SUBTREE);
            if (ret != 0) {
                LOG_ERR("Failed to save app settings! (%d)", ret);
            }
        }
    }
    ´´´

- **Avoid redundant checks in if conditions**

    **Good Example**:

    ```c
    enum mounting_location sample_srv_get_mounting_location(void)
    {
        enum mounting_location loc;

        if (is_right_hoof) {
            if (is_hind_hoof) {
                loc = MNT_LOC_RIGHT_HIND;
            } else {
                loc = MNT_LOC_RIGHT_FRONT;
            }
        } else {
            if (is_hind_hoof) {
                loc = MNT_LOC_LEFT_HIND;
            } else {
                loc = MNT_LOC_LEFT_FRONT;
            }
        }

        return loc;
    }
    ```

    **Bad Example**:

    ```c
    enum mounting_location sample_srv_get_mounting_location(void)
    {
        enum mounting_location loc;

        if (is_right_hoof && is_hind_hoof) {
            loc = MNT_LOC_RIGHT_HIND;

    }

        if (is_right_hoof && !is_hind_hoof) {
            loc = MNT_LOC_RIGHT_FRONT;
        }

        if (!is_right_hoof && is_hind_hoof) {
            loc = MNT_LOC_LEFT_HIND;
        }

            loc = MNT_LOC_LEFT_FRONT;

                return loc;
    }
    ```

- **Do not read outside of an array**

    **Good Example**:

    ```c
    int imu_srv_set_imu_frequency(uint16_t frequency)
    {
      const uint16_t imu_frequencies[] = {12, 26, 52, 104, 208, 416};

      for (size_t i = 0; i < ARRAY_SIZE(imu_frequencies); i++) {
        if (frequency == imu_frequencies[i]) {
          imu_frequency = frequency;
          return 0;
        }
      }

      LOG_DBG("Invalid IMU frequency");
      return -EINVAL;
    }
    ```

    **Bad Example**:

    ```c
    int imu_srv_set_imu_frequency(uint16_t frequency)
    {
      const uint16_t imu_frequencies[] = {12, 26, 52, 104, 208, 416};

      for (size_t i = 0; i <= ARRAY_SIZE(imu_frequencies); i++) {
        if (frequency == imu_frequencies[i]) {
          imu_frequency = frequency;
          return 0;
        }
      }

      LOG_DBG("Invalid IMU frequency");
      return -EINVAL;
    }
    ```

- **ASCII Only in Source Files**:
  - Use only ASCII characters in source files. Non-ASCII characters should be avoided.
  . To check, use:

    ```bash
    LC_ALL=C grep -r --color='auto' -P -n "[\x80-\xFF]" -I app/
    ```

  - *Unacceptable*

    ```c
    τ = 𝙍𝙚𝙦 × 𝘾𝙚𝙦 with 𝙍𝙚𝙦 = (33kOhm||10kOhm) and 𝘾𝙚𝙦 = 33nF
    ```

- **integer promotion**
  - In the following code, you can directly return return (value - actual_min) * ( ..., no need to create extra variable and cast it. (read about integer promotion)

    ```c
    uint32_t result =
      (value - actual_min) * (target_max - target_min) / (actual_max - actual_min) +
      target_min;

    return (uint16_t)result;
    ```

- **Check and avoid overflow and undefined behavior in calculations**

    **Bad Example**:

    ```c

    static uint16_t sample_comp_apply_offset(uint16_t value, uint16_t actual, uint16_t target)
    {
      return value - (actual - target);
    }

    static uint16_t sample_comp_interpolate(uint16_t value, uint16_t actual_min, uint16_t actual_max,
      uint16_t target_min, uint16_t target_max)
    {
      return (value - actual_min) * (target_max - target_min) / (actual_max - actual_min) +
          target_min;
    }
    ```

  - we should add a check in sample_comp_check_cal() that actual - target does not wrap, probably it's fine to assert that actual and target are both in our value range of [0:4095]. (12 Bit adc resolution)
    Then actual - target will never wrap. What I mean with wrap here is something like actual = INT16_MIN and target = 10.
  - Similarly, we should add checks in sample_comp_check_cal() that target_max - target_min cannot wrap and that actual_max - actual_min cannot wrap.

  **Good Example**:

  ```c

  static uint16_t sample_comp_apply_offset(uint16_t value, uint16_t actual, uint16_t target)
  {
    /* Left and right side are in the range [0:4095], so result fits in int. */
    int off = actual - target;

    /* Left and right side are in the range [-4095:4095], so result fits in int. */
    return MIN(MAX(value - off, 0), 4095);
  }

  static uint16_t sample_comp_interpolate(uint16_t value, uint16_t actual_min, uint16_t actual_max,
            uint16_t target_min, uint16_t target_max)
  {

    if (actual_min > value) {
      __ASSERT_NO_MSG(false);
      return target_min;
    }

    /*
    * Interpolate using:
    *
    * result = (value - actual_min) * (target_max - target_min) / (actual_max - actual_min) +
    * target_min
    *          --------------------- a ------------------------
    *          ----------------------------------------- b --------------------------------
    */

    /* Left and right side positive and max 4095. The result fits in int24. */
    int a = (value - actual_min) * (target_max - target_min);

    /* Left and right side positive. Result fits in int24. */
    int b = a / (actual_max - actual_min);

    /* Left side is positive and fits in int25, right side is in the range [0:4095]. Result fits
    * in int25. */
    return MIN(b + target_min, 4095);
  }

  ```

   And check the data before using them:

  ```c
  static int sample_comp_check_cal(const struct adc_cal *cal)
  {
    int c;
    int i;

    /* ensure actual and target values are within the valid range [0, 4095] */
    for (c = 0; c < ADC_CAL_MAIN_CHANNELS; c++) {
      for (i = 0; i < ADC_CAL_MAIN_REF_COUNT; i++) {
        if (cal->main.actual[c][i] > 4095) {
          return -EINVAL;
        }
      }
    }

    .........

    /* ensure monotonic increasing actual values for main cal */
    for (c = 0; c < ADC_CAL_MAIN_CHANNELS; c++) {
      for (i = 1; i < ADC_CAL_MAIN_REF_COUNT; i++) {
        if (cal->main.actual[c][i] < cal->main.actual[c][i - 1]) {
          return -EINVAL;
        }
      }
    }

    .........

    /* ensure monotonic increasing target values for main and aux cal */
    for (i = 1; i < ADC_CAL_MAIN_REF_COUNT; i++) {
      if (cal->main.target[i] > cal->main.target[i - 1] ||
          cal->aux.target[i] > cal->aux.target[i - 1]) {
        return -EINVAL;
      }
    }

    return 0;
  }

  ```

- **Avoid Incorrect Pointer Reassignments**:
  When passing an array or buffer, ensure that the values are copied instead of reassigning the pointer, which can lead to undefined behavior.

  - **Bad Example**:

    ```c
    void strain_counter_get_counter(struct strain_counter *counter)
    {
        counter = cells_strain_counter; // Just reassigns the pointer, does not copy data
    }
    ```

  - **Good Example**:

    ```c
    #include <string.h> // For memcpy

    void strain_counter_get_counter(struct strain_counter *counter, size_t counter_size)
    {
        if (counter_size < sizeof(cells_strain_counter)) {
            // Handle error: Target buffer is too small
            return;
        }

        memcpy(counter, cells_strain_counter, sizeof(cells_strain_counter)); // Safely copies data
    }
    ```

  - **Explanation**: Use `memcpy` or loops to copy data to the target buffer explicitly. Validate the buffer size before copying to prevent overflows.

- **Always Validate Buffer Sizes**:
  When working with buffers, especially in embedded systems, validate their size to ensure you don’t read or write out of bounds.

  - **Bad Example**:

    ```c
    ssize_t ble_srv_consteed_cell_strain_counter_read(struct bt_conn *conn,
                                                      const struct bt_gatt_attr *attr, void *buf,
                                                      uint16_t len, uint16_t offset)
    {
        struct strain_counter counter[PRESSURE_CHANNELS_COUNT];
        strain_counter_get_counter(counter); // Unsafe if buffer size mismatches
        return sizeof(counter); // Risk of returning incorrect size
    }
    ```

  - **Good Example**:

    ```c
    ssize_t ble_srv_consteed_cell_strain_counter_read(struct bt_conn *conn,
                                                      const struct bt_gatt_attr *attr, void *buf,
                                                      uint16_t len, uint16_t offset)
    {
        struct strain_counter counter[PRESSURE_CHANNELS_COUNT];

        if (len < sizeof(counter)) {
            return BT_GATT_ERR(BT_ATT_ERR_INVALID_ATTRIBUTE_LEN); // Handle insufficient buffer size
        }

        strain_counter_get_counter(counter, sizeof(counter)); // Safely fetch values
        memcpy(buf, counter, sizeof(counter)); // Copy to the buffer passed by the caller

        return sizeof(counter); // Correct size return
    }
    ```
