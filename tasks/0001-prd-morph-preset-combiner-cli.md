# Product Requirements Document: Preset Combiner CLI

## Introduction/Overview

The Preset Combiner CLI is a command-line tool designed to automate the generation of upscaler preset files by combining source presets (console/device input configurations with prescale settings) and display presets (CRT/LCD output profiles with visual effects). The tool uses an **adapter-based architecture** to support multiple upscaler formats (Morph4k, RetroTINK 4K, and potentially others in the future).

**Problem Statement**: Users currently need to manually create individual preset files for every combination of console input mode and display profile. With multiple consoles (PS1, N64, Saturn, etc.), multiple input variants (composite, RGB, HDMI mods with various output modes), and multiple display profiles (Sony PVM with different masks, BVM, consumer CRTs, LCDs), this results in hundreds of preset files that are tedious to create and maintain.

**Solution**: This CLI tool will:
1. Read source presets from a `source/` directory (organized by console/variant/input-mode)
2. Read display presets from a `display/` directory (organized by display-type)
3. Match compatible source + display combinations based on resolution compatibility
4. Generate combined preset files in a structured `generated/` output directory
5. Support generating source INI files from JSON templates for highly variable consoles
6. Allow users to configure which combinations to generate via a config file

## Goals

1. Automate the generation of combined upscaler preset files from source and display configurations
2. Reduce manual effort and human error in creating preset packs
3. **Support multiple upscaler formats** through an adapter-based architecture (Morph4k initially, RetroTINK 4K in future)
4. Support complex HDMI prescale scenarios (e.g., 240p in 1080p container with DV1/PixelFX/manual extraction)
5. Enable generation of source presets from JSON templates for consoles with many output modes
6. Provide flexible filtering so users only generate presets for their specific hardware setup
7. Create a library of reusable source and display preset files from the existing repository
8. Ensure generated presets maintain proper resolution compatibility between source and display
9. Make it easy to add support for new upscaler formats by implementing a new adapter

## User Stories

1. **As a retro gaming enthusiast with an N64 UltraHDMI mod**, I want to generate presets for all my CRT display profiles at 1080p 4x DV1 mode, so that I don't have to manually create each combination.

2. **As a preset pack creator**, I want to update a CRT mask in one display file, then regenerate all combinations for all consoles, so that I can easily distribute updated preset packs.

3. **As a user with a specific HDMI setup**, I want to edit a config file to specify I only use "1080p 4x DV1" modes, then generate only those presets, so that I don't clutter my generated folder with modes I don't use.

4. **As a developer adding N64 Digital support**, I want to define all the N64 Digital output modes in a JSON template, then generate all the source INI files automatically, so that I don't have to manually create 20+ INI files.

5. **As a user exploring CRT simulations**, I want to see which display presets are compatible with my source's operated resolution before generating, so that I understand which combinations are valid.

6. **As a power user**, I want to preview what files would be generated with dry-run mode, so that I can verify the output structure before creating hundreds of files.

7. **As a RetroTINK 4K user**, I want to use the same source and display presets but generate RT4K-format files instead of Morph4k format, so that I can maintain one library of presets for multiple devices.

8. **As a developer**, I want to implement a new adapter for a different upscaler device, so that the tool can support formats beyond Morph4k and RT4K without modifying core logic.

## Functional Requirements

### Adapter Architecture

1. **FR-1**: The tool MUST use an adapter-based architecture where all device-specific logic is isolated in adapter modules.

2. **FR-2**: Each adapter MUST implement a standard interface that includes:
   - `parseSourceFile(filePath)` - Parse source preset file into normalized format
   - `parseDisplayFile(filePath)` - Parse display preset file into normalized format
   - `mergePresets(source, display)` - Merge source and display into combined preset
   - `writePreset(preset, outputPath)` - Write combined preset to file
   - `getSectionOwnership()` - Return section ownership rules for this device
   - `validateCompatibility(source, display)` - Check if source and display are compatible

