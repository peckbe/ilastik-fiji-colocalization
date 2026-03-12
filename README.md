ilastik-FIJI Colocalization Pipeline

Automated, blinded fluorescence microscopy pipeline for quantifying glycocalyx
protein colocalization on vasculature in mouse brain tissue. Combines ilastik 3D
pixel classification with FIJI/ImageJ macro-based vessel morphometry.


OVERVIEW

This pipeline measures how much of a fluorescently-labeled protein of interest
colocalizes with tomato lectin (TL)-labeled vasculature in multi-channel Z-stack
.czi files. It was developed for APP/PS1 Alzheimer's Disease mouse models but is
generalizable to any colocalization study with a vessel marker channel.

The pipeline has three stages:

  1. ilastik: 3D pixel classification separates vessel signal from microglial
     contamination in the TL channel
  2. FIJI: Automated ImageJ macro handles thresholding, vessel masking, protein
     overlay measurement, and image export
  3. Analysis: Results are saved progressively as CSV with per-image and
     per-cohort summary statistics

All analysis runs blinded: file order is randomized, cohort names are hidden
during processing, and the unblinding key is printed only after all images are
complete.


REQUIREMENTS

Software
  - Python 3.11+ with Jupyter
  - FIJI (ImageJ) with Bio-Formats plugin
  - ilastik 1.4+ (GPU version recommended for 3D classification)

Python packages

  Option A: recreate the exact conda environment
    conda env create -f environment.yml
    conda activate zo1_env

  Option B: install just the dependencies
    pip install -r requirements.txt

File format

  Input images should be Zeiss .czi files with at least two channels:
    488 nm      Tomato lectin (vessel marker)
    514/561 nm  Protein of interest

  Optional channels:
    590-640 nm  Dextran (vascular permeability tracer)
    405 nm      DAPI (nuclei, not used in vessel analysis)

  Channels are auto-detected from CZI excitation wavelength metadata.


SETUP

Directory structure

  BASE_DIR/
    CD44/                           one subfolder per protein
      CD44_AD_PFC_1.czi
      CD44_AD_PFC_2.czi
      ...
    SDC1/
    ZO-1/
      imagej_results/               created automatically
        processed_images/
        ROIs/
        By_Mouse_Type/
        By_Brain_Region/
        By_Protein/
        vessel_analysis_results.csv
    vessel_classifier/              shared across all proteins
      vessel_pixel_classification_3D.ilp
      training_backups/
      *_TL_zstack.h5
      *_TL_MIP.tif

File naming convention

  (Protein)_(MouseType)_(BrainRegion)_(SampleNumber).czi

  Examples: ZO1_AD_PFC_3.czi, CD44_YOUNG_HIPP_1.czi

  The parser also accepts:
  (Protein)_TL_(Magnification)_(MouseType)_(BrainRegion)_(SampleNumber).czi


WORKFLOW

First-time setup

  1. Open the notebook and fill in the CONFIGURATION cell:
     - BASE_DIR: root folder containing protein subfolders
     - ILASTIK_EXE: full path to ilastik executable
     - FIJI_PATH: path to Fiji.app folder
     - PREDICT_DIR: folder with .czi files for the current protein
     - PROTEINS: list of protein subfolder names
     - MOUSE_TYPES, BRAIN_REGIONS: cohort categories
     - TRAINING_H5S: filenames of h5 files used to train the ilastik project

  2. Run the Dependencies and Configuration cells.

Stage 1: ilastik pipeline

  Run cells in order:

  Backup training files     Copies training h5s to training_backups/ before
                            any re-exports
  Step 1: Export TL z-stacks  Extracts TL channel from each .czi, pads to
                              uniform Z depth, saves as HDF5 + MIP
  (Train ilastik)           Open vessel_pixel_classification_3D.ilp in ilastik
                            GUI, train on exported h5 files
  Step 3: Headless prediction  Runs ilastik batch prediction on all exported
                               h5 files
  Step 4: Generate weighted TL variants  Computes 3 vessel mask variants per
                                         image
  QC comparison             Side-by-side visualization of all 4 TL options

  Step 4 generates three ilastik-processed versions of each TL channel:
    Binary mask: TL MIP x binary vessel segmentation
    Probability weighted: TL MIP x vessel class confidence
    Dominant-class: Per-slice weighted projection (zeroes voxels where microglia
      probability exceeds vessel probability, weights remaining by vessel
      confidence, then max-projects)

Stage 2: FIJI analysis

  Run the RUN ANALYSIS cell. For each image, the FIJI macro:

   1. Opens .czi with Bio-Formats
   2. Creates max-intensity Z-projection
   3. Subtracts background (rolling ball)
   4. (If USE_ROI = True) Splits channels for ROI drawing on TL, user draws
      freehand ROIs (press T to add each)
   5. (If ilastik available) Shows 4 TL options for user selection
   6. Thresholds TL to create vessel binary mask
   7. Multiplies protein x vessel mask
   8. Measures: vessel area fraction, overlay MFI, protein coverage on vessels
   9. (If dextran present) Thresholds dextran, subtracts vessel mask, measures
      parenchymal leakage
  10. Saves composite overlay images (grayscale TL + colored protein + vessel
      outline)

Stage 3: Results

  Results are saved to OUTPUT_DIR/vessel_analysis_results.csv with columns:
    vessel_area_percent            fraction of image covered by vessels
    mean_intensity                 mean fluorescence of protein on vessels
    protein_coverage_percent       fraction of image with protein signal on vessels
    percent_coverage_on_vessels    protein coverage normalized to vessel area
    dextran_coverage_percent       parenchymal dextran (outside vessels)
    dextran_total_percent          total dextran coverage
    + background statistics, ROI dimensions, metadata

  The pipeline also generates summary bar plots by mouse type and brain region.


