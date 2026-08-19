Qzip
=========================
An efficient tool for raw genomic FASTQ sequencing data compression and decompression.


__PROGRAM: Qzip__<br>
__VERSION: 1.0.0-beta.7__<br>
__PLATFORM: Linux / macOS__<br>
__ARCHITECTURE: x86_64__<br>
__COMPILER: gcc (C99)__<br>
__AUTHOR: xiaolong zhang__<br>
__EMAIL: xiaolongzhang2015@163.com__<br>
__DATE:   2024-09-09__<br>
__UPDATE: 2026-07-22__<br>
__DEPENDENCE__<br>
* __GNU make and gcc__<br>
* __zlib__<br>
* __pthread__<br>



# 1. Description

* Qzip is a high-performance compression tool specifically designed for raw genomic FASTQ sequencing data.<br>
* It supports both **reference-free** and **reference-based** compression algorithms, allowing users to balance between compression ratio and computational resources.
* For reference-free compression, two sequence encoding methods are provided: **BWT-MTF-RangeCode** (higher compression rate, more memory) and **BIT2** (faster, less memory).
* For reference-based compression, Qzip leverages a pre-built reference genome index to achieve superior compression ratios by aligning reads to the reference.
* Qzip also provides utilities for **validating** the integrity of compressed files, **viewing** file header information, and **building** reference genome indices.



# 2. Building


## 2.1 Dependencies

Before building Qzip, ensure the following libraries are installed on your system:

* **zlib** — required for gzip-compressed file I/O
* **pthread** — required for multi-threading support
* **libbz2** (optional) — enable BZ2 compressed file support by setting `BZ2_SUPPORT = 1` in the makefile

## 2.2 Compilation

```bash
# Build with debug symbols (default, DEBUG = 1)
make

# Build with optimization (release mode)
# Edit the makefile and set DEBUG = 0, then run:
make

# Clean build artifacts
make clean
```

The compiled binary `qzip` will be generated in the current directory.

## 2.3 Build Options

The makefile provides two configurable switches:

| Switch        | Default | Description                                                                        |
|---------------|---------|------------------------------------------------------------------------------------|
| `DEBUG`       | `0`     | `1`: compile with `-g -O0` for debugging;<br/> `0`: compile with `-O3` for release |
| `BZ2_SUPPORT` | `0`     | `1`: enable BZ2 format support (adds `-lbz2`);<br/> `0`: BZ2 support disabled      |



# 3. Usage

```
Usage: qzip <command> [options]
```

**Commands:**

| Command    | Description                                                        |
|------------|--------------------------------------------------------------------|
| `encode`   | Compress FASTQ file(s) into a compact QZ file                      |
| `decode`   | Decompress a QZ file back to FASTQ file(s)                         |
| `build`    | Build reference genome index files for reference-based compression |
| `info`     | Display header information of a QZ file                            |
| `validate` | Validate the integrity of a QZ file                                |
| `view`     | Quickly view reads from a QZ file (under development)              |



## 3.1 Command: `encode`

Compress one or more FASTQ files (plain or gzipped) into a single compact QZ archive.

```
Usage: qzip encode [options]
```

### 3.1.1 Options Summary

| Option           | Argument | Description                                            |
|------------------|----------|--------------------------------------------------------|
| `-h`, `--help`   | —        | Print help information for the encode command          |
| `-l`, `--list`   | FILE     | **\[Required]** List of FASTQ file paths, one per line |
| `-o`, `--output` | FILE     | **\[Required]** Output compressed QZ file path         |

##### Reference-free Options (default)

| Option         | Argument | Description                                               |
|----------------|----------|-----------------------------------------------------------|
| `-m`, `--mode` | STRING   | Compression mode for sequences: `BWT` (default) or `BIT2` |
| `-s`, `--size` | INT      | BWT block size in MB: `16`, `32`, or `64` (default: `64`) |

##### Reference-based Options

| Option        | Argument | Description                                |
|---------------|----------|--------------------------------------------|
| `-r`, `--ref` | FILE     | Prefix of the reference genome index files |