3. **FR-3**: The tool MUST support selecting an adapter via:
   - CLI flag: `--adapter morph4k` or `--adapter rt4k`
   - Config file: `combiner.config.json` with `"adapter": "morph4k"`
   - Default to `morph4k` if not specified

4. **FR-4**: Adapters MUST be located in an `adapters/` directory within the project, with each adapter in its own subdirectory (e.g., `adapters/morph4k/`, `adapters/rt4k/`).

5. **FR-5**: The initial release MUST include a fully functional Morph4k adapter.

6. **FR-6**: The tool MUST provide documentation on how to create a new adapter, including the adapter interface specification.

7. **FR-7**: When using different adapters, the tool MAY use different file formats (e.g., INI for Morph4k, JSON for RT4K) as determined by the adapter implementation.

### Core Functionality

8. **FR-8**: The tool MUST read source preset files from a `source/` directory organized in the structure:
   ```
   source/
   ├── {Console}/
   │   ├── {input-type}/
   │   │   └── {mode}.ini
   ```
   Example: `source/N64-UltraHDMI/1080p-4x-dv1/1080p-4x-dv1.ini`

9. **FR-9**: The tool MUST read display preset files from a `display/` directory organized in the structure:
   ```
   display/
   ├── {display-type}/
   │   └── {display-name}.[ext]
   ```
   Example: `display/crt/Sony PVM 14.ini` (file extension determined by adapter)

10. **FR-10**: The tool MUST support optional JSON metadata files alongside preset files (same filename, `.json` extension) containing:
    - Source metadata: `display_name`, `console`, `variant`, `input_type`, `operated_resolution`, etc.
    - Display metadata: `display_name`, `display_type`, `compatible_resolutions`, `tags`, etc.

11. **FR-11**: The tool MUST generate combined preset files for every valid source × display combination into a `generated/` directory with structure:
    ```
    generated/
    ├── {Console}/
    │   ├── {input-type}/
    │   │   ├── {display-type}/
    │   │   │   └── {Console} {input-type} - {display-name}.[ext]
    ```
    Example: `generated/N64-UltraHDMI/1080p-4x-dv1/crt/N64-UltraHDMI 1080p-4x-dv1 - Sony PVM 14.ini`

12. **FR-12**: When merging presets, the adapter MUST:
    - Use display preset as the base
    - Overlay source preset sections
    - Preserve sections according to the adapter's ownership rules
    - Update metadata section with combined information

13. **FR-13**: The tool MUST validate compatibility before generating combinations:
    - If source JSON specifies `operated_resolution` AND display JSON specifies `compatible_resolutions`
    - Then source's `operated_resolution` MUST be in display's `compatible_resolutions` array
    - Otherwise, skip the combination and log a warning

14. **FR-14**: If a source or display file is malformed or missing required sections, the tool MUST skip that file, log a warning message, and continue processing other files.

### Source Generation from Templates

15. **FR-15**: The tool MUST support a `source-templates/` directory containing JSON files that define multiple source preset variations.

16. **FR-16**: A source template JSON MUST allow defining multiple output modes for a single device:
   ```json
   {
     "console": "N64-UltraHDMI",
     "base_settings": { /* common INI sections */ },
     "modes": [
       {
         "name": "1080p-4x-dv1",
         "display_name": "N64 UltraHDMI (1080p 4x with DV1)",
         "operated_resolution": "240p",
         "settings": { /* mode-specific INI sections */ }
       },
       {
         "name": "720p-3x-dv1",
         "display_name": "N64 UltraHDMI (720p 3x with DV1)",
         "operated_resolution": "240p",
         "settings": { /* mode-specific INI sections */ }
       }
     ]
   }
   ```

17. **FR-17**: The tool MUST provide a `generate-sources` command that reads source templates and outputs source preset files + JSON files to the `source/` directory.

