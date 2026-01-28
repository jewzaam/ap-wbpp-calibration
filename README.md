# Master Calibration Frame Automation for PixInsight

Fully automated generation of master bias, dark, and flat calibration frames for PixInsight WBPP workflows.

## Overview

This automation tool:
- Automatically discovers and groups calibration frames by FITS keywords
- Generates master bias, dark, and flat frames using PixInsight's ImageIntegration process
- Calibrates flat frames with bias/dark masters using ImageCalibration with dark optimization
- Generates timestamped PixInsight JavaScript scripts for full reproducibility
- Captures execution logs with timestamps for audit trails
- Runs completely hands-off from the command line
- Processes all filters and instrument configurations automatically

## Requirements

- Python 3.9 or later
- PixInsight installed
- `ap-common` package for FITS header processing
- Calibration frames with proper FITS keywords (see Frame Grouping section)

## Installation

Install using pip:

```bash
pip install git+https://github.com/jewzaam/ap-wbpp-calibration.git
```

Or install in development mode:

```bash
git clone https://github.com/jewzaam/ap-wbpp-calibration.git
cd ap-wbpp-calibration
pip install -e .
```

## Quick Start

### 1. Organize Your Frames

Place all calibration frames in a directory. The tool will automatically discover and group them by the `IMAGETYP` FITS keyword:
- `IMAGETYP=bias` - Bias frames
- `IMAGETYP=dark` - Dark frames
- `IMAGETYP=flat` - Flat frames
- `IMAGETYP=light` - Light frames (automatically ignored)

No specific subdirectory structure is required - the tool scans recursively.

### 2. Run the Automation

**Basic usage (generate bias/dark masters):**
```bash
ap-wbpp-calibration \
    <input_dir> \
    <output_dir> \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

**Generate flat masters with existing library:**
```bash
ap-wbpp-calibration \
    <input_dir> \
    <output_dir> \
    --bias-master-dir "D:\Masters\Bias" \
    --dark-master-dir "D:\Masters\Darks" \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

**Generate scripts only (without executing):**
```bash
ap-wbpp-calibration <input_dir> <output_dir> --script-only
```

**Example: Build bias library:**
```bash
ap-wbpp-calibration \
    "D:\AstroData\2026-01-27\bias" \
    "D:\MasterLibrary\bias" \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

**Example: Build flats using library (Linux/Mac):**
```bash
ap-wbpp-calibration \
    ~/AstroData/2026-01-27/flats \
    ~/AstroData/2026-01-27/output \
    --bias-master-dir ~/MasterLibrary/bias \
    --dark-master-dir ~/MasterLibrary/darks \
    --pixinsight-binary /opt/PixInsight/bin/PixInsight
```

> **Note:** To calibrate flats with bias/dark masters, you need existing masters in a library. Newly created masters in the same run are not used for flat calibration. See [Workflow: Building vs. Using Masters](#workflow-building-vs-using-masters) for detailed multi-stage workflows.

## Command Line Options

```
ap-wbpp-calibration [-h] [--bias-master-dir BIAS_MASTER_DIR]
                    [--dark-master-dir DARK_MASTER_DIR]
                    [--script-dir SCRIPT_DIR]
                    [--pixinsight-binary PIXINSIGHT_BINARY]
                    [--instance-id INSTANCE_ID]
                    [--no-force-exit]
                    [--script-only]
                    input_dir output_dir

positional arguments:
  input_dir             Input directory containing calibration frames
  output_dir            Base output directory

optional arguments:
  -h, --help            Show help message and exit
  --bias-master-dir     Directory containing bias master library (for flat calibration)
  --dark-master-dir     Directory containing dark master library (for flat calibration)
  --script-dir          Directory for scripts and logs (default: output_dir/logs)
  --pixinsight-binary   Path to PixInsight binary (required unless --script-only)
  --instance-id         PixInsight instance ID (default: 123)
  --no-force-exit       Keep PixInsight open after execution completes
  --script-only         Generate scripts only, do not execute PixInsight