##### Other Options

| Option            | Argument | Description                                 |
|-------------------|----------|---------------------------------------------|
| `-t`, `--thread`  | INT      | Number of threads to use (default: `1`)     |
| `-v`, `--verbose` | —        | Show detailed encoding progress information |

### 3.1.2 Option Details

#### \[-h \| --help]

Print the help information for the encode command and exit.

---

#### \[-l \| --list]

A text file (without header) containing the full paths of the FASTQ files to be compressed, one file per line. 
Both plain FASTQ (`.fastq` / `.fq`) and gzip-compressed FASTQ (`.fastq.gz` / `.fq.gz`) are supported.

**Example:**
```
/home/user/data/sample1_R1.fastq.gz
/home/user/data/sample1_R2.fastq.gz
```

**Note:** When multiple FASTQ files are provided, all of them will be merged and compressed into a single QZ file. 
This is useful for combining paired-end reads or multiple sequencing runs of the same sample into one archive.

---

#### \[-o \| --output]

The path of the output compressed QZ file. The recommended file extension is `.qz`.

**Example:**
```
--output /home/user/data/sample.qz
```

---

#### \[-m \| --mode]

The compression algorithm used for encoding DNA sequences within the FASTQ data. Two options are available:

| Mode   | Description                                                                                                                                                                              |
|--------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `BWT`  | **(default)** Burrows-Wheeler Transform followed by Move-To-Front and Range Coding. Achieves a higher compression ratio at the cost of increased memory usage and slightly slower speed. |
| `BIT2` | A lightweight 2-bit encoding scheme. Faster and uses less memory, but produces larger compressed files compared to BWT.                                                                  |

**Default:** `BWT`

**Notes:**
- This option is only applicable to **reference-free** compression. When `--ref` is specified (reference-based mode), this option is ignored.
- For computer or server with limited RAM, consider using `BIT2` mode.
- For archival purposes where minimizing storage is the priority, use `BWT` mode.

---

#### \[-s \| --size]

The block size (in megabytes) used by the BWT compression algorithm. A larger block size yields a better compression ratio but consumes more memory during compression.

Valid values: `16`, `32`, `64`

| Block Size | Compression Ratio | Memory Usage |
|------------|-------------------|--------------|
| `16`       | Good              | Low          |
| `32`       | Better            | Medium       |
| `64`       | Best              | High         |

**Default:** `64` (best compression)

**Note:** This option is only effective when `--mode BWT` is used (the default). It has no effect with `--mode BIT2` or in reference-based mode.

---

#### \[-r \| --ref]

Enables **reference-based compression** by providing the prefix of the pre-built reference genome index files. The prefix is typically the same as the reference genome file name.

**Prerequisites:** Before using this option, you must first run `qzip build` to generate the three index files (`.pac`, `.bwt`, `.sa`) from the reference FASTA file.

**Example:**
```bash
# Step 1: Build the reference index
qzip build --ref GRCh38.fa
# This generates: GRCh38.fa.pac, GRCh38.fa.bwt, GRCh38.fa.sa

# Step 2: Compress with reference-based mode
qzip encode --list fastq_list.txt --output sample.qz --ref GRCh38.fa
```

**How it works:** Reads are aligned to the reference genome. Aligned reads are stored as compact alignment records (position + differences) rather than full sequences, resulting in significantly better compression for datasets with high mapping rates.

**Note:** All three index files (`.pac`, `.bwt`, `.sa`) must be present in the same directory as the reference file for compression to work.

---

#### \[-t \| --thread]

The number of parallel threads used during compression. Increasing the thread count can significantly speed up compression on multi-core systems.

**Default:** `1`

**Example:**
```
--thread 8
```

**Notes:**
- Each thread processes a separate block of reads independently.
- The effective speedup depends on the block size, compression mode, and available CPU cores.

---

#### \[-v \| --verbose]

When enabled, Qzip prints detailed progress information during the encoding process, including per-block statistics, compression ratios, and elapsed time for each stage.

