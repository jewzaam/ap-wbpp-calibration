# Master Calibration Frame Automation for PixInsight

Fully automated generation of master bias, dark, and flat calibration frames using saved process icons and WBPP optimize darks functionality.

## Overview

This automation tool:
- Generates master bias frames using a saved integration process icon
- Generates master dark frames using a saved integration process icon  
- Generates master flat frames using a saved integration process icon, with automatic dark optimization per filter (via ImageCalibration with optimize enabled)
- Runs completely hands-off from the command line
- Processes all filters automatically without manual intervention

## Requirements

- PixInsight installed
- Saved process icons for:
  - Bias integration (ImageIntegration with your preferred settings)
  - Dark integration (ImageIntegration with your preferred settings)
  - Flat integration (ImageIntegration with your preferred settings) - optional
- Organized calibration frames in subdirectories:
  ```
  input_directory/
    ├── bias/     (bias frames)
    ├── darks/    (dark frames)
    └── flats/    (flat frames, can be grouped by filter)
  ```

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

Place all calibration frames in a single directory. The tool will automatically discover and group them by the `TYPE` FITS keyword:
- `TYPE=bias` - Bias frames
- `TYPE=dark` - Dark frames
- `TYPE=flat` - Flat frames
- `TYPE=light` - Light frames (automatically ignored)

No subdirectory structure is required.

### 2. Run the Automation

**Basic usage (generate and execute):**
```bash
ap-wbpp-calibration \
    <input_dir> \
    <output_dir> \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

**Generate scripts only (without executing):**
```bash
ap-wbpp-calibration <input_dir> <output_dir> --script-only
```

**With bias/dark master libraries (for flat calibration):**
```bash
ap-wbpp-calibration \
    <input_dir> \
    <output_dir> \
    --bias-master-dir "D:\Masters\Bias" \
    --dark-master-dir "D:\Masters\Darks" \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

**Example (Windows):**
```bash
ap-wbpp-calibration \
    "D:\AstroData\2026-01-27\calibration" \
    "D:\AstroData\2026-01-27\masters" \
    --pixinsight-binary "C:\Program Files\PixInsight\bin\PixInsight.exe"
```

**Example (Linux/Mac):**
```bash
ap-wbpp-calibration \
    ~/AstroData/2026-01-27/calibration \
    ~/AstroData/2026-01-27/masters \
    --pixinsight-binary /opt/PixInsight/bin/PixInsight
```

## Command Line Options

```
ap-wbpp-calibration [-h] [--bias-master-dir BIAS_MASTER_DIR]
                    [--dark-master-dir DARK_MASTER_DIR]
                    [--script-dir SCRIPT_DIR]
                    [--pixinsight-binary PIXINSIGHT_BINARY]
                    [--instance-id INSTANCE_ID]
                    [--script-only]
                    input_dir output_dir

positional arguments:
  input_dir             Input directory containing calibration frames
  output_dir            Output directory for master calibration frames

optional arguments:
  -h, --help            Show help message and exit
  --bias-master-dir     Directory containing bias master library (for flat calibration)
  --dark-master-dir     Directory containing dark master library (for flat calibration)
  --script-dir          Directory for generated PixInsight scripts (default: output_dir/scripts)
  --pixinsight-binary   Path to PixInsight binary (required unless --script-only)
  --instance-id         PixInsight instance ID (default: 123)
  --script-only         Generate scripts only, do not execute PixInsight
```

## How It Works

1. **Master Bias**: Integrates all bias frames using your saved process icon
2. **Master Dark**: Integrates all dark frames using your saved process icon
3. **Master Flats**: 
   - Groups flat frames by filter (using FITS `FILTER` keyword)
   - For each filter:
     - Integrates flat frames using your saved process icon (or defaults)
     - Calibrates the integrated master flat with optimized darks (ImageCalibration with `optimize=true`)
     - Saves as `MasterFlat_<FilterName>.fit`

## Output

Master frames are saved to the output directory:
```
output_directory/
  ├── MasterBias.fit
  ├── MasterDark.fit
  ├── MasterFlat_<Filter1>.fit
  ├── MasterFlat_<Filter2>.fit
  └── ...
```

## Frame Grouping

Frames are automatically grouped by FITS keywords to ensure only compatible frames are combined:

**Bias frames** grouped by:
- Camera (INSTRUME)
- Set Temperature (SET-TEMP)
- Gain
- Offset
- Readout Mode (READOUTM)

**Dark frames** grouped by:
- All bias grouping criteria above
- Exposure time

**Flat frames** grouped by:
- All bias grouping criteria above
- Observation date (DATE)
- Filter

See `ap_wbpp_calibration/config.py` for the complete configuration.

## Optimize Darks for Flats

The script automatically enables "Optimize" in ImageCalibration when calibrating master flats with darks. This rescales the master dark to match the thermal noise of each flat frame, handling variations in exposure time and temperature.

This is equivalent to manually checking "Optimize" in WBPP's flat calibration settings for each filter.

## Troubleshooting

### "No frames found"
- Verify files have `.fit` or `.fits` extensions
- Check that files have the `TYPE` FITS keyword set correctly
- Ensure the input directory path is correct

### "PixInsight not found"
- Verify the `--pixinsight-binary` path is correct
- On Windows: `C:\Program Files\PixInsight\bin\PixInsight.exe`
- On Linux/Mac: `/opt/PixInsight/bin/PixInsight` or `/Applications/PixInsight/bin/PixInsight`
- Use quotes around paths with spaces

### "No matching master found for flat calibration"
- Ensure bias/dark masters have matching instrument settings (camera, temperature, gain, offset, readout mode)
- Masters are matched only by instrument settings, not by date or filter
- Use `--bias-master-dir` and `--dark-master-dir` to specify master library locations

### PixInsight execution fails
- Check the generated script at `<output_dir>/scripts/calibrate_masters.js`
- Verify PixInsight can open the script manually
- Check PixInsight's console output for error messages

## Project Structure

```
ap-wbpp-calibration/
├── ap_wbpp_calibration/
│   ├── calibrate_masters.py    # Main CLI entry point
│   ├── config.py                # Grouping configuration
│   ├── grouping.py              # Frame grouping logic
│   ├── master_matching.py       # Master library matching
│   ├── script_generator.py      # JavaScript code generation
│   └── templates/               # Jinja2 templates for PixInsight scripts
├── tests/                       # Unit tests
├── examples/                    # Example PixInsight scripts
├── README.md
├── CRITICAL_INFO.md            # Implementation details
└── pyproject.toml

## Notes

- The script runs PixInsight in headless/automation mode
- No GUI interaction is required
- All processing is logged to the console
- Temporary files are automatically cleaned up
- The script exits PixInsight automatically when complete