18. **FR-18**: When generating source files from templates, the tool MUST:
    - Create directory structure: `source/{console}/{mode-name}/`
    - Generate preset file with merged base_settings + mode settings (format determined by adapter)
    - Generate `{mode-name}.json` with mode metadata

### User Configuration

19. **FR-19**: The tool MUST support a `combiner.config.json` file that allows users to filter which combinations to generate:
    ```json
    {
      "enabled_sources": ["N64-UltraHDMI/1080p-4x-dv1", "PS1/rgb"],
      "enabled_displays": ["crt/*", "lcd/Modern-4K"],
      "compatibility_strict": true
    }
    ```

20. **FR-20**: If `combiner.config.json` exists, the tool MUST only generate combinations where:
    - Source path matches a pattern in `enabled_sources` (supports wildcards)
    - Display path matches a pattern in `enabled_displays` (supports wildcards)
    - If `compatibility_strict` is true, resolution compatibility is enforced

21. **FR-21**: If `combiner.config.json` does not exist, the tool MUST generate all valid combinations.

### CLI Interface

22. **FR-22**: The tool MUST be invoked via command-line (e.g., `npm run combine` or `preset-combiner`).

23. **FR-23**: The tool MUST support the following commands:
    - `combine` - Generate combined presets (default command)
    - `generate-sources` - Generate source INI files from templates
    - `validate` - Check compatibility without generating files
    - `list` - List all possible combinations

24. **FR-24**: The `combine` command MUST support these flags:
    - `--adapter <name>` - Specify which adapter to use (morph4k, rt4k, etc.)
    - `--dry-run` - Preview what would be generated without creating files
    - `--verbose` - Output detailed logging
    - `--no-overwrite` - Don't overwrite existing generated files
    - `--source <pattern>` - Filter to specific source patterns
    - `--display <pattern>` - Filter to specific display patterns

25. **FR-25**: The `generate-sources` command MUST support:
    - `--adapter <name>` - Specify which adapter format to use
    - `--template <name>` - Generate from specific template
    - `--all` - Generate from all templates in `source-templates/`
    - `--overwrite` - Overwrite existing source files

26. **FR-26**: The tool MUST provide a `--help` flag for all commands.

### Section Ownership (Morph4k Adapter)

27. **FR-27**: The Morph4k adapter MUST use the following section ownership rules when merging:

**Source Sections** (from source preset):
- `[dv1]` - DV1 device detection
- `[rx]` - Receiver/input settings
- `[fingerprint]` - Resolution fingerprints
- `[analog]` - Analog signal processing
- `[prescale]` - Prescale settings for HDMI container extraction

**Display Sections** (from display preset):
- `[slotmask]` - CRT mask patterns
- `[vertical-scanlines]` - Vertical scanline settings
- `[horizontal-scanlines]` - Horizontal scanline settings
- `[color_csc_matrix]` - Color space conversion matrix
- `[color_input_table]` - Color input table
- `[color_output_table]` - Color output table
- `[scaler]` - Scaling/zoom/aspect settings
- `[tx]` - Transmitter/output settings
- `[post-processing]` - Post-processing effects

**Merged Sections** (combine from both, source takes precedence on conflicts):
- `[main]` - Metadata (display_name should be regenerated)
- `[shift-crop]` - Shift and crop settings
- `[deinterlacer]` - Deinterlacing settings
- `[pre-processing]` - Pre-processing settings
- `[video-misc]` - Miscellaneous video settings

28. **FR-28**: The section ownership list for each adapter MUST be configurable via an adapter-specific configuration file or as part of the adapter implementation.

### Output and Reporting

29. **FR-29**: The tool MUST display a summary report after execution showing:
    - Number of source files found
    - Number of display files found
    - Number of valid combinations (after compatibility filtering)
    - Number of presets generated successfully
    - Number of combinations skipped (with reasons)
    - Execution time
    - Current adapter being used

30. **FR-30**: In `--verbose` mode, the tool MUST log:
    - Current adapter being used
    - Each source file being read
    - Each display file being read
    - Compatibility check results for each combination
    - Which sections are being merged from each file (adapter-specific)
    - Output file path for each generated preset

