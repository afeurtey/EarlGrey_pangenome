# Development notes for the pangenome pipeline of Earlgrey

### February 2026
We decided to get rid of the EarlGrey set up steps, such a the DFAM configuration etc, in the pangenome pipeline. Before using the pangenome pipeline, a user would have to run the normal installation and set up script. So there is no need to repeat the steps here. The only thing left to do in the pangenome pipeline is to carry out the checks. 

The pipeline expects a config.yaml file called `config/config.yaml`. This file contains the necessary information to run the pipeline such as the species and their genome files. It can also contain some additional options, but for some defaults will be applied (see the validate_parameters function for more details).

I am testing with snakemake version 9.9.0. 

`snakemake --cores 1 --dag ` will run the parameter checks and create the dag. 

Now copy all the text which is not part of the check messages into a `temp.txt` file and use the following command to create the DAG plot. 

`cat temp.txt | dot -Tsvg > dag.svg` 

If you want a diagram that represents a summary of the DAG, independently of the different samples, you can use `--rulegraph` instead. 

`snakemake --cores 1` to run the pipeline.

### March 2026 - Toby Testing and Development Notes

I will first build a mamba env with earlgrey 7.0.3 & snakemake

```bash
mamba create -c conda-forge -c bioconda -n earlgrey-pan-dev earlgrey=7.0.3 snakemake
```

Configure earl grey properly by running `earlGrey`

```bash
earlGrey 
# edited /data/toby/EarlGrey_pangenome$ bash configure_dfam39.sh to only be part 0
bash configure_dfam39.sh
```

Now, make a test directory with two test genomes:

```bash
mkdir -p /data/toby/EarlGrey_pangenome/test
cd /data/toby/EarlGrey_pangenome/test

# get some test genomes
cp /data/toby/tools/test.fasta /data/toby/EarlGrey_pangenome/test/genome1.fasta
cp /data/toby/tools/test.fasta /data/toby/EarlGrey_pangenome/test/genome2.fasta
```

Edit the config file to point to these two genomes:

```yaml
genome: 
  genome1: "/data/toby/EarlGrey_pangenome/test/genome1.fasta"
  genome2: "/data/toby/EarlGrey_pangenome/test/genome2.fasta"
species: ["genome1", "genome2"]
output_dir: "/data/toby/EarlGrey_pangenome/test/results"
threads: 8
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3
consensus_library : "results/combined_all_species.fa"
script_dir: "/data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts"
run_heliano: False
```

I saved the original as: `/data/toby/EarlGrey_pangenome/pangenome/config/config.yaml.bak`

Try and run the pipeline:

```bash
cd /data/toby/EarlGrey_pangenome/pangenome
snakemake --cores 1 --dag 2>&1 | sed -n '/^digraph/,$p' > /data/toby/EarlGrey_pangenome/test/temp.txt
cat /data/toby/EarlGrey_pangenome/test/temp.txt | dot -Tsvg > /data/toby/EarlGrey_pangenome/test/dag.svg
```

This should create a DAG plot in the test directory.

Now, run the pipeline with a dry run to check for any errors:

```bash
snakemake --cores 1 --dry-run
```

Now, run the pipeline for real:

```bash
snakemake --cores 1
```

**Issue Fixed (March 3, 2026):** RepeatMasker was running even when `repeatmasker_species` and `custom_library` were not specified, causing errors because it was called with empty `-species` parameter.

**Solution:** Modified `rules/lib_construct.smk` to:
1. Added `get_masked_genome_input()` function that returns the appropriate input:
   - Masked genome if `repeatmasker_species` or `custom_library` is specified
   - Prep genome directly if neither is specified
2. Updated `build_db` rule to use this conditional input function
3. Added `ruleorder` directive to prioritize the correct repeatmasker rule when needed

Now the workflow correctly skips the initial RepeatMasker step when no masking parameters are provided, going directly from `prep_genome` to `build_db`.

Now, try another dry run to confirm the issue is resolved:

```bash
snakemake --cores 1 --dry-run
```

Now, run the pipeline again to ensure it executes without errors:

```bash
snakemake --cores 1
```

This failed because I think the pipeline is using the wrong earl grey installation. it should be using the version in the active conda env, not in the EarlGrey_pangenome directory.

**Issue Fixed (March 3, 2026):** The pipeline was using hardcoded script paths pointing to the local repository instead of the conda environment's EarlGrey installation.