CONFIGURATION REFERENCE

Processing modes
  DEBUG_MODE          False     Pauses at each step, processes 1 image per
                                cohort, fixed random seed
  USE_ROI             False     Enable freehand ROI selection (multiple ROIs)
  AUTO_MODE           False     Skip manual thresholding, use Otsu automatically
  AUTO_SAVE_IMAGES    False     Save overlay images without prompting
  BG_CORRECTION       True      Measure off-vessel background and subtract
                                before coverage measurement
  ILASTIK_TL_MODE     'manual'  How ilastik TL variants are used:
                                'manual' / 'auto' / 'off'

Vessel mask cleanup
  TL_GAUSSIAN_BLUR       0      Sigma for pre-threshold blur (0 = disabled)
  TL_CLOSE_ITERATIONS    0      Morphological close iterations (0 = disabled)
  TL_FILL_HOLES          True   Fill internal holes in vessel cross-sections
  TL_REMOVE_SMALL        5      Remove particles smaller than N px (0 = disabled)

Overlay image colors (RGB)
  OVERLAY_PROTEIN_COLOR    [255, 0, 0]      Red protein signal
  OVERLAY_VESSEL_COLOR     [255, 255, 0]    Yellow vessel outline
  OVERLAY_DEXTRAN_COLOR    [255, 0, 255]    Magenta dextran signal


RE-ANALYSIS (OPTION 4)

When running analysis on images that already have results, the pipeline offers
4 options:

  1. Process ALL: overwrite existing results
  2. Skip existing: only process new images
  3. Cancel
  4. Re-run with saved masks: reuses saved vessel masks and dextran thresholds
     for reproducibility

Option 4 is useful for iterating on background correction or threshold settings
without re-doing manual vessel thresholding.


BLINDING

Blinding is always active:
  - File processing order is randomized
  - Cohort names are hidden during FIJI processing
  - FIJI Log window is suppressed
  - Each image is identified only as Image_001, Image_002, etc.
  - The unblinding key prints after all images are complete
  - In DEBUG_MODE, the random seed is fixed (42) for reproducible ordering


UTILITIES

The Sweep cell at the bottom copies all overlay images into a single
overlay_review/ folder for quick visual review.


DEBUG TOOLS

ilastik_debug_tools.ipynb is a standalone notebook for diagnosing pipeline
issues. It includes:
  - Channel/laser assignment table with image dimensions, excitation
    wavelengths, and detector settings per file
  - Exported h5 file verification (Z depth, file size)
  - Probability file check (shape, missing targets)
  - Colored mask export for visual inspection
  - Threshold tuning

Set BASE_DIR and PREDICT_DIR at the top of the notebook. No other cells need
to be run first.


TROUBLESHOOTING

ilastik "Project could not be loaded": This happens when RE_EXPORT = True
overwrites the training h5 files that the .ilp project references internally.
Restore them from training_backups/ and re-open the project.

Z-stack shape mismatch in ilastik prediction: Step 1 pads all z-stacks to a
uniform depth (the max Z across all images). If you add new images with more
Z-slices than the original batch, the padded depth changes and old probability
volumes won't match. Re-run Step 1 + Step 3 for the full dataset.

Wrong channel in exported h5 / QC shows protein instead of TL: The h5 files may
have been exported before auto-detection was added or with a stale channel
config. Use the debug tools notebook to verify channel assignments (C1: 594nm,
C2: 514nm, C3: 488nm, etc.), set RE_EXPORT = True, re-run Step 1, then re-run
Steps 3-4. Set RE_EXPORT = False after.


ADAPTING FOR OTHER COLOCALIZATION STUDIES

The core logic (threshold one channel to create a mask, multiply by another
channel, measure overlap) can be adapted for other experimental designs.

Different marker channels

  Edit get_channel_config() in the Helpers cell to change which wavelengths
  map to which roles. For example, to swap so 488=protein and 561=vessel marker:

    if 480 <= ex <= 495:
        config['protein'] = ch_num
    elif 510 <= ex <= 565:
        config['TL'] = ch_num

  Or set AUTO_DETECT_CHANNELS = False and fill in CHANNEL_CONFIG manually.

Different tissue types or species

  Update MOUSE_TYPES, BRAIN_REGIONS, and PROTEINS to match your experimental
  groups. Adjust the filename parser in _parse_standardized_filename() if your
  naming convention differs.

Using without ilastik

  Set ILASTIK_TL_MODE = 'off' and skip Steps 1, 3, 4, and QC. The pipeline
  will threshold the raw TL MIP directly in FIJI.

Arbitrary two-channel colocalization

  Assign your "mask" channel as TL and your "signal" channel as protein in the
  channel config. The pipeline will threshold the mask, multiply by the signal,
  and report coverage. Variable names will still reference "vessel" but the math
  is the same.

Adding new measurements

  Add a variable to the macro init block, compute it at the appropriate point,
  and append it to the results string as key:value. The Python side automatically
  parses any key-value pairs from the results file into the CSV.

Different image formats

  Bio-Formats supports .lif, .nd2, .ims, .lsm, and .tif in addition to .czi.
  Change the glob pattern from '*.czi' to your extension. If your format doesn't
  expose excitation wavelengths through OME metadata, use manual channel
  configuration.


LICENSE

MIT License. Created by Benjamin Peck (2026), Northeastern University.