```

## Output Structure

The tool creates an organized directory structure:

```
output_dir/
├── master/                          # Master calibration frames
│   ├── masterBias_<metadata>.xisf
│   ├── masterDark_<metadata>.xisf
│   ├── masterFlat_<metadata>.xisf
│   └── calibrated/                  # Calibrated flat frames (if using bias/dark)
│       └── <group_name>/
│           └── *_c.xisf
└── logs/                            # Scripts and execution logs
    ├── 20260127_143052_calibrate_masters.js
    └── 20260127_143052.log
```

**Timestamped files:** Both the generated script and execution log use the same timestamp (`YYYYMMDD_HHMMSS`) for easy correlation.

## How It Works

### Master Bias
1. Discovers all bias frames in input directory
2. Groups by instrument settings (camera, temperature, gain, offset, readout mode)
3. Integrates each group using ImageIntegration with no normalization
4. Saves as `masterBias_<metadata>.xisf`

### Master Dark
1. Discovers all dark frames
2. Groups by instrument settings and exposure time
3. Integrates each group using ImageIntegration with no normalization
4. Saves as `masterDark_<metadata>.xisf`

### Master Flats
1. Discovers all flat frames
2. Groups by instrument settings, date, and filter
3. For each group:
   - **If bias/dark masters provided**: Calibrates flats using ImageCalibration with dark optimization
   - Integrates calibrated (or raw) flats using ImageIntegration with multiplicative normalization
   - Saves as `masterFlat_<metadata>.xisf`

The metadata in filenames includes all grouping keywords to ensure uniqueness and traceability.

## Frame Grouping

Frames are automatically grouped by FITS keywords to ensure only compatible frames are combined:

**Bias frames** grouped by:
- `INSTRUME` - Camera model
- `SET-TEMP` / `SETTEMP` - Set temperature
- `GAIN` - Gain setting
- `OFFSET` - Offset setting
- `READOUTM` - Readout mode

**Dark frames** grouped by:
- All bias grouping criteria above, plus:
- `EXPOSURE` / `EXPTIME` - Exposure time in seconds

**Flat frames** grouped by:
- All bias grouping criteria above, plus:
- `DATE-OBS` - Observation date
- `FILTER` - Filter name

The tool uses `ap-common` for FITS header normalization, handling various keyword conventions automatically.

See `ap_wbpp_calibration/config.py` for the complete configuration.

## Flat Calibration with Dark Optimization

When `--bias-master-dir` and `--dark-master-dir` are specified, the tool:

1. Finds matching bias and dark masters for each flat group based on instrument settings
2. Uses ImageCalibration with `optimizeDarks=true` to calibrate each flat frame
3. Dark optimization rescales the master dark to match each flat's thermal noise
4. Handles exposure time variations between flats and darks automatically

This is equivalent to WBPP's flat calibration with optimize enabled, but works outside the WBPP workflow.

## Master Library Matching

When searching for bias/dark masters in library directories:

- Masters are matched **only by instrument settings** (camera, temperature, gain, offset, readout mode)
- Date and filter are ignored (they vary per flat group)
- Dark masters with lower or equal exposure time are preferred
- If no lower exposure dark exists, the next higher exposure is used
- Master filenames can use any convention - the tool reads FITS headers for matching

This allows maintaining a single master library that can be reused across observation sessions.

## Workflow: Building vs. Using Masters

**Important:** Master matching happens at **script generation time**, not at runtime. This means:

- **Newly created masters in the same run are NOT used for flat calibration**
- You must use existing masters from a library, or run the tool in stages
- This is **by design** - it keeps the tool predictable and focused on a single task per run

### Typical Workflows

**Building a Complete Master Library (3 separate runs):**

```bash
# Stage 1: Generate bias masters
ap-wbpp-calibration ./calibration/bias ./masters/bias \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"

# Stage 2: Generate dark masters
ap-wbpp-calibration ./calibration/darks ./masters/darks \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"

# Stage 3: Generate flat masters using library
ap-wbpp-calibration ./calibration/flats ./masters/flats \
    --bias-master-dir ./masters/bias \
    --dark-master-dir ./masters/darks \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

**Using an Existing Master Library (single run):**

