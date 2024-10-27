## **Best Practices For Better Code Quality**

### **Common Mistakes in Pull Requests **

- **Empty Lines and Spaces**:
  - Ensure there is only one empty line at the end of each file.
    I can recommend the following two vscode settings (add to user settings (JSON)):
    ```json
    "files.insertFinalNewline": true,
    "files.trimFinalNewlines": true,
    ```
  - It will always add a trailing newline but remove them if there are more than one.
  - Avoid trailing spaces in the code.

- **Grammar and Spelling**:
  - Check grammar, spelling, and capitalization when writing code. Consider using a plugin for automatic corrections.

- **Writing Style**:
  - Write consistent sentences and maintain consistent naming conventions throughout the project.

- **Comments**:
  - Comments should add value to the code and should not repeat what the code already conveys.
  - Maintain comments like any other part of the code and avoid obvious or redundant comments.
  - Merge similar comments for clarity. For example the following comments can be merged together:

    ```c
    /* Sample all channels in the sequence */
    err = adc_read_dt(&adc_channels[0], &sequence); // The ADC device is the same for all channels.
    ```

- **Formatting**:
  - Always apply `clang-format` to maintain consistent code formatting.
---