31. **FR-31**: If no source files are found, the tool MUST display an error and exit.

32. **FR-32**: If no display files are found, the tool MUST display an error and exit.

33. **FR-33**: The tool MUST create output directories as needed.

### File Processing

34. **FR-34**: Adapters SHOULD preserve file formatting and comments where possible when merging presets.

35. **FR-35**: The Morph4k adapter MUST handle multi-line values (like `data` field in `[slotmask]` sections with `<<EOF` markers) correctly.

36. **FR-36**: Adapters MUST support both Unix (LF) and Windows (CRLF) line endings.

37. **FR-37**: When generating metadata sections, adapters MUST create combined metadata from source and display information (format determined by adapter).
    - `display_name` = `{source.display_name} - {display.display_name}`
    - `description` = Auto-generated description mentioning source and display
    - Preserve other `[main]` fields from display preset

## Non-Goals (Out of Scope)

1. **GUI Interface**: This tool will not include a graphical user interface. It is strictly CLI-based.

2. **Preset Validation**: The tool will not validate that generated presets are functionally correct for Morph4k hardware. It assumes input files are valid.

3. **Live Device Detection**: The tool will not detect connected hardware or recommend presets based on hardware.

4. **Preset Testing**: The tool will not test generated presets on actual hardware.

5. **File Editing**: The tool will not modify existing source or display preset files (only reads).

6. **Web Service**: The tool will not provide a web API or server interface.

7. **Binary Formats**: The tool will not support binary preset formats. Adapters work with text-based formats only (INI, JSON, XML, etc.).

8. **Automatic Section Detection**: The tool will not automatically determine section ownership by analyzing files. Section ownership must be explicitly defined in each adapter.

9. **Multiple Adapters Simultaneously**: The tool will not generate presets for multiple adapter formats in a single run. Users must specify one adapter per execution.

## Design Considerations

### Adapter Architecture

The tool uses an adapter pattern to isolate device-specific logic. This enables:

1. **Separation of Concerns**: Core combination logic is independent of file format details
2. **Extensibility**: New upscaler formats can be added without modifying core code
3. **Maintainability**: Each adapter is self-contained and can be updated independently
4. **Testability**: Adapters can be tested in isolation

**Adapter Interface** (TypeScript):

```typescript
interface PresetAdapter {
  // Adapter metadata
  readonly name: string;
  readonly version: string;
  readonly fileExtension: string;

  // File operations
  parseSourceFile(filePath: string): Promise<SourcePreset>;
  parseDisplayFile(filePath: string): Promise<DisplayPreset>;
  mergePresets(source: SourcePreset, display: DisplayPreset): CombinedPreset;
  writePreset(preset: CombinedPreset, outputPath: string): Promise<void>;

  // Configuration
  getSectionOwnership(): SectionOwnership;
  validateCompatibility(source: SourcePreset, display: DisplayPreset): ValidationResult;

  // Template generation (optional)
  generateSourcesFromTemplate?(template: SourceTemplate): Promise<SourcePreset[]>;
}
```

**Adapter Directory Structure**:

```
src/adapters/
├── index.ts                    # Adapter registry
├── adapter-interface.ts        # Adapter interface definition
├── morph4k/
│   ├── index.ts                # Morph4k adapter implementation
│   ├── parser.ts               # INI parser with EOF support
│   ├── merger.ts               # Section merging logic
│   ├── section-ownership.ts    # Morph4k section rules
│   └── types.ts                # Morph4k-specific types
└── rt4k/
    ├── index.ts                # RT4K adapter implementation (future)
    ├── parser.ts               # RT4K format parser
    └── types.ts                # RT4K-specific types
```

### Directory Structure

