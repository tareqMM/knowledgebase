# Generating Doxygen Documentation for a Zephyr Project

## Step 1: Install Doxygen and Graphviz
Graphviz is used to create call graphs and diagrams.

```bash
sudo apt-get install doxygen graphviz
```

## Step 2: Create a `Doxyfile`
If your project does not already include a `Doxyfile`, create one with the following command:

```bash
doxygen -g
```

This command generates a `Doxyfile` with default settings.

## Step 3: Configure the `Doxyfile`
Edit the `Doxyfile` to customize your documentation settings:

- **PROJECT_NAME**: Set this to your project’s name, e.g., `PROJECT_NAME = "Name of your project"`
- **INPUT**: Specify the directories containing your source code. For example:
  ```plaintext
  INPUT = src include
  ```
- **RECURSIVE**: Enable recursive scanning for source files.
  ```plaintext
  RECURSIVE = YES
  ```
- **EXTRACT_ALL**: Set this to `YES` to extract all entities.
  ```plaintext
  EXTRACT_ALL = YES
  ```
- **OUTPUT_DIRECTORY**: Define the directory where the documentation will be generated. For example:
  ```plaintext
  OUTPUT_DIRECTORY = docs
  ```
- **EXCLUDE_PATTERNS**: Optionally exclude certain directories, such as `tests` or `build`, if needed.

- **GENERATE_LATEX**: If PDF output is unnecessary, set this to `NO` to disable LaTeX generation.

## Step 4: Generate Documentation
Generate the documentation by running:

```bash
doxygen Doxyfile
```

Doxygen will output the documentation in the specified directory. To view the HTML documentation, open `index.html` in the output directory (e.g., `docs/html/index.html`).
