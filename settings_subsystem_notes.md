# Settings Subsystem Overview

- **Easy way to store key/value pairs** (usable at runtime)
- Utilizes the following storage backends:
  - **LittleFS**
  - **NVS**
  - **FAT**

## Key Functions

### Initialization
- `int settings_subsys_init(void)`
  - Initializes settings and backend.
  - Should be called at application startup. If using a filesystem (FS) as the backend, call this after mounting the FS. For FCB backend, no such restriction applies.

### Registration and Loading
- `int settings_register(struct settings_handler *cf)`
  - Registers a handler for settings items stored in RAM with default commit priority.

- `int settings_load(void)`
  - Loads serialized items from registered persistence sources.
  - Calls handlers for values found in serialized item subtrees registered earlier.

- `int settings_load_subtree(const char *subtree)`
  - Loads a limited set of serialized items from registered persistence sources.
  - Calls handlers for values in the specified subtree.

- `int settings_load_subtree_direct(const char *subtree, settings_load_direct_cb cb, void *param)`
  - Loads a limited set of serialized items using a callback, bypassing the normal data workflow in the settings module.
  - Does not call commit function; the user is responsible for calling commit after completion.

### Saving and Deleting
- `int settings_save(void)`
  - Saves currently running serialized items that differ from persisted values.

- `int settings_save_subtree(const char *subtree)`
  - Saves a limited set of currently running serialized items within a specific subtree that differ from persisted values.

- `int settings_save_one(const char *name, const void *value, size_t val_len)`
  - Writes a single serialized value to persisted storage if it has changed.

- `int settings_delete(const char *name)`
  - Deletes a single serialized item from persisted storage by setting its value to `NULL`.

### Committing
- `int settings_commit(void)`
  - Commits all settings handlers, applying settings that were set but not yet applied.

- `int settings_commit_subtree(const char *subtree)`
  - Commits settings handlers within a specific subtree, applying settings that were set but not yet applied.

## Runtime Operations

- `int settings_runtime_set(const char *name, const void *data, size_t len)`
  - Sets a value with a specific key to a module handler.

- `int settings_runtime_get(const char *name, void *data, size_t len)`
  - Retrieves a value corresponding to a key from a module handler.

- `int settings_runtime_commit(const char *name)`
  - Applies settings in a module handler.

## Key Structure Convention

Keys follow a tree structure format:
```
<package>/[<group>/]<setting>
```

Examples:
```
sensor
├── x_axis
│   ├── offset
│   └── threshold
└── y_axis
    ├── offset
    └── threshold

backend
├── server
└── port
```

Key Examples:
- `sensor/x_axis/offset`
- `sensor/x_axis/threshold`
- `sensor/y_axis/offset`
- `sensor/y_axis/threshold`
- `backend/server`
- `backend/port`

## Handlers

Settings handlers for a subtree include a set of handler functions, registered with:
- `settings_register()` for dynamic handlers
- `SETTINGS_STATIC_HANDLER_DEFINE()` for static handlers.

Handler functions:
- **h_get**: Called when retrieving a settings element by name using `settings_runtime_get()`.
- **h_set**: Called when loading from persisted storage with `settings_load()` or using `settings_runtime_set()`.
- **h_commit**: Called after all settings are loaded; delays individual setting effect, useful for interdependent settings.
- **h_export**: Called to write all current settings, triggered by `settings_save()`.

Handlers can prioritize commits using `cprio`, useful for initializing services that other `h_commit` calls depend on.

## Backends

Backends manage loading and saving data for settings handlers, registered with:
- `settings_src_register()` for load-capable backends
- `settings_dst_register()` for save-capable backends.

Backend handler functions:
- **csi_load**: Loads values from persistent storage using `settings_load()`.
- **csi_save**: Saves a single setting to persistent storage using `settings_save_one()`.
- **csi_save_start**: Starts saving all current settings using `settings_save()` or `settings_save_subtree()`.
- **csi_save_end**: Completes saving all current settings.

## Loading Data from Persisted Storage

- `settings_load()` uses `h_set` to load settings data from storage to volatile memory.
- After loading, `h_commit` signals that settings were successfully retrieved.

## Storing Data to Persistent Storage

- `settings_save_one()` uses a backend to store data.
- `settings_save()` uses `h_export` to store multiple data items in one operation.
- Only keys covered by `h_export` are stored by `settings_save()`.


# Different Approaches for Using the Zephyr Subsystem

### 1. Approach