```
morph4k-presets/
├── source/                     # Source presets (input configurations)
│   ├── N64/
│   │   ├── composite/
│   │   │   ├── composite.ini
│   │   │   └── composite.json
│   │   └── rgb/
│   │       ├── rgb.ini
│   │       └── rgb.json
│   ├── N64-UltraHDMI/
│   │   ├── 1080p-4x-dv1/
│   │   │   ├── 1080p-4x-dv1.ini
│   │   │   └── 1080p-4x-dv1.json
│   │   └── 720p-3x-dv1/
│   │       ├── 720p-3x-dv1.ini
│   │       └── 720p-3x-dv1.json
│   └── Analogue-Pocket/
│       ├── GBA-1080p-dv1/
│       └── GB-1080p-dv1/
├── display/                    # Display presets (output profiles)
│   ├── crt/
│   │   ├── Sony PVM 14.ini
│   │   ├── Sony PVM 14.json
│   │   ├── BVM Mask.ini
│   │   └── BVM Mask.json
│   └── lcd/
│       ├── Modern 4K.ini
│       └── Modern 4K.json
├── source-templates/           # Templates for generating source presets
│   ├── N64-UltraHDMI.json
│   └── Analogue-Pocket.json
├── generated/                  # Auto-generated combined presets
│   ├── N64-UltraHDMI/
│   │   └── 1080p-4x-dv1/
│   │       ├── crt/
│   │       │   ├── N64-UltraHDMI 1080p-4x-dv1 - Sony PVM 14.ini
│   │       │   └── N64-UltraHDMI 1080p-4x-dv1 - BVM Mask.ini
│   │       └── lcd/
│   │           └── N64-UltraHDMI 1080p-4x-dv1 - Modern 4K.ini
│   └── N64/
│       └── composite/
│           └── crt/
│               └── N64 composite - Sony PVM 14.ini
├── combiner.config.json        # User configuration (optional)
├── section-ownership.json      # Section ownership rules (optional)
└── ...
```

### Example Source Preset

**File**: `source/N64-UltraHDMI/1080p-4x-dv1/1080p-4x-dv1.ini`

```ini
[dv1]
detect.name = N64
map.ar = 4/3

[rx]
input_cs = auto
edid_file = default.bin
fxd_mode = on
dv1_mode = on

[prescale]
h = 4
v = 4/1

[fingerprint]
name = N64:240p-in-1080p
fingerprint = 67.5+0.4,1125+2,0,-,-,-,-,-
settings = 2200,1920,1080,192,41,16/9,0,0/1,-1,0.0,0,0,-1,0,0,0,0,0,0,0,0,0,3,9,0,0,16,16,16,1,11,68,1,0,1,0,6,0
```

**File**: `source/N64-UltraHDMI/1080p-4x-dv1/1080p-4x-dv1.json`

```json
{
  "display_name": "N64 UltraHDMI (1080p 4x with DV1)",
  "console": "N64",
  "variant": "UltraHDMI",
  "input_type": "digital",
  "operated_resolution": "240p",
  "hdmi_container": "1080p",
  "prescale_method": "dv1"
}
```

### Example Display Preset

**File**: `display/crt/Sony PVM 14.ini`

```ini
[slotmask]
intensity = 1
conversion = none
data = <<EOF
vMFX
# mode: 1
# w,h
15,15
# lut
0x000040,0x000040,0x400000...
EOF

[vertical-scanlines]
mode = off

[scaler]
zoom_factor = fill
aspect_ratio = auto

[color_csc_matrix]
saturation = 1
rgb2xyz = 0.412...|0.357...|...

[tx]
hdr_mode = disabled
quantization_range = full
```

**File**: `display/crt/Sony PVM 14.json`

```json
{
  "display_name": "Sony PVM 14M2U",
  "display_type": "crt",
  "compatible_resolutions": ["240p", "480i"],
  "tags": ["professional-monitor", "aperture-grille"]
}
```

### Example Generated Preset

**File**: `generated/N64-UltraHDMI/1080p-4x-dv1/crt/N64-UltraHDMI 1080p-4x-dv1 - Sony PVM 14.ini`