**Default:** Disabled (quiet mode)



## 3.2 Command: `decode`

Decompress a QZ file back to the original FASTQ file(s).

```
Usage: qzip decode [options]
```

### 3.2.1 Options Summary

| Option           | Argument | Description                                                                     |
|------------------|----------|---------------------------------------------------------------------------------|
| `-h`, `--help`   | —        | Print help information for the decode command                                   |
| `-i`, `--input`  | FILE     | **\[Required]** Input compressed QZ file                                        |
| `-o`, `--output` | DIR      | **\[Required]** Output directory for decoded FASTQ file(s)                      |
| `-r`, `--ref`    | FILE     | Prefix of the reference genome index (.pac) — required only for reference-based |
| `-t`, `--thread` | INT      | Number of threads to use (default: `1`)                                         |
| `-z`, `--gzip`   | —        | Output gzip-compressed FASTQ instead of plain FASTQ                             |

### 3.2.2 Option Details

#### \[-h \| --help]

Print the help information for the decode command and exit.

---

#### \[-i \| --input]

The path of the input compressed QZ file to be decompressed.

**Example:**
```
--input /home/user/data/sample.qz
```

---

#### \[-o \| --output]

The output directory where the decoded FASTQ file(s) will be written. The output file name(s) are derived from the original file name(s) stored inside the QZ archive.

**Example:**
```
--output /home/user/data/decoded/
```

**Note:** The directory must exist before running the decode command.

---

#### \[-r \| --ref]

Specifies the prefix of the reference genome index file (`.pac`) used during decompression of reference-based compressed QZ files. Unlike encoding, only the `.pac` file is required for decoding — the `.bwt` and `.sa` index files are not needed.

**Prerequisites:** This option is **only required** when the QZ file was compressed with the reference-based algorithm. You can check whether a QZ file was compressed with reference-based mode by running `qzip info`.

**Example:**
```bash
qzip decode --input sample.qz --output ./out/ --ref GRCh38.fa
```

**Note:** If you omit `--ref` for a reference-based compressed file, Qzip cannot properly reconstruct the original sequences and will report an error.

---

#### \[-t \| --thread]

The number of parallel threads used during decompression.

**Default:** `1`

---

#### \[-z \| --gzip]

When enabled, the decoded output will be written as gzip-compressed FASTQ files (`.fastq.gz`) instead of plain FASTQ files. This is useful when you want to save disk space for the decompressed output.

**Default:** Disabled (output plain FASTQ)

**Example:**
```
qzip decode --input sample.qz --output ./out/ --gzip
```



## 3.3 Command: `build`

Build index files for a reference genome, required for reference-based compression.

```
Usage: qzip build [options]
```

### 3.3.1 Options Summary

| Option         | Argument | Description                                                      |
|----------------|----------|------------------------------------------------------------------|
| `-h`, `--help` | —        | Print help information for the build command                     |
| `-r`, `--ref`  | FILE     | **\[Required]** Reference genome in FASTA format (.fa or .fa.gz) |

### 3.3.2 Option Details

#### \[-h \| --help]

Print the help information for the build command and exit.

---

#### \[-r \| --ref]

The path to the reference genome file in FASTA format. Both plain (`.fa` / `.fasta`) and gzipped (`.fa.gz` / `.fasta.gz`) FASTA files are supported.

**Example:**
```
qzip build --ref /home/user/reference/GRCh38.fa
```

### 3.3.3 Output

Three index files are generated in the same directory as the input reference file:

| Index File  | Description                                                               |
|-------------|---------------------------------------------------------------------------|
| `<ref>.pac` | 2-bit packed reference sequence — required for both encoding and decoding |
| `<ref>.bwt` | Burrows-Wheeler Transform index — required for encoding only              |
| `<ref>.sa`  | Suffix Array index — required for encoding only                           |

**Note:** Building the index is a one-time operation per reference genome. Once built, the index files can be reused for all subsequent compression and decompression tasks using that reference.



