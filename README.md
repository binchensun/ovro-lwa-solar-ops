# ovro-lwa-solar-ops

Scripts and codes related to operations of OVRO-LWA for solar studies

# Realtime Pipeline


## Environment

```bash
source /opt/devel/solarpipe/conda_start.sh
conda activate lwasolarpipe
```

## directorys

- src for operation code: `/opt/devel/solarpipe/operation/ovro-lwa-solar-ops`
- caltables: `/opt/devel/solarpipe/operation/caltab`



## Update caltables

```bash
# remove old caltables
pdsh -w lwacalim[05-09] 'rm -rf /fast/solarpipe/caltables/*'

# copy new caltables
pdsh -w lwacalim[05-09] 'cp -r /opt/devel/solarpipe/operation/caltab/caltables_latest/* /fast/solarpipe/caltables/'
```


## Slurm managed run


By default, realtime pipeline runs on 10 nodes from calim cluster, every node requires 12 cpus and 32G Mem.

```bash
#!/bin/bash
#SBATCH --job-name=solar-test
#SBATCH --partition=general
#SBATCH --ntasks=10
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=12
#SBATCH --mem=32G
#SBATCH --time=14:00:00
#SBATCH --output=/lustre/solarpipe/slurmlog/%j.out
#SBATCH --error=/lustre/solarpipe/slurmlog/%j.err
#SBATCH --mail-user=pz47@njit.edu
```

To start (restart). First check the existing running job with `sinfo`, then `scancel` if necessary. Then:

```bash
sbatch runSlurm_solarPipeline.sh slow # for slow pipeline
sbatch runSlurm_solarPipeline.sh fast # for fast pipeline
```


To run slow imaging for given period of time, do:

```bash
sbatch runSlurm_solarPipeline.sh testslowfixedtime \
 "2024-09-15T20:00:00"  "2024-09-15T21:00:00" 
```

## Native-cadence fast imaging

Fast visibility data are averaged to 10 seconds by default. To preserve the
native 0.1-second integrations and make one image set per integration, run the
pipeline with:

```bash
python solar_realtime_pipeline.py \
  --slowfast fast \
  --no_average_fast \
  --file_path /path/to/fast/
```

The pipeline passes the number of native integrations to WSClean through
`-intervals-out`, groups the resulting `t####` products by time, and writes one
MFS and one fine-channel FITS product for every 0.1-second integration. Output
names include the cadence and millisecond timestamp, for example:

```text
ovro-lwa-48.lev1_mfs_100ms.2024-02-15T190100.100Z.image_I.fits
ovro-lwa-48.lev1_fch_100ms.2024-02-15T190100.100Z.image_I.fits
```

## Offline slow products for fast imaging

Realtime processing keeps only the most recent slow-visibility calibration and
all-sky products in its temporary working directories. This rolling-cache
behavior remains the default.

For offline processing, the pipeline can instead save every successfully imaged
slow timestamp to a persistent directory. Enable this behavior with
`--operation_mode offline`, provide `--slow_products_dir`, and request both the
all-sky products and self-calibration tables:

```bash
python solar_realtime_pipeline.py \
  --operation_mode offline \
  --slowfast slow \
  --save_allsky \
  --save_selfcaltab \
  --slow_products_dir /lustre/bin.chen/20240215_typeJ/slow_products \
  --file_path /lustre/bin.chen/20240215_typeJ/slow/ \
  --nolustre \
  --start_time 2024-02-15T19:01:00 \
  --end_time 2024-02-15T19:06:00
```

When persistent slow archiving is enabled, `--save_allsky` and
`--save_selfcaltab` must be supplied together so every completed bundle is
usable by the fast pipeline.

Each timestamp is copied only after its imaging step succeeds. A completed
bundle has the following layout:

```text
/lustre/bin.chen/20240215_typeJ/slow_products/
└── 2024/02/15/20240215_190150/
    ├── allsky/
    │   ├── 20240215_190150_23MHz..._allsky-image.fits
    │   ├── 20240215_190150_23MHz..._allsky-model.fits
    │   └── ...
    ├── caltables/
    │   ├── 20240215_190150_23MHz....gcal/
    │   └── ...
    └── COMPLETE
```

The temporary bundle is renamed into place only after all requested files have
been copied. Existing completed bundles are not overwritten.

Use the same directory when processing fast visibilities offline:

```bash
python solar_realtime_pipeline.py \
  --operation_mode offline \
  --slowfast fast \
  --no_average_fast \
  --slow_products_dir /lustre/bin.chen/20240215_typeJ/slow_products \
  --slow_products_warn_seconds 10 \
  --file_path /lustre/bin.chen/20240215_typeJ/fast/ \
  --nolustre \
  --start_time 2024-02-15T19:01:50 \
  --end_time 2024-02-15T19:01:59
```

For each frequency band, fast processing selects the nearest completed bundle
that contains both its self-calibration tables and its all-sky image/model
files. The selected products are copied to a disposable working directory so
the archive remains unchanged. A warning is logged when the selected slow and
fast timestamps differ by more than `--slow_products_warn_seconds`; processing
continues with the nearest products.

If `--operation_mode offline` is used without `--slow_products_dir`, the
pipeline logs a warning and falls back to the existing realtime rolling-cache
behavior for both slow and fast processing.


# Alignment to sunrise time

 Use `sunwait` (https://github.com/risacher/sunwait) in `crontab` to submit slurm script.

Need to git clone download source file.

The crontab command for daily schedule.

```bash
0 10 * * * sunwait wait rise 37.2332N 118.2872W && sbatch /lustre/peijin/ovro-lwa-solar-ops/runSlurm_solarPipeline.sh slow
```



# Beam Operations

The beam module handles solar beam scheduling and calibration.

## Solar SDF Generation

Generate solar schedule definition files (SDF) for beam observations:

```python
import beam_scheduling_sdf as bss

# Generate solar SDF for 7 days
bss.make_solar_sdf(ndays=7)
```
