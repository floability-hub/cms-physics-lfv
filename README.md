# CMS Physics LFV

## Overview

This Floability backpack demonstrates a distributed CMS
lepton-flavor-violation (LFV) workflow. Coffea processes 2018 NanoAOD ROOT
files through TaskVine workers and saves the resulting features as Parquet
files.

The default data profile stages four example ROOT files (approximately 756 MiB)
from the public `floability` S3 bucket. The backpack uses this small dataset to
demonstrate the distributed processing and feature-construction pipeline.

## Install Floability

Install and activate Floability by following the
[official installation instructions](https://floability.readthedocs.io/en/stable/getting-started/installation/).
Verify the installation before running the backpack:

```bash
floability --version
```

## Run the Backpack

Run these commands from the repository root. Floability prepares the pinned
software environment, stages the input data, launches TaskVine workers, and
starts JupyterLab. Open the URL printed in the terminal and run all notebook
cells. Save the notebook, then press `Ctrl+C` in the Floability terminal to
stop the interactive run and clean up its processes.

The notebook is named after the backpack, so Floability selects it
automatically without an `--entrypoint` argument.

### Local workers

Omit `--batch-type` to launch workers directly on the current machine:

```bash
floability run --backpack . --data-profile s3_test_data
```

### HTCondor workers

```bash
floability run --backpack . \
  --data-profile s3_test_data \
  --batch-type condor
```

### Slurm workers

```bash
floability run --backpack . \
  --data-profile s3_test_data \
  --batch-type slurm
```

The selected batch system must already be available and configured on the
machine where Floability is launched.

### Non-interactive execution

Use `execute` to run the notebook from beginning to end without opening a
browser. The generated `output/` directory is not part of the original
backpack, so request it explicitly when syncing results back from the
Floability instance:

```bash
floability execute --backpack . \
  --data-profile s3_test_data \
  --sync-path output
```

Add `--batch-type condor` or `--batch-type slurm` to this command to use an HPC
batch system.

## What the Workflow Does

1. Floability stages the `DYData` and `ETauData` ROOT files under
   `workflow/data/samples/test/`.
2. The notebook reads the manager name and allowed manager ports supplied by
   Floability and creates a Coffea `TaskVineExecutor`.
3. Coffea divides each dataset into 20,000-event chunks and sends them to
   TaskVine workers.
4. `selection.py` defines baseline and tight/loose electrons, muons, and
   hadronic taus; constructs the LFV channels; applies the third-lepton veto,
   exclusive-channel selection, trigger matching, and jet cleaning.
5. `kinematics.py` calculates lepton, dilepton, missing-momentum, jet, dijet,
   collinear-mass, and Higgs-frame quantities.
6. `makeDF.py` accumulates 57 output fields for each dataset, and the notebook
   writes the collected arrays to Parquet.

The current notebook configuration uses `year="2018"` and `type="mc"`. Review
those settings and the selection definitions before adapting the backpack to a
different year, dataset, or physics result.

## Outputs

The notebook writes:

```text
workflow/output/results/makeDF/2018/mc/
├── DYJets.parq
└── ETau.parq
```

Each file is a one-row Parquet table whose 57 columns contain the accumulated
per-event arrays for that dataset. It is not a conventional table with one
Parquet row per event.

A validated local run of the default profile completed 50 TaskVine tasks with
no failed tasks and produced both Parquet files. The exact worker count and
completion order depend on the execution site.

## Common Options

HPC home directories often have limited quotas. Use `--base-dir` to place
Floability instances, prepared environments, packed archives, logs, and the
default data cache on a larger project or scratch filesystem:

```bash
floability run --backpack . \
  --data-profile s3_test_data \
  --batch-type slurm \
  --base-dir "$SCRATCH/floability" \
  --data-cache-dir "$PROJECT/floability-data-cache" \
  --manager-ports 9123:9150 \
  --worker-transfer-ports 10000:11000
```

Replace `$SCRATCH` and `$PROJECT` with persistent, high-capacity locations
available at your site. Avoid temporary node-local storage when you want to
reuse cached environments and data across runs.

- `--base-dir PATH` changes the root used for instances, environment caches,
  logs, and the default data cache. Without it, Floability uses
  `~/floability-base-dir`.
- `--data-cache-dir PATH` moves only the data cache. This is useful when
  instances and reusable input data belong on different filesystems.
- `--manager-ports START:END` restricts the TaskVine manager to a port range
  permitted by the site's firewall between workers and the launch host.
- `--worker-transfer-ports START:END` restricts ports used for direct
  worker-to-worker transfers when peer transfers are enabled.
- `--sync-path PATH` copies an additional generated file or directory back to
  the backpack. The path is relative to `workflow/` and cannot escape that
  directory; repeat the option to sync multiple paths.

Worker counts and resource requests belong in `compute/compute.yml` so the
backpack carries its compute specification across sites. This example requests
2-4 workers with 4 cores, 12,000 MB of memory, and 24,000 MB of disk per
worker.

## Backpack Contents

- `workflow/cms-physics-lfv.ipynb` — TaskVine manager setup, Coffea execution,
  and Parquet output.
- `workflow/processors/makeDF.py` — Coffea processor and output-field
  accumulation.
- `workflow/utils/selection.py` — lepton definitions, channel selection,
  trigger matching, and jet cleaning.
- `workflow/utils/kinematics.py` — event-level kinematic feature construction.
- `data/data.yml` — public S3 inputs and their workflow staging locations.
- `software/environment.yml` — pinned Coffea, Dask, HEP, and TaskVine software
  environment.
- `compute/compute.yml` — TaskVine worker limits and resource requests.