## 3.4 Command: `info`

Display the basic header information of a QZ file, such as software version, original file name, block count, compression mode, and other metadata.

```
Usage: qzip info [options]
```

### 3.4.1 Options Summary

| Option           | Argument | Description                                       |
|------------------|----------|---------------------------------------------------|
| `-h`, `--help`   | —        | Print help information for the info command       |
| `-i`, `--input`  | FILE     | **\[Required]** Input compressed QZ file          |

### 3.4.2 Option Details

#### \[-h \| --help]

Print the help information for the info command and exit.

---

#### \[-i \| --input]

The path of the QZ file whose header information you want to inspect.

**Example:**
```
qzip info --input sample.qz
```

**Typical output includes:**
- Qzip software version used for compression
- Original FASTQ file name(s)
- Number of blocks
- Compression mode (reference-free or reference-based)
- Sequence encoding method (BWT, BIT2, or Reference-based)
- Block size (if BWT mode)
- Total number of reads



## 3.5 Command: `validate`

Validate the structural integrity and (optionally) the data content consistency of a QZ file.

```
Usage: qzip validate [options]
```

### 3.5.1 Options Summary

| Option           | Argument | Description                                                                     |
|------------------|----------|---------------------------------------------------------------------------------|
| `-h`, `--help`   | —        | Print help information for the validate command                                 |
| `-i`, `--input`  | FILE     | **\[Required]** Input compressed QZ file                                        |
| `-r`, `--ref`    | FILE     | Prefix of the reference genome index (.pac) — required only for reference-based |
| `-d`, `--deep`   | —        | Perform deep data content consistency check (default: structural check only)    |
| `-t`, `--thread` | INT      | Number of threads to use (default: `1`)                                         |

### 3.5.2 Option Details

#### \[-h \| --help]

Print the help information for the validate command and exit.

---

#### \[-i \| --input]

The path of the QZ file to validate.

**Example:**
```
qzip validate --input sample.qz
```

---

#### \[-r \| --ref]

Specifies the prefix of the reference genome index file (`.pac`) required for validating reference-based compressed QZ files. Same requirement as the decode command — only the `.pac` file is needed.

**Prerequisites:** Only required when the QZ file was compressed with the reference-based algorithm. Check with `qzip info` if unsure.

---

#### \[-d \| --deep]

When enabled, Qzip performs a thorough byte-level consistency check of the data content in addition to the default structural integrity check. The deep check verifies that the decompressed data matches the original data by comparing checksums and data signatures stored in the QZ file.

**Default:** Disabled (structural check only)

**Performance note:** Enabling deep validation is more time-consuming than the default structural check, as it requires fully decompressing and verifying each block. Use it when you need maximum confidence in data integrity (e.g., after transferring files across systems or long-term archival retrieval).

---

#### \[-t \| --thread]

The number of parallel threads used during validation.

**Default:** `1`



## 3.6 Command: `view`

Quickly view a subset of reads from a QZ file. *(Under development)*

```
Usage: qzip view [options]
```

### 3.6.1 Options Summary

| Option           | Argument | Description                                       |
|------------------|----------|---------------------------------------------------|
| `-h`, `--help`   | —        | Print help information for the view command       |
| `-i`, `--input`  | FILE     | **\[Required]** Input compressed QZ file          |

### 3.6.2 Option Details

#### \[-h \| --help]

Print the help information for the view command and exit.

---

#### \[-i \| --input]

The path of the QZ file from which reads will be displayed. *(This command is under development and may have limited functionality.)*



# 4. Examples


## 4.1 Command: `encode`

### 4.1.1 Basic Compression (Reference-free, BWT mode)

Compress a pair of FASTQ files using the default BWT algorithm:

```bash
# Create a file list
cat > fastq_list.txt << EOF
/home/user/data/sample_R1.fastq.gz
/home/user/data/sample_R2.fastq.gz
EOF

# Compress
qzip encode --list fastq_list.txt --output sample.qz --thread 8
```