If you have multiple modules in which each has global variable(s) that is/are set and read (for example, using setter and getter functions that will be set by the user and stored in the settings storage), you can use the following approach:

**Main APP Subtree**: `APP_SETTINGS_SUBTREE "app"`  
The other settings will be saved under it, like:
- `app/sample_srv/sample_interval`
- `app/report_srv/report_interval`

In each module, you need to:

- Include the header:
  ```c
  #include <zephyr/settings/settings.h>
  ```

- Define the setting's key:
  ```c
  #define SAMPLE_INTERVAL_SETTING_KEY "sample_interval"
  ```

- Create the `.h_export` and `.h_set` handlers:

  ```c
  // Will be called after settings_load_subtree(APP_SETTINGS_SUBTREE) is called
  static int sample_interval_setting_set(const char *name, size_t len, settings_read_cb read_cb,
                                         void *cb_arg)
  {
      const char *next;
      uint8_t sample_interval_setting;

      if (settings_name_steq(name, SAMPLE_INTERVAL_SETTING_KEY, &next) && !next) {
          if (len != sizeof(sample_interval_setting)) {
              return -EINVAL;
          }

          read_cb(cb_arg, &sample_interval_setting, sizeof(sample_interval_setting));
          if ((sample_interval_setting >= 1000 && sample_interval_setting <= 50000)) {
              sample_interval = sample_interval_setting; // Save the loaded setting into the global variable
          }

          return 0;
      }

      return -ENOENT;
  }

  // Will be called after settings_save_subtree(APP_SETTINGS_SUBTREE) is called
  static int sample_interval_setting_export(int (*cb)(const char *name, const void *value,
                                                      size_t val_len))
  {
      uint8_t sample_interval_setting = sample_interval; // Save the global variable value into the setting variable

      return cb(APP_SETTINGS_SUBTREE "/" SAMPLE_INTERVAL_SETTING_KEY, &sample_interval_setting,
                sizeof(sample_interval_setting));
  }
  ```

- Register the handlers dynamically or statically:

  ```c
  SETTINGS_STATIC_HANDLER_DEFINE(sample_interval_handler, APP_SETTINGS_SUBTREE "/" SAMPLE_INTERVAL_SETTING_KEY, NULL,
                                 sample_interval_setting_set, NULL, sample_interval_setting_export);
  ```

  or

  ```c
  // Must be defined in the file scope of the module 
  struct settings_handler sample_interval_handler = {
      .name = APP_SETTINGS_SUBTREE "/" SAMPLE_INTERVAL_SETTING_KEY,
      .h_set = sample_interval_setting_set,
      .h_export = sample_interval_setting_export
  };

  // Then call settings_register(&sample_interval_handler); in the init function of the module
  ```

- Call `settings_subsys_init();` in the main function.
- Use `settings_load_subtree(APP_SETTINGS_SUBTREE)` (for example, in the startup function) to load the settings from storage.
- Use `settings_save_subtree(APP_SETTINGS_SUBTREE);` before shutting down (for example, at the end of a teardown function).
- Note: If there were no settings saved before, or if the settings were removed due to a factory reset (the subtree no longer exists), `settings_save_subtree()` will not call the `.h_set` handler and will not return an error, so it may be challenging to log a message indicating no settings were written.

  - If the modules do not have global variables that are set and accessed via setter and getter functions, you can use the runtime set and get functions:
    - `int settings_runtime_set(const char *name, const void *data, size_t len)` – Sets a value with a specific key to a module handler.
    - `int settings_runtime_get(const char *name, void *data, size_t len)` – Retrieves a value corresponding to a key from a module handler.
      - Note: Registering the handler will fail if an existing handler with the same name is present!

- We use `settings_save_subtree(APP_SETTINGS_SUBTREE)` instead of `settings_save()` and `settings_load_subtree(APP_SETTINGS_SUBTREE)` instead of `settings_load()` because all our modules store their settings under `APP_SETTINGS_SUBTREE`. Therefore, we are only interested in loading and saving the settings related to this subtree. If we have multiple subtrees of interest, we can call `settings_save()` and `settings_load()` to save and load settings across all subtrees.

- Here, we save and read the settings only once per boot cycle, as our use case ensures the device will shut down properly (i.e., the battery will not be removed abruptly, and the device will not reboot unexpectedly). This guarantees settings are saved before shutdown or reboot. Loading and saving settings only once per boot cycle also reduces flash write and read access, especially when the user can change values frequently during operation.

