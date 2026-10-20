---
date: 2026-10-07
description: "A controlled-English guide to install MarkItDown from the Microsoft source repository and convert local files to Markdown."
tags:
  - reference
  - markitdown
  - documentation
status: active
project: MarkItDown
source_path: /home/tim/work/markitdown
---

# MarkItDown User Guide

## Purpose

Use MarkItDown to convert a supported file to Markdown.

Use the Markdown output for text analysis or LLM input.

Do not use MarkItDown when you need a high-fidelity copy of the original document.

## Installation

The source repository is `/home/tim/work/markitdown`.

The Python environment is `/home/tim/work/markitdown-venv`.

The installation uses the editable package in the source repository. It also installs the optional dependencies for all supported formats.

Open a terminal. Run these commands:

```bash
cd /home/tim/work/markitdown
../markitdown-venv/bin/python -m pip install -e 'packages/markitdown[all]'
```

Do not run `pip install -e .` from the repository root. The root directory does not contain the Python package configuration.

The `markitdown` command is in the virtual environment. Use its full path:

```bash
/home/tim/work/markitdown-venv/bin/markitdown --help
```

## Convert a file

Run this command to convert a PDF file:

```bash
/home/tim/work/markitdown-venv/bin/markitdown input.pdf -o output.md
```

Replace `input.pdf` with the input file path.

Replace `output.md` with the output file path.

MarkItDown writes the result to the output file.

Run this command to write the result to standard output:

```bash
/home/tim/work/markitdown-venv/bin/markitdown input.pdf
```

Run this command to redirect the result to a file:

```bash
/home/tim/work/markitdown-venv/bin/markitdown input.pdf > output.md
```

Use `-o` or shell redirection. Do not use both methods in the same command.

## Convert data from standard input

Use standard input when another command provides the file data.

```bash
cat input.pdf | /home/tim/work/markitdown-venv/bin/markitdown
```

For input from standard input, provide a file extension when MarkItDown cannot identify the file type:

```bash
cat input.pdf | /home/tim/work/markitdown-venv/bin/markitdown -x pdf
```

## Check the installation

Run these commands:

```bash
/home/tim/work/markitdown-venv/bin/markitdown --version
/home/tim/work/markitdown-venv/bin/markitdown --help
```

Check that the version command returns a version number.

Check that the help command lists the input and output options.

## Supported input formats

MarkItDown supports formats that include:

- PDF
- Word documents
- PowerPoint files
- Excel files
- HTML
- CSV, JSON, and XML
- Images
- Audio
- ZIP archives
- EPUB files

Some formats need optional dependencies. The installation command in this guide installs the `all` extra.

## Review the output

Open the Markdown file after conversion.

Check headings, lists, tables, links, and special characters.

Compare important values with the source file.

Correct any conversion errors before you use the Markdown output.

MarkItDown can change the layout of tables and diagrams.

Scanned documents can require OCR. The default local conversion can omit text that exists only in an image.

## Security

MarkItDown accesses files and resources with the permissions of the current process.

Only convert files from trusted sources.

Do not enable third-party plugins unless you trust the plugins.

Do not use Azure options unless the task requires them and the correct endpoint is configured.

## Update the source code

Run these commands to get the latest repository changes and update the editable installation:

```bash
cd /home/tim/work/markitdown
git pull --ff-only
../markitdown-venv/bin/python -m pip install -e 'packages/markitdown[all]'
```

Check the repository changes before you update the source code.

Do not use `git pull` if the repository has local changes that you need to keep.

## Remove the installation

Run this command to remove the Python package:

```bash
/home/tim/work/markitdown-venv/bin/python -m pip uninstall markitdown
```

This command does not remove the source repository or the virtual environment.

Remove `/home/tim/work/markitdown` or `/home/tim/work/markitdown-venv` only when you no longer need those directories.

## Installation result

The source repository is checked out at `/home/tim/work/markitdown`.

The `markitdown[all]` package is installed in `/home/tim/work/markitdown-venv`.

A local PDF conversion completed successfully during installation verification.

## Related

- [[MarkItDown Reference]]
- [[Index]]
- [[Home]]
