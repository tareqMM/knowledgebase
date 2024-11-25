## **Git Best Practices**

- **Commit Messages**:
  - Follow this format for writing commit messages:
    - Subject/summary (one line)
    - Blank line
    - Body (paragraphs with empty lines between)
    - Blank line
    - Trailers (e.g., `Fixes: CNT-127`, `Signed-off-by: Your Name <email>`)

- **Wrap Lines**:
  - Wrap lines in self-written paragraphs to 72 characters to adhere to standard commit message style. (Use `gwap` command in vim.)
  - If you have something not "self-written," like a URL or a compiler message, don't wrap it.

- **Rename files in the correct way**
  - ` git mv old_filename new_filename ` then commit it.

---