**Solution:** Modified `rules/lib_construct.smk` to:
1. Added `script_dir` parameter to `prep_genome` rule and changed the headSwap.sh call from `../scripts/headSwap.sh` to `{params.script_dir}/headSwap.sh`
2. Added `script_dir` parameter to `testrainer` rule and changed the TEstrainer call from `scripts/TEstrainer/TEstrainer_for_earlGrey.sh` to `{params.script_dir}/TEstrainer/TEstrainer_for_earlGrey.sh`
3. Also updated `/data/toby/EarlGrey_pangenome/scripts/headSwap.sh` to dynamically determine its own directory instead of hardcoding it (though this won't be used when the conda env version is used)

Now the pipeline correctly uses the scripts from the conda environment specified in `config["script_dir"]`.

Clean previous outputs and try again:

```bash
snakemake --cores 1 --delete-all-output
snakemake --cores 1
```

**Issue Fixed (March 3, 2026):** BuildDatabase was creating files in the wrong directory and the output specification was incorrect.

**Problem:** BuildDatabase creates multiple files (`.nhr`, `.nin`, `.nsq`, `.nnd`, `.nni`, `.nog`, `.translation`, `.njs`) in the current working directory, not a single `.db` file.

**Solution:** Modified `rules/lib_construct.smk` to:
1. Updated `build_db` rule to output the actual files BuildDatabase creates (`.nhr`, `.nin`, `.nsq`) instead of a non-existent `.db` file
2. Added `cd {params.outdir}` to change to the output directory before running BuildDatabase
3. Updated `repeatmodeler` rule to:
   - Accept the database files as input instead of a single `.db` file
   - Change to the database directory before running RepeatModeler
   - Pass just the database name (not full path) to RepeatModeler

Now the workflow correctly creates database files in the expected output directories and RepeatModeler can find them.

The pipeline is now working up to the clustering point. So far, I have added `snakemake`.

At the moment, there are too many conflicts to add `mmseqs2` to the mamba env, but this would eventually be preferred. Thus, I will continue with clustering with cd-hit for now.

---

## March 3, 2026 - Major Pipeline Refinement Session

### Summary of Changes

Today I worked through a comprehensive debugging and refinement session to make the pangenome pipeline fully functional. Below are all the major changes made:

#### 1. Clustering Implementation
- **Added genome identifiers to sequences**: Modified `clustering.smk` to prefix each sequence header with the genome name (e.g., `>genome1_family-13`) to track sequence origins
- **Added library source prefixes**: RepeatMasker libraries get `REPMASKER_{species}_` prefix (e.g., `>REPMASKER_fungi_TE_name#DNA/TcMar-Tc1`), custom libraries get `CUSTOM_` prefix. This makes it easy to identify both the source and the specific RepeatMasker species/clade used.
- **Fixed library extraction**: RepeatMasker species/clade library is now extracted once to `{OUTDIR}/{REPSPEC}.RepeatMasker.lib` instead of per-genome
- **Conditional library inclusion**: Clustering now conditionally includes RepeatMasker species libraries and/or custom libraries when specified in config
- **Single cd-hit-est run**: Changed from two-pass clustering to single pass with parameters: `-d 0 -aS 0.8 -c 0.8 -G 0 -g 1 -b 500 -r 1`
- **Fixed output location**: Clustered library outputs to `{OUTDIR}/combinedLibraries/combined_all_species.clstrd.fa`
- **Added cleanup**: Automatically removes temporary concatenated file and `.clstr` intermediate files

#### 2. RepeatMasker Annotation Fixes
- **Fixed library path**: Updated `annotate_simple.smk` to use absolute path with `realpath` for library input
- **Fixed working directory**: RepeatMasker now runs with `cd` into output directory and `-dir` parameter to ensure outputs go to correct location
- **Library reference**: Changed to use clustered pangenome library instead of per-genome libraries

#### 3. Switch to Full Annotation Pipeline
- **Activated annotate.smk**: Changed `Snakefile` to include `rules/annotate.smk` instead of `annotate_simple.smk`
- **Updated library references**: Modified `repeatmasker_annotation`, `calculate_divergence`, and `sweep_up_files` rules to use pangenome clustered library
- **Commented out obsolete rules**: Disabled `create_repeatmasker_library` and `combine_libraries` rules that are redundant in pangenome approach
- **Fixed HELIANO variable**: Added proper handling for both `heliano` and `run_heliano` config keys with boolean/string conversion

#### 4. Summary Charts and Divergence
- **Fixed autoPie.sh inputs**: Changed from `.bed` file to `.summary` file as required by the script
- **Added `.summary` output**: Updated `merge_repeats` rule to declare `.filteredRepeats.summary` as an output
- **Added divergence to workflow**: Updated `rule all` to request `{species}_divergence_summary_table.tsv` outputs
- **Fixed divergence file handling**: Changed from `mv` to `cp` for withDivergence.gff to keep output file in expected location while updating main GFF
- **Added divergence dependency**: Made `sweep_up_files` depend on divergence summary to ensure proper execution order

#### 5. Output Organization
- **Removed duplicate libraries**: Eliminated per-genome copies of pangenome library in summary directories - now only exists in `combinedLibraries/`
- **Single pangenome library**: Final clustered library at `{OUTDIR}/combinedLibraries/combined_all_species.clstrd.fa` is the single source of truth

#### 6. Bug Fixes and Refinements
- **Fixed variable naming**: Changed `{output.count}` to `{output.highLevelCount}` in autoPie.sh call
- **Removed duplicate closing**: Fixed syntax error from duplicate `"""` in sweep_up_filesrule
- **Disabled make_directories**: Commented out `make_directories()` call in onstart that was creating incorrectly-named directory with literal list string
- **Fixed REPSPEC usage**: Changed from treating `{repspec}` as wildcard to using `REPSPEC` as constant throughout pipeline
- **Added missing HELI variable**: Added `HELI = config.get("run_heliano", False)` definition to Snakefile

#### 7. Workflow Execution
- **Updated rule all**: Now requests final summary files including divergence outputs, not just intermediate RepeatMasker files
- **Proper dependencies**: Ensured correct execution order through input/output dependencies

### Pipeline Architecture

The final pangenome pipeline flow:
1. **Prep genomes** → Create cleaned, indexed genome files
2. **Build databases** → Create BLAST databases for RepeatModeler
3. **RepeatModeler** → De novo repeat discovery with fallback for classification failures
4. **TEstrainer** → Refine consensus sequences
5. **Extract libraries** → Extract RepeatMasker species library if specified (once, not per-genome)
6. **Clustering** → Combine all strained sequences with genome prefixes, plus RepeatMasker/custom libraries with source prefixes, cluster with cd-hit-est
7. **Annotation** → Annotate each genome with pangenome clustered library
8. **Merge repeats** → Merge overlapping repeat annotations
9. **Summary charts** → Generate pie charts and counts
10. **Divergence calculation** → Calculate divergence and generate landscape plots
11. **Sweep up** → Copy final results to summary directories

### Key Files Modified
- `Snakefile` - Changed to use annotate.smk, updated rule all, disabled make_directories
- `rules/lib_construct.smk` - Added extract_repeatmasker_library rule, fixed script paths
- `rules/clustering.smk` - Complete rewrite with genome prefixes, library prefixes, single cd-hit run
- `rules/annotate.smk` - Updated for pangenome library, fixed all input/output paths, added .summary output
- `config/config.yaml` - Configured for test genomes with proper script_dir path

### Testing
Successfully ran full pipeline on two test genomes (genome1.fasta and genome2.fasta, both copies of test.fasta) with all stages completing:
- Library construction: 40 sequences per genome after TEstrainer
- Clustering: Combined 80 sequences → 35 unique sequences in pangenome library
- Annotation: Both genomes annotated with pangenome library
- Divergence: Landscape plots and summary tables generated
- Final outputs: All summary files, charts, and annotations created successfully

## March 4, 2026 - Testing optional units

I need to test each optional unit to ensure correct functioning. A full list of optional units to test:
- HELIANO **SUCCESS**
- Initial RepeatMasker with species term **SUCCESS**
- Initial RepeatMasker with custom library **SUCCESS**
- Changing BEAT iterations
- Changing flank size
- Changing clustering parameters (e.g. cd-hit identity threshold), or avoiding this step entirely
- Removing TE annotations < 100bp
- Creating a softmasked genome for each input genome after annotation
- Changing max number of sequences used to generate a consensus sequence
- Changing min number of sequences used to generate a consensus sequence
- Printing help

### Unit test 1: HELIANO
To test the HELIANO unit, I will set `run_heliano: True` in the config file and run the pipeline. I will check that the HELIANO-specific outputs are generated correctly and that the workflow executes without errors.

```yaml
genome: 
  genome1: "/data/toby/EarlGrey_pangenome/test/genome1.fasta"
  genome2: "/data/toby/EarlGrey_pangenome/test/genome2.fasta"
species: ["genome1", "genome2"]
output_dir: "/data/toby/EarlGrey_pangenome/test/unit_test_1_HELIANO"
threads: 8
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3
script_dir: "/data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts"
run_heliano: True
```

I will run the pipeline and check for the presence of HELIANO-specific outputs, such as any files or logs that indicate HELIANO was executed. I will also verify that the main workflow completes successfully without errors.

```bash
snakemake --cores 8
```

This ran successfully, and I can confirm that the HELIANO-specific outputs were generated as expected. The workflow executed without any errors, indicating that the HELIANO unit is functioning correctly when enabled. I did fix `mergeRepeats.R` which was not parsing the attributes column for the HELIANO outputs correctly, and have pushed this to the earl grey repo.

### Unit test 2: Initial RepeatMasker with species term
To test the initial RepeatMasker step with a species term, I will set `repeatmasker_species: "lepidoptera"` in the config file and run the pipeline. I will check that the RepeatMasker step runs correctly and that the specified species library is used for masking.

```yaml
genome: 
  genome1: "/data/toby/EarlGrey_pangenome/test/genome1.fasta"
  genome2: "/data/toby/EarlGrey_pangenome/test/genome2.fasta"
species: ["genome1", "genome2"]
output_dir: "/data/toby/EarlGrey_pangenome/test/unit_test_2_REPEATMASKER"
threads: 8
repeatmasker_species: "lepidoptera"  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3
script_dir: "/data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts"
run_heliano: True
```

I will run the pipeline and check the logs to confirm that RepeatMasker is using the `lepidoptera` library for masking. I will also verify that the workflow completes successfully without errors.

```bash
# make the DAG to check the RepeatMasker step is included
snakemake --cores 1 --dag 2>&1 | sed -n '/^digraph/,$p' > /data/toby/EarlGrey_pangenome/test/unit_test_2_REPEATMASKER/temp.txt
cat /data/toby/EarlGrey_pangenome/test/unit_test_2_REPEATMASKER/temp.txt | dot -Tsvg > /data/toby/EarlGrey_pangenome/test/unit_test_2_REPEATMASKER/dag.svg

# run the pipeline
snakemake --cores 8
```

I had to fix `lib_construct.smk` as it was not finding the correct masked genome file (simply pointing to prep.masked rather than original input.masked!).

Okay, another issue here. When using RepeatMasker there aren't enough sequences left for RepeatModeler to run successfully in my test fasta. I will need to run on bigger test sets. I will use Z. tritici and fungi RepeatMasker library for this test.

```bash
cp /legserv/NGS_data/Zymoseptoria/Zt_
Reference_genomes/19Pangenome_genomes/IPO323/Zymoseptoria_tritici.MG2.dna.toplevel.mt+.fa /data/toby/EarlGrey_pangenome/test/IPO323.fa

cp /legserv/NGS_data/Zymoseptoria/Zt_Reference_genomes/19Pangenome_genomes/1A5/ST99CH_1A5.fa /data/toby/EarlGrey_pangenome/test/1A5.fa
```

Here is the updated yaml for this test:

```yaml
genome: 
  IPO323: "/data/toby/EarlGrey_pangenome/test/IPO323.fa"
  1A5: "/data/toby/EarlGrey_pangenome/test/1A5.fa"
species: ["IPO323", "1A5"]
output_dir: "/data/toby/EarlGrey_pangenome/test/unit_test_2_REPEATMASKER"
threads: 16
repeatmasker_species: "fungi"  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3
script_dir: "/data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts"
run_heliano: True
```

```bash
snakemake --cores 16
```

This completed successfully, and the libraries were correctly combined and clustered. The initial RepeatMasker step ran with the `fungi` library as expected, and the workflow executed without errors. I can confirm that the initial RepeatMasker with species term is functioning correctly when enabled.

### Unit test 3: Initial RepeatMasker with custom library
To test the initial RepeatMasker step with a custom library, I will create a small custom library file with a few repeat sequences and set `custom_library: "/path/to/custom_library.fa"` in the config file. I will run the pipeline and check that RepeatMasker uses the custom library for masking, and that the workflow completes successfully without errors.

```yaml
genome: 
  IPO323: "/data/toby/EarlGrey_pangenome/test/IPO323.fa"
  1A5: "/data/toby/EarlGrey_pangenome/test/1A5.fa"
species: ["IPO323", "1A5"]
output_dir: "/data/toby/EarlGrey_pangenome/test/unit_test_3_REPEATMASKERCUSTOM"
threads: 16
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: "/data/toby/EarlGrey_pangenome/test/test_custom_library.fa"        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3
script_dir: "/data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts"
run_heliano: True
```

```bash
snakemake --cores 16
```

This worked. I had to fix some of the onstart function messages to update with the new config parameters, but the pipeline ran successfully with the custom library. RepeatMasker used the specified custom library for masking, and the workflow completed without errors. I can confirm that the initial RepeatMasker with custom library is functioning correctly when enabled.

I have also added a function to either not cluster sequences at all, or give users the ability to change the clustering parameters.

### Unit test 4: Changing all numeric parameters
To test changing all numeric parameters, I will modify the config file to set different values for `iterations`, `flank`, `max_consensus_seqs`, and `min_consensus_seqs`. I will run the pipeline and check that these parameters are correctly applied in the relevant steps (e.g., TEstrainer iterations, flank size in library construction, and consensus sequence generation). I will also verify that the workflow completes successfully without errors.

```yaml
genome: 
  genome_1: "/data/toby/EarlGrey_pangenome/test/genome1.fasta"
  genome_2: "/data/toby/EarlGrey_pangenome/test/genome2.fasta"
species: ["genome_1", "genome_2"]
output_dir: "/data/toby/EarlGrey_pangenome/test/unit_test_4_NUMERICPARAMS"
threads: 16
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 15
flank: 2000
max_consensus_seqs: 30
min_consensus_seqs: 5

# Clustering options (for combining TE libraries from multiple genomes)
skip_clustering: False  # Set to True to skip clustering (just concatenate all libraries)
clustering_identity: 0.8  # cd-hit sequence identity threshold (0.0-1.0, default 0.8)
clustering_coverage: 0.8  # cd-hit alignment coverage for shorter sequence (0.0-1.0, default 0.8)

script_dir: "/data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts"
run_heliano: True
```

```bash
snakemake --cores 16
```

## March 4, 2026 - Configuration Enhancement and Validation

### Summary of All Changes Made Today

Today I worked with Claude to enhance the pangenome pipeline configuration system and add several new optional features. Below is a comprehensive summary of all changes:

#### 1. Fixed Configuration Validation and Reporting
**File:** `scripts/on_start_functions.py`

**Problem:** The validation function was using old parameter names from the original EarlGrey pipeline (e.g., `num`, `no_seq`, `RepSpec`, `startCust`) that didn't match the pangenome-specific config structure.

**Changes:**
- Updated parameter defaults dictionary to use correct pangenome parameter names:
  - `num` → `iterations`
  - `no_seq` → `max_consensus_seqs`
  - `min_seq` → `min_consensus_seqs`
  - `Flank` → `flank`
  - Added: `skip_clustering`, `clustering_identity`, `clustering_coverage`, `softmask`, `margin`, `repeatmasker_species`, `custom_library`
  
- Fixed validation logic to handle both empty strings and None/missing values properly
- Updated "Pipeline behaviour" reporting to correctly detect and report:
  - RepeatMasker species vs custom library usage (distinguishes between them)
  - Clustering parameters or skip status
  - Softmask status
  - Margin (short TE removal) status
  - HELIANO status

**Impact:** Config validation now accurately reflects what the pangenome pipeline will do with clear, informative messages.

#### 2. Implemented Clustering Control
**Files:** `rules/clustering.smk`, `config/config.yaml`

**Problem:** Users had no control over clustering behavior - it always ran cd-hit with hardcoded parameters.

**Changes:**
- Added three new config parameters:
  - `skip_clustering: False` - Set to True to concatenate libraries without clustering
  - `clustering_identity: 0.8` - cd-hit sequence identity threshold (0.0-1.0)
  - `clustering_coverage: 0.8` - cd-hit alignment coverage threshold (0.0-1.0)
  
- Modified `cluster_all_species` rule to:
  - Read these parameters from config
  - Conditionally run cd-hit or just copy the combined file if skipping
  - Use user-specified identity and coverage thresholds in cd-hit command
  - Clean up .clstr files in both paths

**Impact:** Users can now:
- Skip clustering entirely for maximum subfamily resolution
- Adjust stringency (e.g., 0.95/0.90 for stricter, 0.70/0.70 for more permissive)
- See clustering parameters reported during validation

#### 3. Implemented Softmasking and Short TE Removal
**Files:** `rules/annotate.smk`, `rules/lib_construct.smk`, `Snakefile`, `config/config.yaml`

**Problem:** 
- Softmasked genome generation wasn't integrated into the pangenome pipeline
- Short TE removal (margin) option wasn't exposed
- The `prep_genome` rule wasn't creating the backup file needed for softmasking

**Changes:**
- Added two new config parameters:
  - `softmask: False` - Set to True to generate softmasked genomes using bedtools
  - `margin: False` - Set to True to remove TEs <100bp from annotations
  
- Fixed `prep_genome` rule in `lib_construct.smk`:
  - Added `backup` output: `{species}.bak.gz`
  - Creates backup in output directory (not input directory)
  - Properly cleans up temporary files
  
- Updated `annotate.smk`:
  - Added boolean-to-string conversion for `SOFTMASK` and `MARGIN` variables (True/False → "yes"/"no")
  - Matches pattern used for HELIANO to ensure shell commands work correctly
  
- Modified `Snakefile`:
  - Added `SOFTMASK` and `MARGIN` global variables
  - Made softmasked genomes conditionally requested in `rule all` based on config

**Impact:** 
- Softmasked genomes automatically generated when requested
- Short TEs can be filtered from final annotations
- Both features properly validated and reported

#### 4. Fixed RepeatMasker Output Filename Issue
**File:** `rules/lib_construct.smk`

**Problem:** RepeatMasker appends `.masked` to the input filename. With input `genome1.prep`, output is `genome1.prep.masked`, not `genome1.masked`.

**Changes:**
- Updated both `repeatmasker` and `repeatmasker_custom` rules
- Changed output specification from `{species}.masked` to `{species}.prep.masked`
- Updated `get_masked_genome_input()` function to return correct path

**Impact:** Pipeline correctly finds RepeatMasker output files.

#### 5. Fixed RepeatMasker Working Directory Issue
**File:** `rules/lib_construct.smk`

**Problem:** RepeatMasker was creating output files in the current working directory (pangenome/) instead of species-specific directories. Caused RM_* temp directories to appear in wrong location.

**Changes:**
- Modified both `repeatmasker` and `repeatmasker_custom` rules to:
  - Change `params.outdir` to use wildcard-based path: `"{outdir}/{species}_EarlGrey/{species}_RepeatMasker"`
  - Add `cd {params.outdir}` before RepeatMasker execution
  - Use `realpath` for input genome paths
  - Explicitly set `-dir {params.outdir}` parameter

**Impact:** All RepeatMasker outputs now correctly created in species-specific directories, matching the full annotation pipeline behavior.

#### 6. Code Cleanup
**Files:** `rules/annotate.smk`, `config/config.yaml`

**Changes:**
- Removed obsolete `CONSENSUS_LIB` variable that was never used (only referenced in commented-out rules)
- Suggested removing commented-out dead code:
  - Lines 18-102 in `annotate.smk`: old `rule all`, `prep_genome`, `create_repeatmasker_library`, `combine_libraries`
  - Lines 287-306: conditional rules for old single-genome approach
- Restored preferred config formatting (double quotes, capitalized booleans, helpful comments)

**Impact:** Cleaner, more maintainable codebase.

### Files Modified Summary

1. **`pangenome/scripts/on_start_functions.py`**
   - Fixed all parameter names and validation logic
   - Updated all reporting messages for accuracy

2. **`pangenome/rules/clustering.smk`**
   - Added skip/parameter control for clustering

3. **`pangenome/rules/lib_construct.smk`**
   - Fixed RepeatMasker output paths and working directories
   - Added backup file creation to `prep_genome` rule

4. **`pangenome/rules/annotate.smk`**
   - Added boolean-to-string conversion for SOFTMASK and MARGIN
   - Removed unused CONSENSUS_LIB variable

5. **`pangenome/Snakefile`**
   - Added SOFTMASK and MARGIN global variables
   - Made softmasked genomes conditionally requested in rule all

6. **`pangenome/config/config.yaml`**
   - Added clustering control parameters with comments
   - Added output options (softmask, margin) with comments
   - Restored preferred formatting

### Tests Completed Today
1. ✅ Configuration validation messages display correctly
2. ✅ Clustering parameters load and display properly
3. ✅ Softmask boolean conversion works (True → "yes", False → "no")

### Tests Still Needed

Based on the original testing checklist, here are the tests that still need to be performed:

#### 1. **Complete Unit Test 4: Numeric Parameters**
**Status:** Started but needs completion
**Config:**
```yaml
iterations: 15        # Changed from default 10
flank: 2000          # Changed from default 1000
max_consensus_seqs: 30  # Changed from default 20
min_consensus_seqs: 5   # Changed from default 3
```
**Verification needed:**
- Check TEstrainer runs 15 iterations (look at log files)
- Verify flank size of 2000bp used in BEAT process
- Confirm consensus generation uses 30 max / 5 min sequences

#### 2. **Unit Test 5: Skip Clustering**
**Config:**
```yaml
skip_clustering: True
```
**Verification needed:**
- Check that `combined_all_species.clstrd.fa` is created
- Verify it contains ALL sequences from all genomes without reduction
- Compare sequence count: input (sum of all strains) should equal output
- Confirm no `.clstr` file is generated
- Annotation should still work with unclustered library

#### 3. **Unit Test 6: Custom Clustering Parameters**
**Config Option A - More Stringent:**
```yaml
skip_clustering: False
clustering_identity: 0.95
clustering_coverage: 0.90
```
**Config Option B - More Permissive:**
```yaml
skip_clustering: False
clustering_identity: 0.70
clustering_coverage: 0.70
```
**Verification needed:**
- Compare output library sizes between stringent/permissive/default (0.8)
- More stringent should result in MORE sequences (less clustering)
- More permissive should result in FEWER sequences (more clustering)
- Check cd-hit log files confirm correct parameters used

#### 4. **Unit Test 7: Softmasked Genome Generation**
**Config:**
```yaml
softmask: True
```
**Verification needed:**
- Check `.softmasked.fasta` files created in `{species}_summaryFiles/`
- Verify repeat regions are lowercase, non-repeat regions uppercase
- File should exist for each input genome
- Confirm workflow requests these files (check DAG)

#### 5. **Unit Test 8: Margin (Remove Short TEs)**
**Config:**
```yaml
margin: True
```
**Verification needed:**
- Compare `.filteredRepeats.bed` with and without margin
- All annotations <100bp should be absent when margin: True
- Check statistics - total repeat count should be lower
- Annotation coverage (bp) should be reduced

#### 6. **Unit Test 9: Combined Options Test**
**Config:**
```yaml
skip_clustering: True
softmask: True
margin: True
run_heliano: True
repeatmasker_species: "fungi"
```
**Verification needed:**
- All features work together without conflicts
- Check final outputs have all expected files
- Validation messages report all options correctly

### Recommended Testing Order

1. **Unit Test 4** (numeric parameters) - validates basic parameter passing
2. **Unit Test 7** (softmask) - new feature, critical to verify
3. **Unit Test 8** (margin) - new feature, quick to verify
4. **Unit Test 5** (skip clustering) - validates major new feature
5. **Unit Test 6** (clustering parameters) - builds on test 5
6. **Unit Test 9** (combined) - integration test

### Quick Test Commands

For each test, follow this pattern:
```bash
# Update config.yaml with test parameters
snakemake --cores 1 --dry-run  # Check validation messages
snakemake --cores 16            # Run pipeline
# Verify outputs as described above
```

### Unit Test 7: Softmasked Genome Generation
To test the softmasked genome generation, I will set `softmask: True` in the config file and run the pipeline. I will check that the `.softmasked.fasta` files are created in the `{species}_summaryFiles/` directories for each input genome. I will verify thatthe repeat regions are correctly softmasked (lowercase) and non-repeat regions are uppercase. I will also confirm that the workflow requests these files by checking the DAG.

```yaml
genome:
  genome_1: /data/toby/EarlGrey_pangenome/test/genome1.fasta
  genome_2: /data/toby/EarlGrey_pangenome/test/genome2.fasta
species: [genome_1, genome_2]
output_dir: /data/toby/EarlGrey_pangenome/test/unit_test_7_SOFTMASK
threads: 8
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3

# Clustering options (for combining TE libraries from multiple genomes)
skip_clustering: False  # Set to True to skip clustering (just concatenate all libraries)
clustering_identity: 0.8  # cd-hit sequence identity threshold (0.0-1.0, default 0.8)
clustering_coverage: 0.8  # cd-hit alignment coverage for shorter sequence (0.0-1.0, default 0.8)

# Output options
softmask: True  # Set to True to generate softmasked genome for each input
margin: False  # Set to True to remove short TE sequences (<100bp)

script_dir: /data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts
run_heliano: True
```

```bash
# first make the DAG to confirm the softmasked genome files are requested
mkdir -p /data/toby/EarlGrey_pangenome/test/unit_test_7_SOFTMASK
snakemake --cores 1 --dag 2>&1 | sed -n '/^digraph/,$p' > /data/toby/EarlGrey_pangenome/test/unit_test_7_SOFTMASK/temp.txt
cat /data/toby/EarlGrey_pangenome/test/unit_test_7_SOFTMASK/temp.txt | dot -Tsvg > /data/toby/EarlGrey_pangenome/test/unit_test_7_SOFTMASK/dag.svg

# run the pipeline
snakemake --cores 8
```

The softmasked files were generated, but are empty. I have run some fixes and will try again.

**Issue Fixed (March 5, 2026):** Softmasked genome files were being created but were empty (0 bytes).

**Root Causes Identified:**
1. **Config file conflict**: The `annotate.smk` file had a `configfile: "config/config.yaml"` directive at the top, which was overriding the custom config passed via command line. This meant `softmask: true` from the test config wasn't being read.

2. **Variable scoping issue**: `SOFTMASK` and `MARGIN` variables were being defined in `annotate.smk` after the `include:` statements in the main `Snakefile`. This caused:
   - Variables to be undefined when rules in `annotate.smk` were parsed
   - Shell command receiving Python boolean `True` instead of string `"yes"`, failing the `if [ "True" == "yes" ]` test

**Solution Applied:**
Modified three files to fix variable scoping and config loading:

1. **`pangenome/rules/annotate.smk`**:
   - Removed `configfile: "config/config.yaml"` directive (line 3)
   - Removed duplicate `SOFTMASK` and `MARGIN` variable definitions (lines 12-13)
   - Added comment noting these variables are now defined in main Snakefile

2. **`pangenome/Snakefile`**:
   - Added boolean-to-string conversion for `SOFTMASK` and `MARGIN` before `include:` statements:
     ```python
     _softmask_val = config.get("softmask", False)
     SOFTMASK = "yes" if (_softmask_val is True or _softmask_val == "yes") else "no"
     _margin_val = config.get("margin", False)
     MARGIN = "yes" if (_margin_val is True or _margin_val == "yes") else "no"
     ```
   - Moved all `include:` statements to **after** all global variables are defined
   - This ensures `SOFTMASK` and `MARGIN` are available when `annotate.smk` is parsed

3. **Rule execution verified**:
   - Checked log files confirmed shell command now receives: `if [ "yes" == "yes" ]`
   - bedtools maskfasta executes correctly with proper inputs

```bash
snakemake --cores 8
```

**Result:** Softmasked genome files now generate correctly:
- Each file: **11M** in size (~10.6M bases)
- Contains properly softmasked sequences with lowercase bases indicating repeat regions
- Verified with `grep -o "[atcgn]"` showing lowercase repeat masking present

**Unit Test 7: ✓ PASSED**

### Unit Test 8: Margin (Remove Short TEs)
To test the margin feature that removes short TE annotations, I will set `margin: True` in the config file and run the pipeline. I will compare the `.filteredRepeats.bed` files generated with `margin: True` vs `margin: False` to confirm that all annotations <100bp are removed when margin is enabled. I will also check the statistics to see that the total repeat count and coverage (bp) are reduced when margin is applied.

```yaml
genome:
  genome_1: /data/toby/EarlGrey_pangenome/test/genome1.fasta
  genome_2: /data/toby/EarlGrey_pangenome/test/genome2.fasta
species: [genome_1, genome_2]
output_dir: /data/toby/EarlGrey_pangenome/test/unit_test_8_MARGIN
threads: 8
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3

# Clustering options (for combining TE libraries from multiple genomes)
skip_clustering: False  # Set to True to skip clustering (just concatenate all libraries)
clustering_identity: 0.8  # cd-hit sequence identity threshold (0.0-1.0, default 0.8)
clustering_coverage: 0.8  # cd-hit alignment coverage for shorter sequence (0.0-1.0, default 0.8)

# Output options
softmask: False  # Set to True to generate softmasked genome for each input
margin: True  # Set to True to remove short TE sequences (<100bp)

script_dir: /data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts
run_heliano: True
```

```bash
snakemake --cores 8
```

This worked as expected and all annotations <100bp were removed from the `.filteredRepeats.bed` files. The total repeat count and coverage were reduced compared to a run with `margin: False`. I can confirm that the margin feature is functioning correctly when enabled.

### Unit Test 5: Skip Clustering
To test the skip clustering feature, I will set `skip_clustering: True` in the config file and run the pipeline. I will check that the `combined_all_species.clstrd.fa` file is created and contains all sequences from all genomes without any reduction. I will compare the sequence count in the input (sum of all strains) to the output to confirm they match. I will also verify that no `.clstr` file is generated and that annotation still works with the unclustered library.

```yaml
genome:
  genome_1: /data/toby/EarlGrey_pangenome/test/genome1.fasta
  genome_2: /data/toby/EarlGrey_pangenome/test/genome2.fasta
species: [genome_1, genome_2]
output_dir: /data/toby/EarlGrey_pangenome/test/unit_test_5_SKIP_CLUSTERING
threads: 8
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3

# Clustering options (for combining TE libraries from multiple genomes)
skip_clustering: True  # Set to True to skip clustering (just concatenate all libraries)
clustering_identity: 0.8  # cd-hit sequence identity threshold (0.0-1.0, default 0.8)
clustering_coverage: 0.8  # cd-hit alignment coverage for shorter sequence (0.0-1.0, default 0.8)

# Output options
softmask: False  # Set to True to generate softmasked genome for each input
margin: False  # Set to True to remove short TE sequences (<100bp)

script_dir: /data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts
run_heliano: True
```

```bash
# check DAG to confirm combined_all_species.clstrd.fa is requested and no .clstr file is generated
mkdir -p /data/toby/EarlGrey_pangenome/test/unit_test_5_SKIP_CLUSTERING
snakemake --cores 1 --dag 2>&1 | sed -n '/^digraph/,$p' > /data/toby/EarlGrey_pangenome/test/unit_test_5_SKIP_CLUSTERING/temp.txt
cat /data/toby/EarlGrey_pangenome/test/unit_test_5_SKIP_CLUSTERING/temp.txt | dot -Tsvg > /data/toby/EarlGrey_pangenome/test/unit_test_5_SKIP_CLUSTERING/dag.svg

# run the pipeline
snakemake --cores 8
```

This worked as expected. The `combined_all_species.clstrd.fa` file was created and contains all sequences from both genomes without any reduction. The sequence count in the output matches the sum of all strains from the input libraries. No `.clstr` file was generated, confirming that clustering was skipped. Annotation still worked successfully with the unclustered library. I can confirm that the skip clustering feature is functioning correctly when enabled.

### Unit Test 6: Custom Clustering Parameters
To test the custom clustering parameters, I will set `skip_clustering: False` and adjust the `clustering_identity` and `clustering_coverage` parameters in the config file. I will run the pipeline with more stringent (e.g., 0.95/0.90). I will also check the cd-hit log files to confirm that the correct parameters were used.

```yaml
genome:
  genome_1: /data/toby/EarlGrey_pangenome/test/genome1.fasta
  genome_2: /data/toby/EarlGrey_pangenome/test/genome2.fasta
species: [genome_1, genome_2]
output_dir: /data/toby/EarlGrey_pangenome/test/unit_test_6_CUSTOM_CLUSTERING
threads: 8
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3

# Clustering options (for combining TE libraries from multiple genomes)
skip_clustering: False  # Set to True to skip clustering (just concatenate all libraries)
clustering_identity: 0.95  # cd-hit sequence identity threshold (0.0-1.0, default 0.8)
clustering_coverage: 0.90  # cd-hit alignment coverage for shorter sequence (0.0-1.0, default 0.8)

# Output options
softmask: False  # Set to True to generate softmasked genome for each input
margin: False  # Set to True to remove short TE sequences (<100bp)

script_dir: /data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts
run_heliano: True
```

```bash
snakemake --cores 8
```

This worked as expected. The output library size was larger compared to the default parameters, indicating that fewer sequences were clustered together due to the more stringent identity and coverage thresholds. The cd-hit log files confirmed that the correct parameters (0.95 identity and 0.90 coverage) were used during clustering. I can confirm that the custom clustering parameters are functioning correctly when enabled.

### Unit Test 9: Combined Options Test
To test the combined options, I will set multiple parameters in the config file (e.g., `skip_clustering: True`, `softmask: True`, `margin: True`, `run_heliano: True`, `repeatmasker_species: "fungi"`). I will run the pipeline and check that all features work together without conflicts. I will verify that the final outputs contain all expected files and that the validation messages report all options correctly.

I will need to use IPO323 and 1A5 for this test to ensure there are enough sequences for RepeatModeler to run successfully with the fungi library.

```yaml
genome:
  IPO323: /data/toby/EarlGrey_pangenome/test/IPO323.fa
  1A5: /data/toby/EarlGrey_pangenome/test/1A5.fa
species: [IPO323, 1A5]
output_dir: /data/toby/EarlGrey_pangenome/test/unit_test_9_COMBINED_OPTIONS
threads: 16
repeatmasker_species: "fungi"  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3

# Clustering options (for combining TE libraries from multiple genomes)
skip_clustering: True  # Set to True to skip clustering (just concatenate all libraries)
clustering_identity: 0.8  # cd-hit sequence identity threshold (0.0-1.0, default 0.8)
clustering_coverage: 0.8  # cd-hit alignment coverage for shorter sequence (0.0-1.0, default 0.8)

# Output options
softmask: True  # Set to True to generate softmasked genome for each input
margin: True  # Set to True to remove short TE sequences (<100bp)

script_dir: /data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts
run_heliano: True
```

```bash
snakemake --cores 16
```

This worked as expected. All features worked together without conflicts, and the final outputs contained all expected files. The validation messages correctly reported all options that were enabled in the config file. I can confirm that the combined options are functioning correctly when enabled together.

## March 5, 2026 - Optimising cores and memory usage using snakemake schedule and resources

I want to make it so the user only specifies threads in the snakemake command so that snakemake can scale and parallelise correctly. I have implemented these changes and now need to test that these work as expected.

```yaml
# EarlGrey Pangenome Pipeline Configuration
# Run with: snakemake --cores N (where N = number of cores to use)
# Example: snakemake --cores 16

genome:
  IPO323: /data/toby/EarlGrey_pangenome/test/IPO323.fa
  1A5: /data/toby/EarlGrey_pangenome/test/1A5.fa
species: [IPO323, 1A5]
output_dir: /data/toby/EarlGrey_pangenome/test/unit_test_10_CPU_MEMORY
repeatmasker_species: ""  # e.g. "arthropoda" or "" for none
custom_library: ""        # path to custom library or "" for none
iterations: 10
flank: 1000
max_consensus_seqs: 20
min_consensus_seqs: 3

# Clustering options (for combining TE libraries from multiple genomes)
skip_clustering: False  # Set to True to skip clustering (just concatenate all libraries)
clustering_identity: 0.8  # cd-hit sequence identity threshold (0.0-1.0, default 0.8)
clustering_coverage: 0.8  # cd-hit alignment coverage for shorter sequence (0.0-1.0, default 0.8)

# Output options
softmask: False  # Set to True to generate softmasked genome for each input
margin: False  # Set to True to remove short TE sequences (<100bp)

script_dir: /data/toby/miniforge3/envs/earlgrey-pan-dev/share/earlgrey-7.0.3-0/scripts
run_heliano: True
```

### Testing Resource Allocation

To verify that CPU and memory allocation is working correctly, we need to test multiple scenarios and check that Snakemake is properly scheduling jobs based on resource constraints.

#### Test 1: Basic Execution Without Memory Constraints

```bash
cd /data/toby/EarlGrey_pangenome/pangenome
snakemake --configfile ../test/unit_test_10_CPU_MEMORY/config.yaml \
  --cores 16 --dry-run -p
```

**Expected output:**
Look for lines showing thread allocation per rule in the job stats summary. RepeatMasker should show 4 threads (accounting for the 4× multiplier), while RepeatModeler and TEstrainer should show 16.

#### Test 2: Execution With Memory Constraints

```bash
# Run with both cores and memory limits (simulating 16 cores, 64GB RAM)
snakemake --configfile ../test/unit_test_10_CPU_MEMORY/config.yaml \
  --cores 16 --resources mem_mb=64000 --dry-run -p
```

**Expected behavior:**
- RepeatModeler (16GB each): Max 4 could run, but cores limit to 1
- TEstrainer (8GB each): Max 8 could run, but cores limit to 1  
- RepeatMasker (4 cores, ~2GB): Max 4 can run (cores: 16÷4=4, mem: 64÷2=32)

#### Test 3: Actual Run With Resource Tracking

```bash
# Create test directory
mkdir -p /data/toby/EarlGrey_pangenome/test/unit_test_10_CPU_MEMORY

# Copy config to test directory
cp /data/toby/EarlGrey_pangenome/pangenome/config/config.yaml \
   /data/toby/EarlGrey_pangenome/test/unit_test_10_CPU_MEMORY/config.yaml

# Edit the config as shown above (or use the config already prepared)

# Run the actual pipeline with resource monitoring
cd /data/toby/EarlGrey_pangenome/pangenome
snakemake --configfile ../test/unit_test_10_CPU_MEMORY/config.yaml \
  --cores 16 --resources mem_mb=64000 \
  --printshellcmds \
  --reason \
  2>&1 | tee ../test/unit_test_10_CPU_MEMORY/resource_test.log
```

#### Verification Commands

**1. Check Thread Allocation in Logs**

```bash
# Verify RepeatMasker thread allocation
grep "RepeatMasker.*-pa" ../test/unit_test_10_CPU_MEMORY/resource_test.log

# Expected: Should see -pa 1, -pa 2, or -pa 4 (which uses 4, 8, or 16 actual threads)
# Example: RepeatMasker ... -pa 1 ...
```

**2. Check Parallel Job Execution**

```bash
# Extract job start times to see parallel execution
grep -E "rule (repeatmasker|testrainer|repeatmodeler)" \
  .snakemake/log/*.snakemake.log | tail -20
```

**Expected pattern for RepeatMasker:**
Multiple jobs starting at similar times = parallel execution ✓

**3. Check Resource Usage During Execution** (optional, run in separate terminal)

```bash
# Monitor actual resource usage while pipeline runs
watch -n 2 'ps aux | grep -E "(RepeatMasker|RepeatModeler|TEstrainer|cd-hit)" | grep -v grep'
```

**4. Verify Memory Limits Were Respected**

```bash
# Check if any jobs were killed due to OOM
grep -i "memory\|killed\|oom" ../test/unit_test_10_CPU_MEMORY/resource_test.log

# Check job retry attempts (if memory was insufficient)
grep "Retrying" ../test/unit_test_10_CPU_MEMORY/resource_test.log
```

**Expected:** No OOM errors if memory was sufficient, or successful retries with increased memory

**5. Verify RepeatMasker Thread Math**

```bash
# Extract RepeatMasker commands from log
cd /data/toby/EarlGrey_pangenome/test/unit_test_10_CPU_MEMORY
grep "RepeatMasker.*-pa" resource_test.log | while read line; do
    # Extract -pa value
    pa_value=$(echo "$line" | grep -oP '(?<=-pa )\d+')
    actual_threads=$((pa_value * 4))
    echo "RepeatMasker -pa $pa_value → uses $actual_threads threads"
done | sort -u
```

**Expected output:**
```
RepeatMasker -pa 1 → uses 4 threads
RepeatMasker -pa 2 → uses 8 threads  (if running with more cores)
RepeatMasker -pa 4 → uses 16 threads  (if running with 64+ cores)
```

**6. Verify cd-hit Memory Parameter**

```bash
# Check cd-hit memory allocation
grep "cd-hit-est.*-M" resource_test.log | head -1

# Extract memory parameter
grep -oP "cd-hit-est.*-M \K\d+" resource_test.log | head -1
```

**Expected:** Should match or be less than requested memory (e.g., 16000 for 16GB base × attempt)

**7. Calculate Observed Parallelization**

```bash
# Count how many RepeatMasker jobs ran in parallel
cat << 'EOF' > analyze_parallelization.py
#!/usr/bin/env python3
import sys
import re
from datetime import datetime
from collections import defaultdict

# Parse log file for job timings
log_file = sys.argv[1] if len(sys.argv) > 1 else "resource_test.log"

# Track job starts
job_starts = []
with open(log_file, 'r') as f:
    for line in f:
        # Match lines like: "[Wed Mar  5 10:00:00 2026] rule repeatmasker"
        match = re.search(r'\[(.*?)\] rule (\w+)', line)
        if match:
            timestamp_str = match.group(1)
            rule_name = match.group(2)
            try:
                # Try to parse timestamp
                timestamp = datetime.strptime(timestamp_str, "%a %b %d %H:%M:%S %Y")
                job_starts.append((timestamp, rule_name))
            except:
                pass

# Group by rule
rules = defaultdict(list)
for ts, rule in job_starts:
    rules[rule].append(ts)

print("Job Parallelization Analysis:")
print("=" * 60)
for rule, times in sorted(rules.items()):
    print(f"\n{rule}: {len(times)} job(s)")
    if len(times) > 1:
        times_sorted = sorted(times)
        # Check if jobs started within 10 seconds (likely parallel)
        first_time = times_sorted[0]
        parallel = sum(1 for t in times_sorted if (t - first_time).seconds < 10)
        print(f"  - {parallel} job(s) started within 10 seconds (parallel)")
        
EOF

python3 analyze_parallelization.py
```

**Expected output example:**
```
Job Parallelization Analysis:
============================================================

repeatmasker: 4 jobs
  - 4 jobs started within 10 seconds (parallel)

testrainer: 4 jobs
  - 1 jobs started within 10 seconds (parallel)  (memory-limited)

repeatmodeler: 4 jobs
  - 1 jobs started within 10 seconds (parallel)  (core-limited)
```

**8. Final Validation Checklist**

```bash
cd /data/toby/EarlGrey_pangenome/test/unit_test_10_CPU_MEMORY
cat << 'EOF' > validation_checklist.sh
#!/bin/bash

echo "Resource Allocation Validation Checklist"
echo "=========================================="
echo ""

PANGENOME_DIR="/data/toby/EarlGrey_pangenome/pangenome"
TEST_DIR="/data/toby/EarlGrey_pangenome/test/unit_test_10_CPU_MEMORY"

# 1. Check config has no threads parameter
if grep -q "^threads:" "$TEST_DIR/config.yaml" 2>/dev/null; then
    echo "❌ FAIL: threads parameter found in config.yaml"
else
    echo "✅ PASS: No threads parameter in config.yaml"
fi

# 2. Check RepeatMasker thread allocation
if [ -f "$TEST_DIR/resource_test.log" ]; then
    rm_threads=$(grep "RepeatMasker.*-pa" "$TEST_DIR/resource_test.log" | \
                 grep -oP '(?<=-pa )\d+' | head -1)
    if [ -n "$rm_threads" ]; then
        actual=$((rm_threads * 4))
        echo "✅ PASS: RepeatMasker using -pa $rm_threads ($actual actual threads)"
    else
        echo "⚠️  WARNING: No RepeatMasker -pa found in log"
    fi
else
    echo "⚠️  WARNING: resource_test.log not found (pipeline not run yet)"
fi

# 3. Check resource directives exist in rules
if grep -q "resources:" "$PANGENOME_DIR/rules/lib_construct.smk"; then
    echo "✅ PASS: Resource directives found in lib_construct.smk"
else
    echo "❌ FAIL: No resource directives in lib_construct.smk"
fi

# 4. Check cd-hit memory parameter
if grep -q 'cd-hit-est.*-M {resources.mem_mb}' "$PANGENOME_DIR/rules/clustering.smk"; then
    echo "✅ PASS: cd-hit using dynamic memory allocation"
else
    echo "❌ FAIL: cd-hit not using resources.mem_mb"
fi

# 5. Check TEstrainer uses all cores
if grep -q "threads: workflow.cores" "$PANGENOME_DIR/rules/lib_construct.smk"; then
    echo "✅ PASS: TEstrainer configured to use all cores"
else
    echo "❌ FAIL: TEstrainer not using workflow.cores"
fi

# 6. Check for OOM errors
if [ -f "$TEST_DIR/resource_test.log" ]; then
    if grep -qi "out of memory\|oom\|killed.*memory" "$TEST_DIR/resource_test.log"; then
        # Check if retries were successful
        if grep -q "Retrying.*succeeded" "$TEST_DIR/resource_test.log"; then
            echo "⚠️  WARNING: OOM occurred but retry succeeded"
        else
            echo "❌ FAIL: Out of memory errors detected"
        fi
    else
        echo "✅ PASS: No OOM errors detected"
    fi
fi

echo ""
echo "Validation complete!"
EOF

chmod +x validation_checklist.sh
./validation_checklist.sh
```

**Expected output:**
```
Resource Allocation Validation Checklist
==========================================

✅ PASS: No threads parameter in config.yaml
✅ PASS: RepeatMasker using -pa 1 (4 actual threads)
✅ PASS: Resource directives found in lib_construct.smk
✅ PASS: cd-hit using dynamic memory allocation
✅ PASS: TEstrainer configured to use all cores
✅ PASS: No OOM errors detected

Validation complete!
```

#### Test Different Resource Scenarios

**Scenario 1: High-Core, Low-Memory (32 cores, 32GB)**
```bash
snakemake --configfile ../test/unit_test_10_CPU_MEMORY/config.yaml \
  --cores 32 --resources mem_mb=32000
```
**Expected:** Fewer parallel memory-intensive jobs, RepeatModeler limited to 2 at a time

**Scenario 2: Low-Core, High-Memory (8 cores, 128GB)**
```bash
snakemake --configfile ../test/unit_test_10_CPU_MEMORY/config.yaml \
  --cores 8 --resources mem_mb=128000
```
**Expected:** Memory not limiting, but cores limit parallelization

**Scenario 3: Balanced (64 cores, 256GB)**
```bash
snakemake --configfile ../test/unit_test_10_CPU_MEMORY/config.yaml \
  --cores 64 --resources mem_mb=256000
```
**Expected:** Optimal parallelization, both cores and memory well-utilized

### Summary of Resource Management Implementation

**Key changes made:**
1. Removed `threads` parameter from config.yaml
2. Added `threads:` directives to all rules (using `workflow.cores` or calculated)
3. Added `resources: mem_mb` to memory-intensive rules (RepeatModeler, TEstrainer, cd-hit, HELIANO, mergeRepeats, divergence calculation)
4. Fixed RepeatMasker 4× thread multiplier issue (`-pa N` → uses `4N` threads)
5. Implemented retry scaling for OOM scenarios (`lambda wildcards, attempt: base_memory * attempt`)

**Benefits:**
- Automatic resource scaling from laptop (8 cores, 16GB) to HPC (64+ cores, 256GB+)
- Protection against memory exhaustion with automatic retry
- Optimal parallelization based on available resources
- Explicit documentation of resource needs in code
- Proper handling of quirky tool behaviors (RepeatMasker thread multiplier)
- Single point of control via `--cores` and `--resources` command line arguments

**Unit Test 10: ✓ READY FOR TESTING** (code complete, pending verification run)

---

**March 5, 2026 (later) - Fixed Write-Protected File Removal Issue**

**Issue:** Pipeline was stopping to ask permission to remove write-protected `.prep.orig` files during `prep_genome` step.

**Root Cause:** Input genome files have read-only permissions (`-r-xr-xr-x`). When copied with `cp {input.genome} {output.gen_prep}.orig`, the copy inherits those permissions. When `rm` tries to remove write-protected files, it prompts for user confirmation, pausing the pipeline.

**Solution:** Updated all `rm` commands to use `rm -f` (force removal without prompting):
- [lib_construct.smk](rules/lib_construct.smk#L49): `rm -f {output.gen_prep}.tmp {output.gen_prep}.orig`
- [lib_construct.smk](rules/lib_construct.smk#L56): `rm -f {output.gen_prep}.bak`
- [annotate.smk](rules/annotate.smk#L197): `rm -f {input.backup}.tmp`

This prevents interactive prompts when removing files with any permission settings.

## March 6, 2026 - Final Validation and Documentation Updates

I have completed half of unit test 10, I will now run the full test and verify all outputs and resource management features work as expected. After that, I will update the documentation to reflect the new resource management features and provide guidance on how to use them effectively.

Run the full test directing the output to a log file for detailed analysis:

```bash
cd /data/toby/EarlGrey_pangenome/pangenome/

snakemake --configfile ../test/unit_test_10_CPU_MEMORY/config.yaml \
  --cores 16 --resources mem_mb=64000 \
  --printshellcmds \
  --reason \
  2>&1 | tee ../test/unit_test_10_CPU_MEMORY/resource_test.log
```

Now verify the results:

```bash
cd /data/toby/EarlGrey_pangenome/test/unit_test_10_CPU_MEMORY
# Check RepeatMasker thread allocation
bash ./validation_checklist.sh
python3 analyze_parallelization.py
```

These all passed successfully, confirming that resource management is working as intended.

### Pipeline subsets consistent with initial implementation
In original Earl Grey, there are three commands: full pipeline with `earlGrey`, library construction with `earlGreyLibConstruct`, and annotation with an already-curated library with `earlGreyAnnotationOnly`. I would like to add these as options at the pangenome level so that users can run just the library construction or annotation steps if they wish, without needing to run the full pipeline. 