```ini
[main]
display_name = N64 UltraHDMI (1080p 4x with DV1) - Sony PVM 14M2U
description = Combined preset for N64 UltraHDMI at 1080p 4x with DV1 prescale, output to Sony PVM 14M2U CRT

[dv1]
detect.name = N64
map.ar = 4/3

[rx]
input_cs = auto
edid_file = default.bin
fxd_mode = on
dv1_mode = on

[prescale]
h = 4
v = 4/1

[slotmask]
intensity = 1
conversion = none
data = <<EOF
vMFX
...
EOF

[scaler]
zoom_factor = fill
aspect_ratio = auto

[color_csc_matrix]
saturation = 1
rgb2xyz = 0.412...|0.357...|...

[tx]
hdr_mode = disabled
quantization_range = full

[fingerprint]
name = N64:240p-in-1080p
fingerprint = 67.5+0.4,1125+2,0,-,-,-,-,-
settings = 2200,1920,1080,192,41,16/9,0,0/1,-1,0.0,0,0,-1,0,0,0,0,0,0,0,0,0,3,9,0,0,16,16,16,1,11,68,1,0,1,0,6,0
```

## Technical Considerations

### Technology Stack
- **Language**: Node.js with TypeScript
- **Architecture**: Adapter pattern for device-specific logic
- **Programming Paradigm**: Functional programming principles
- **Development Methodology**: Test-Driven Development (TDD)
- **Testing Framework**: Vitest for unit and integration tests
- **INI Parsing** (Morph4k adapter): Robust INI parser supporting multi-line values and EOF markers
- **CLI Framework**: `commander` for command parsing
- **File System**: Node.js `fs/promises` for async operations
- **JSON Schema**: Validation for JSON metadata files

### Key Technical Challenges

1. **Adapter Architecture**: Design a clean, extensible adapter interface that accommodates different file formats and merging strategies

2. **HDMI Prescale Understanding**: The tool must correctly handle the concept of "operated resolution" vs "HDMI container resolution"

3. **Multi-line INI Values** (Morph4k): Must parse and preserve `<<EOF` heredoc syntax in slotmask data

4. **Resolution Compatibility**: Must validate that source's operated resolution matches display's compatible resolutions

5. **Template Expansion**: Must generate multiple source files from a single template JSON

6. **Section Merging**: Must correctly merge sections based on adapter-specific ownership rules

7. **Adapter Loading**: Dynamically load and instantiate adapters based on CLI flags or config

### Performance
- Should handle 50+ sources × 20+ displays = 1000+ combinations in under 10 seconds
- Parallel file reading where possible
- Caching of parsed JSON metadata

### Error Handling
- Graceful handling of malformed INI/JSON files
- Clear error messages with file paths
- Non-zero exit codes on failure for CI/CD integration
- Warning logs for skipped combinations

### Development Methodology

#### Test-Driven Development (TDD)

The project MUST follow Test-Driven Development practices:

1. **Red-Green-Refactor Cycle**: Write failing tests first, implement minimal code to pass, then refactor
2. **Test Coverage**: Aim for >90% code coverage for core logic and adapters
3. **Test Types**:
   - **Unit Tests**: Test individual functions and modules in isolation
   - **Integration Tests**: Test adapter implementations with real file parsing
   - **End-to-End Tests**: Test complete workflows (source + display → generated preset)
4. **Test Organization**: Mirror source code structure in test directory
5. **Mocking**: Use mocks/stubs for file system operations in unit tests
6. **Test Data**: Maintain fixture files (example INI, JSON files) in `tests/fixtures/`

#### Functional Programming Principles

The codebase MUST adhere to functional programming principles:

