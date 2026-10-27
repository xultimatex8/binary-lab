# Functional Requirements — BinaryLab

Requirement ids: **FR-E** = analysis engine, **FR-G** = graphical interface. The
[PE analysis flow](pe-format-analysis.md) shows how FR-E01 to FR-E32 work end to end.

## Analysis engine

### File input and identification
- **FR-E01**: Open a file from a path given by the user and read it as data only. The file is never executed, loaded as a library or passed to another program.
- **FR-E02**: Report basic file information: name, full path, size in bytes.
- **FR-E03**: Calculate the SHA-256 hash of the file contents and expose it as the file's reproducible identifier.
- **FR-E04**: Identify the file format by inspecting its contents (the `MZ` signature and the PE signature), not by trusting the extension.
- **FR-E05**: Report clearly when a file is not a supported format, instead of failing or guessing. Files with an `.exe` or `.dll` extension that are not valid PE files are reported as such.
- **FR-E06**: Support 32-bit (PE32) and 64-bit (PE32+) executables.

### PE parsing
- **FR-E07**: Parse the DOS header and locate the PE headers through its offset field.
- **FR-E08**: Parse the PE signature and the COFF file header: target machine, number of sections, timestamp, characteristics.
- **FR-E09**: Parse the optional header: entry point, image base, section and file alignment, image size, subsystem, DLL characteristics.
- **FR-E10**: Distinguish executables from DLLs using the file characteristics.
- **FR-E11**: Parse the data directory table and identify which directories are present.
- **FR-E12**: Parse the section table: name, virtual size and address, raw size and offset, characteristics.
- **FR-E13**: Convert relative virtual addresses (RVAs) to file offsets using the section table, and report RVAs that fall outside every section.

### Section analysis
- **FR-E14**: Decode section characteristics into readable permissions (readable, writable, executable) and content types (code, initialized data, uninitialized data).
- **FR-E15**: Calculate the Shannon entropy of each section's raw data.
- **FR-E16**: Compare virtual size against raw size per section and expose the difference.
- **FR-E17**: Detect data stored after the last section (overlay) and report its size and offset.

### Import analysis
- **FR-E18**: Parse the import directory and list every imported library with its imported functions (by name or by ordinal).
- **FR-E19**: Group notable imported functions by the operating system capability they relate to (for example process and memory management, networking, registry, cryptography, file system).
- **FR-E20**: Report an absent or empty import table as a neutral observation, not as an error.

### String extraction
- **FR-E21**: Extract printable ASCII strings above a configurable minimum length.
- **FR-E22**: Extract UTF-16 little-endian strings above a configurable minimum length.
- **FR-E23**: Record the file offset and, when it falls inside a section, the section where each string was found.
- **FR-E24**: Bound the amount of data extracted so very large files cannot exhaust memory.

### Security indicators
- **FR-E25**: Evaluate a defined set of rule-based checks over the parsed data. The initial catalog is in the [rule table](pe-format-analysis.md#rule-catalog).
- **FR-E26**: Every finding carries an id, a severity (informational, low or medium), a short title and the concrete evidence that triggered it (field names and values, section names, offsets).
- **FR-E27**: Findings never state or imply that a file is malicious or safe. They describe a characteristic and what it may mean.
- **FR-E28**: Each rule can be unit-tested in isolation from the parser and the interface.

### Explanations
- **FR-E29**: Provide, for every finding and for every major structure (headers, sections, imports, strings), a plain-language explanation of what it is and why it exists.
- **FR-E30**: For every finding, explain why it may matter, what legitimate reasons may cause it, and suggest follow-up checks.

### Robustness and safety
- **FR-E31**: Validate every offset, size and count against the file bounds before reading. Truncated, inconsistent or deliberately malformed files produce a handled error or a partial result with warnings, never a crash or an out-of-bounds read.
- **FR-E32**: When parsing fails midway, keep and return everything parsed up to that point together with the reason the rest could not be read.

### Reports
- **FR-E33**: Build a structured report containing file information, parsed structures, findings with their evidence, and the explanations shown.
- **FR-E34**: Export the report to JSON and to Markdown.
- **FR-E35**: Include the SHA-256 hash, the BinaryLab version and the analysis date in every exported report so results can be reproduced and compared.

## Graphical interface

- **FR-G01**: Main window with a menu, a status bar and an area for the analysis results.
- **FR-G02**: Open a file through a file dialog and by dragging it onto the window.
- **FR-G03**: Summary view — file name, size, SHA-256 (with a copy action), format, architecture, executable or DLL, and a count of findings by severity.
- **FR-G04**: Headers view — the parsed DOS, COFF and optional header fields with their values, each with an inline explanation.
- **FR-G05**: Sections view — table with name, addresses, sizes, permissions and entropy, with flagged sections visibly marked.
- **FR-G06**: Imports view — expandable tree of libraries and functions, filterable by name, with the capability group shown where known.
- **FR-G07**: Strings view — table with value, offset and encoding, with text filtering and a minimum-length control.
- **FR-G08**: Findings view — list ordered by severity; selecting a finding shows its evidence, explanation, legitimate causes and suggested follow-up checks.
- **FR-G09**: Selecting a finding highlights the related item (section, import or header field) in its own view.
- **FR-G10**: Export the report as JSON or Markdown through a save dialog.
- **FR-G11**: Analysis runs off the interface thread, with a progress indication, so the window stays responsive on large files.
- **FR-G12**: Loading and error states — clear messages for unsupported formats, unreadable files, truncated files and partial results (FR-E05, FR-E31, FR-E32).
- **FR-G13**: A persistent notice that findings are indicators and not verdicts (FR-E27).
- **FR-G14**: About screen — version, license and scope and limitations.

## Known constraints

- **Static analysis only.** BinaryLab never runs the file, so it cannot observe runtime behavior, network traffic, unpacked code or anything produced dynamically. Packed or encrypted files can hide most of what static analysis looks for.
- **Indicators are not verdicts.** Every rule describes an unusual characteristic that also occurs in legitimate software (installers, protectors, game anti-cheat, some compilers). A file with no findings is not necessarily safe.
- **PE only, Windows only.** ELF, Mach-O, .NET metadata, scripts and documents are out of scope for the MVP. A managed .NET assembly is still a valid PE file and is parsed as one, but its managed code is not analyzed.
- **No signature or reputation database.** The MVP does not compare against known-malware hashes, YARA rules or online services and never uploads the analyzed file anywhere.
- **Analyzed files are untrusted input.** The parser must remain robust to malformed data; correctness of the parser is prioritized over adding new analyses (see the incremental approach in the [README](../README.md#technical-highlights)).