- If an uncontrolled reboot or shutdown is possible, settings may not save to storage if the device reboots or shuts down before calling `settings_save_subtree(APP_SETTINGS_SUBTREE);` in the teardown function. Therefore, it is advisable to save each setting separately as it changes, as shown below:

  ```c
  int sample_srv_set_sample_interval(uint8_t sample_interval_setting)
  {
      int ret;

      ret = settings_save_one(APP_SETTINGS_SUBTREE "/" SAMPLE_INTERVAL_SETTING_KEY, &sample_interval_setting,
                              sizeof(sample_interval_setting));
      if (ret != 0) {
          LOG_ERR("Failed to write sample_interval (%d)", ret);
          return ret;
      }

      sample_interval = sample_interval_setting;

      return 0;
  }
  ```

  - This will immediately save the new sample interval in the settings storage and update the global variable for use.
  - Note: Using `settings_save_one()` does not require a `.h_export` handler.

- If each module has only one setting, you can use `settings_load_subtree_direct()` without registering handlers via static or dynamic registration. Define the direct loader handler and use it as a parameter in `settings_load_subtree_direct()`:

  Example:
  ```c
  // In the init function of the module

  settings_load_subtree_direct(APP_SETTINGS_SUBTREE "/" SAMPLE_INTERVAL_SETTING_KEY,
                               sample_interval_setting, &ret);
  if (ret != 0) {
      LOG_WRN("sample_interval unknown!");
  }
  ```

- Note: If no settings are saved, or they are removed (e.g., due to a factory reset), the function parameter `*param` (here, `ret`) will be negative, and the `.h_set` handler will not be called. Check this parameter to inform the user if no settings were written.

- If you have multiple settings in a module, handle them in the `.h_export` and `.h_set` handlers, as shown below:

  ```c
  int alpha_handle_set(const char *name, size_t len, settings_read_cb read_cb,
                       void *cb_arg) {
    const char *next;
    size_t next_len;
    int rc;

    if (settings_name_steq(name, "angle/1", &next) && !next) {
      if (len != sizeof(angle_val)) {
        return -EINVAL;
      }
      rc = read_cb(cb_arg, &angle_val, sizeof(angle_val));
      printk("<alpha/angle/1> = %d\n", angle_val);
      return 0;
    }

    next_len = settings_name_next(name, &next);

    if (!next) {
      return -ENOENT;
    }

    if (!strncmp(name, "length", next_len)) {
      next_len = settings_name_next(name, &next);

      if (!next) {
        rc = read_cb(cb_arg, &length_val, sizeof(length_val));
        printk("<alpha/length> = %" PRId64 "\n", length_val);
        return 0;
      }

      if (!strncmp(next, "1", next_len)) {
        rc = read_cb(cb_arg, &length_1_val, sizeof(length_1_val));
        printk("<alpha/length/1> = %d\n", length_1_val);
        return 0;
      }

      if (!strncmp(next, "2", next_len)) {
        rc = read_cb(cb_arg, &length_2_val, sizeof(length_2_val));
        printk("<alpha/length/2> = %d\n", length_2_val);
        return 0;
      }

      return -ENOENT;
    }

    return -ENOENT;
  }
  ```

  ```c
  int alpha_handle_export(int (*cb)(const char *name, const void *value,
                                    size_t val_len)) {
    printk("export keys under <alpha> handler\n");
    (void)cb("alpha/angle/1", &angle_val, sizeof(angle_val));
    (void)cb("alpha/length", &length_val, sizeof(length_val));
    (void)cb("alpha/length/1", &length_1_val, sizeof(length_1_val));
    (void)cb("alpha/length/2", &length_2_val, sizeof(length_2_val));

    return 0;
  }
  ```

### 2. Approach

If you have many modules, each with multiple settings to store, and want to avoid registering and defining the settings handlers for each module, encapsulate this complexity in a separate module. This module will register the handlers and provide setters and getters for the settings for other modules. See how this is done in:
- `ws/nrf/lib/bin/lwm2m_carrier/os/lwm2m_settings.c`
- `ws/nrf/samples/cellular/modem_shell/src/ppp/ppp_settings.c`
```
# Links

- [Settings Subsystem Documentation](https://docs.zephyrproject.org/latest/services/settings/index.html)  
  A short API description with small examples.

- [API Documentation](https://docs.zephyrproject.org/apidoc/latest/group__settings.html)  
  Detailed API documentation for the settings subsystem.

- [Comprehensive Sample with Many Different Use Cases](https://docs.zephyrproject.org/latest/samples/subsys/settings/README.html)  
  This example is extensive (580 lines) and complex, including unrelated code such as filesystem mounting.