### 4.1.2 Fast Compression (Reference-free, BIT2 mode)

Use BIT2 mode for faster compression with lower memory usage:

```bash
qzip encode --list fastq_list.txt --output sample.qz --mode BIT2 --thread 8
```

### 4.1.3 Reference-based Compression (Best Compression Ratio)

First build the reference index, then compress using the reference:

```bash
# Step 1: Build index (one-time per reference)
qzip build --ref GRCh38.fa

# Step 2: Compress with reference
qzip encode --list fastq_list.txt --output sample.qz --ref GRCh38.fa --thread 8
```

### 4.1.4 Adjust BWT Block Size

Use a smaller BWT block size to reduce memory usage:

```bash
qzip encode --list fastq_list.txt --output sample.qz --mode BWT --size 16 --thread 4
```


## 4.2 Command: `decode`

### 4.2.1 Decompress to Plain FASTQ

```bash
qzip decode --input sample.qz --output ./decoded/ --thread 8
```

### 4.2.2 Decompress to Gzipped FASTQ

```bash
qzip decode --input sample.qz --output ./decoded/ --gzip --thread 8
```

### 4.2.3 Decompress Reference-based QZ File

```bash
qzip decode --input sample.qz --output ./decoded/ --ref GRCh38.fa --thread 8
```


## 4.3 Command: `build`

### 4.3.1 Build Reference Index

```bash
qzip build --ref /home/user/reference/GRCh38.fa
```


## 4.4 Command: `info`

### 4.4.1 Inspect QZ File Metadata

```bash
qzip info --input sample.qz
```


## 4.5 Command: `validate`

### 4.5.1 Quick Data Block Check

```bash
# add option '--ref GRCh38.fa' for Reference-based QZ file
qzip validate --input sample.qz
```

### 4.5.2 Deep Content Check

```bash
# add option '--ref GRCh38.fa' for Reference-based QZ file
qzip validate --input sample.qz --deep --thread 4
```


# 5. Input and Output


## 5.1 FASTQ Input Format

Qzip accepts both plain FASTQ (`.fastq`, `.fq`) and gzip-compressed FASTQ (`.fastq.gz`, `.fq.gz`) as input. Each FASTQ file should follow the standard four-line-per-read format:

```
@read_identifier
ACGTACGTACGT...
+
IIIIIIIIIIII...
```

## 5.2 QZ File Format

The compressed output file uses the `.qz` extension. A QZ file contains:

- **Header block:** Stores metadata including software version, original file name(s), compression mode, block count, and checksums.
- **Data blocks:** Each block contains independently compressed reads (sequences + quality scores + identifiers), enabling parallel compression and decompression.

## 5.3 QZ File Integrity

After compression, you can verify the integrity of a QZ file using the `validate` command. Qzip stores internal checksums for each data block, enabling detection of file corruption or incomplete transfers.



# 6. Compression Modes Comparison


| Feature             | BIT2                           | BWT (default)                    | Reference-based                           |
|---------------------|--------------------------------|----------------------------------|-------------------------------------------|
| Compression Ratio   | Good                           | Better                           | Best                                      |
| Compression Speed   | Fast                           | Moderate                         | Moderate                                  |
| Decompression Speed | Fast                           | Moderate                         | Moderate                                  |
| Memory Usage        | Low                            | High                             | Moderate                                  |
| Reference Required  | No                             | No                               | Yes                                       |
| Best For            | Quick compression, limited RAM | Archival, maximum ratio (no ref) | Production pipelines with known reference |

## 6.1 When to Use Each Mode

- **BIT2:** When you need fast turnaround, have limited system memory, or the compression ratio is not the primary concern.
- **BWT:** When you want the best possible compression ratio without a reference genome, or when storing data for long-term archival.
- **Reference-based:** When you have a known reference genome and want the absolute best compression ratio — ideal for production pipelines processing large volumes of human resequencing data.



# 7. Citation

If you find Qzip useful in your research, please cite the following article:

*To be added upon publication.*