```bash
# Generate flats using existing bias/dark library
ap-wbpp-calibration ./new_flats ./output \
    --bias-master-dir ~/MasterLibrary/bias \
    --dark-master-dir ~/MasterLibrary/darks \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

**Generating Uncalibrated Flat Masters (single run):**

```bash
# Generate flat masters without calibration
ap-wbpp-calibration ./flats ./output \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

### Why This Design?

This approach follows the **single responsibility principle**:
- Each run has a clear, focused purpose
- Master matching is explicit and predictable
- You control exactly what masters are used
- No hidden dependencies or unexpected behavior
- Matches typical astrophotography workflows where bias/dark libraries are built once and reused

## Logs and Reproducibility

The tool generates timestamped artifacts for full reproducibility:

- **JavaScript script** (`<timestamp>_calibrate_masters.js`): Complete PixInsight script showing all parameters
- **Execution log** (`<timestamp>.log`): Full console output from PixInsight execution

Both files use the same timestamp for easy correlation. The script can be manually re-run in PixInsight for debugging or customization.

## Troubleshooting

### "No frames found"
- Verify files have `.fit` or `.fits` extensions
- Check that files have the `IMAGETYP` FITS keyword set correctly (`bias`, `dark`, or `flat`)
- Ensure the input directory path is correct
- Try running with increased verbosity to see discovered files

### "PixInsight not found"
- Verify the `--pixinsight-binary` path is correct
- On Windows: `C:\Program Files\PixInsight\bin\PixInsight.exe`
- On Linux/Mac: `/opt/PixInsight/bin/PixInsight` or `/Applications/PixInsight/bin/PixInsight`
- Use quotes around paths with spaces

### "No matching master found for flat calibration"
- Check that bias/dark masters have matching instrument settings in their FITS headers
- Masters must match: `INSTRUME`, `SET-TEMP`, `GAIN`, `OFFSET`, `READOUTM`
- Date and filter differences are OK - they're expected to vary
- Use `--script-only` to generate the script and check the master paths

### PixInsight execution fails
- Check the generated script at `<output_dir>/logs/<timestamp>_calibrate_masters.js`
- Review the execution log at `<output_dir>/logs/<timestamp>.log`
- Verify PixInsight can execute the script manually: `PixInsight -r=<script_path>`
- Check that PixInsight supports the `.xisf` format (should be standard)

### "ImageIntegration: Cannot execute instance in the global context"
- This means no images were added to the integration
- Check the log for the number of images found
- Verify input files exist and are readable
- Ensure files have valid FITS headers

## Project Structure

```
ap-wbpp-calibration/
├── ap_wbpp_calibration/
│   ├── __init__.py
│   ├── calibrate_masters.py    # Main CLI entry point
│   ├── config.py                # Grouping configuration
│   ├── grouping.py              # Frame grouping logic
│   ├── master_matching.py       # Master library matching
│   ├── script_generator.py      # JavaScript code generation
│   └── templates/               # Jinja2 templates for PixInsight scripts
│       ├── combined.j2
│       ├── ImageIntegration_bias.j2
│       ├── ImageIntegration_dark.j2
│       ├── ImageIntegration_flat.j2
│       └── ImageCalibration_flat.j2
├── tests/                       # Unit tests
├── examples/                    # Example PixInsight scripts from WBPP
├── README.md
└── pyproject.toml
```

## Development

Run tests:
```bash
pytest tests/
```

Run with coverage:
```bash
pytest tests/ --cov=ap_wbpp_calibration
```

Format code:
```bash
black ap_wbpp_calibration/ tests/
ruff check ap_wbpp_calibration/ tests/
```

## Notes

- The tool runs PixInsight in automation mode (`--automation-mode`) to prevent interactive dialogs
- No GUI interaction is required - fully headless execution
- All console output is captured to timestamped log files
- PixInsight exits automatically after execution (override with `--no-force-exit`)
- The tool uses PixInsight's FileFormat API to write `.xisf` files directly, avoiding format selection prompts
- Generated scripts use WBPP's exact parameter sets for compatibility

## License

See LICENSE file for details.