1. **Pure Functions**: Functions should be pure wherever possible (no side effects, deterministic output)
2. **Immutability**: Avoid mutating data structures; use immutable operations
3. **Function Composition**: Build complex operations by composing smaller functions
4. **Higher-Order Functions**: Use map, filter, reduce, and custom higher-order functions
5. **Type Safety**: Leverage TypeScript's type system to enforce correctness
6. **Avoid Classes for Logic**: Prefer functions and modules over classes (except for adapters which may use classes for interface implementation)
7. **Declarative Style**: Write code that expresses "what" rather than "how"

**Example Functional Patterns**:
```typescript
// Pure function for merging sections
const mergeSections = (sourceSections: Sections, displaySections: Sections, ownership: SectionOwnership): Sections => ({
  ...displaySections,
  ...filterByOwnership(sourceSections, ownership.source),
  ...mergeSharedSections(sourceSections, displaySections, ownership.merged)
});

// Composition for processing presets
const processPreset = pipe(
  parseSourceFile,
  validateSource,
  matchWithDisplays,
  mergePresets,
  writeOutputFiles
);
```

**Benefits of FP + TDD**:
- Pure functions are easier to test (no setup/teardown)
- Predictable behavior reduces bugs
- Composable functions enable flexible combinations
- Type safety catches errors at compile time
- Refactoring is safer with comprehensive tests

## Success Metrics

1. **Automation Efficiency**: Reduces time to generate full preset pack from hours to under 1 minute.

2. **Accuracy**: Generated presets work correctly on hardware (validated by testing pilot combinations).

3. **Usability**: User can generate their first custom preset pack within 10 minutes of reading documentation.

4. **Flexibility**: Supports at least 3 different HDMI prescale modes (DV1, PixelFX, manual) for a single console.

5. **Maintainability**: Adding a new display preset and regenerating all combinations takes under 2 minutes.

6. **Extensibility**: A developer can implement a new adapter for a different upscaler format within 4-8 hours.

7. **Adoption**: Tool is used to generate preset packs for multiple upscaler devices.

## Initial Implementation Focus

For the initial implementation, the tool should focus on creating **ONE complete example**:

**Source**: N64 with UltraHDMI (1080p 4x with DV1 auto-prescale)
**Display**: 240p CRT simulation (Sony PVM 14 with aperture grille mask)

This means:
1. Extract/create one source preset: `source/N64-UltraHDMI/1080p-4x-dv1/`
2. Extract/create one display preset: `display/crt/Sony PVM 14`
3. Implement basic combine logic
4. Generate: `generated/N64-UltraHDMI/1080p-4x-dv1/crt/N64-UltraHDMI 1080p-4x-dv1 - Sony PVM 14.ini`
5. Validate the generated preset works correctly

Once this foundation is working, expand to more sources and displays.

## Open Questions

1. **Section Conflicts**: If both source and display define the same property within a merged section (like `[shift-crop]`), which value wins? Should we have more granular precedence rules?

2. **Template Validation**: Should source templates be validated against a JSON schema to catch errors early?

3. **Display Variants**: Some displays might need variants for different resolutions (e.g., "Sony PVM 14 - 240p" vs "Sony PVM 14 - 480i"). Should these be separate files or one file with conditional sections?

4. **Operated Resolution Metadata**: Should operated resolution be extracted from INI files automatically, or always specified in JSON metadata?

5. **Backwards Compatibility**: Should the tool support reading existing repo presets and suggesting source/display splits?

6. **Custom Section Ownership**: Will users need to customize section ownership frequently, or can we ship sensible defaults?

7. **Dry-Run Output Format**: Should dry-run mode output a simple list, or show detailed merge previews?

8. **Config File Location**: Should `combiner.config.json` be in repo root, or in a `.morph-combiner/` directory?

9. **Adapter Auto-Detection**: Should the tool attempt to detect which adapter to use based on file extensions or directory structure, or always require explicit specification?

10. **Adapter Versioning**: How should adapter versions be handled? Should presets record which adapter version generated them?

11. **Cross-Adapter Source/Display Reuse**: Can source/display presets be shared across adapters, or should each adapter have its own source/display folders?
