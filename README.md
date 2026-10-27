# hyp3-srg bundle

hyp3-srg is ASF's HyP3 plugin for the Stanford Radar Group (SRG) SAR processor. Its `back_projection` workflow makes geocoded single-look complex images (GSLCs) directly from Sentinel-1 Level-0 raw data by time-domain back-projection onto a DEM grid, so two passes over the same area form an interferogram with a plain conjugate product. The plugin is Python; the processing itself is the SRG Fortran and C code, which lives in a separate repository and has to be compiled. This repository holds both as git submodules, pinned to the commits that were checked, so the whole algorithm can be cloned in one step:

```sh
git clone --recurse-submodules https://github.com/gchang/hyp3-srg-bundle.git
```

| Submodule | Repository | Commit | Role |
|---|---|---|---|
| `hyp3-srg` | https://github.com/ASFHyP3/hyp3-srg (tag `v0.10.2`) | c0b9c3946778dcddd3598361122c958cf9d07f41 | the plugin: downloads, DEM, workflow, product zip (`python -m hyp3_srg`) |
| `srg` | https://github.com/gchang/hyp3-srg-backprojection | 1ed982bbbb9d9666958dee877627ba23480dd45d | the SRG processor (ASFHyP3/srg `v0.4.3` with three fixes, see its `MAAP_CHANGES.md`) |

`hyp3-srg/Dockerfile` builds the processor from ASFHyP3/srg `v0.4.3`. The `srg` submodule is that release with three small fixes: work-folder paths longer than about 60 characters no longer crash the processor, snaphu compiles with GCC 15 and later, and a failed processing step now stops the workflow instead of letting it go on with empty images. Results are unchanged.

## Licenses

Each submodule keeps its own terms; nothing here adds a license. `hyp3-srg` is BSD-3-Clause (ASF Tools Team). The SRG code in `srg` is Stanford Radar Group research code delivered to ASF, with no license file; inside it, `sbas/svd.f90` is under the GNU LGPL and `snaphu_v2.0b0.0.0/` is SNAPHU by Curtis Chen (Stanford University, its own terms in `snaphu_v2.0b0.0.0/README`; noncommercial use for its CS2 solver parts). snaphu is used only by the `time_series` workflow, not by `back_projection`.

## Software

`environment.yml` holds the checked conda-forge set: Python 3.12, the plugin's run and test packages, GCC and gfortran, make, FFTW and wget.

```sh
conda env create -f environment.yml
conda activate hyp3-srg-bundle
```

The CPU build follows the builder stage of `hyp3-srg/Dockerfile`, which runs `hyp3-srg/scripts/build_proc.sh` inside the SRG folder with `USEGPU=false`. conda-forge's FFTW has no static library, so it is linked as a shared library with an rpath. From the bundle root:

```sh
export PROC_HOME=$PWD/srg      # where the plugin finds the SRG programs; needed to build and to run
(cd srg && USEGPU=false FFTW_LIB="$CONDA_PREFIX/lib/libfftw3f.so -Wl,-rpath,$CONDA_PREFIX/lib" \
    bash ../hyp3-srg/scripts/build_proc.sh)
python -m pip install --no-deps --no-build-isolation ./hyp3-srg
```

`build_proc.sh` does not stop on errors and prints "built ..." either way; check that `srg/sentinel/sentinel_raw_process_cpu` and `srg/bin/snaphu` exist afterwards. The build writes object files and programs into `srg/`, and each run downloads the EGM2008 geoid grid (149 MB) into `srg/DEM/`, so the `srg` folder must stay writable.

The author's tests (unit tests with mocks; they do not run the SRG programs):

```sh
(cd hyp3-srg && python -m pytest tests)
```

## Data

The workflow fetches everything itself:
- Sentinel-1 IW RAW Level-0 frames from ASF (`https://datapool.asf.alaska.edu/RAW/SA/<name>.zip`), about 1.6 GB each, unzipped to `.SAFE` folders in the work folder. This needs an Earthdata login, from `~/.netrc` (machine `urs.earthdata.nasa.gov`) or the `EARTHDATA_USERNAME` and `EARTHDATA_PASSWORD` environment variables.
- Precise orbits from `s1-orbits.asf.alaska.edu` (no login).
- Copernicus 30 m DEM tiles from `pangea.stanford.edu` and the EGM2008 geoid grid from `lavas-data.s3.us-west-2.amazonaws.com` (no login).

## Running one case

The plugin's README example, the two consecutive frames of 2023-12-29 over southern Nevada, limited with `--bounds` to a 0.25 by 0.2 degree box over Las Vegas. `--bounds` must come **after** the granule names: both take a list of values, so a `--bounds` placed first swallows the granules. Run it in an empty work folder; outputs go to the current folder.

```sh
export PROC_HOME=/path/to/hyp3-srg-bundle/srg
mkdir -p work && cd work
python -um hyp3_srg ++process back_projection \
    S1A_IW_RAW__0SDV_20231229T134339_20231229T134411_051870_064437_4F42-RAW \
    S1A_IW_RAW__0SDV_20231229T134404_20231229T134436_051870_064437_5F38-RAW \
    --bounds -115.30 36.05 -115.05 36.25
```

Input: the two Level-0 zips are 1,599,181,361 and 1,602,012,829 bytes (3.2 GB, unzipped into `.SAFE` folders that stay in the work folder); one DEM tile (26 MB) and the geoid grid (149 MB). Output in the work folder: `S1A_IW_RAW__0SDV_20231229T134339_20231229T134436_051870_064437_5F38.zip` (about 62 MB, stored without compression), holding the geocoded SLC `.geo` (complex64, 5401 by 1441 on the DEM grid of 1/6 by 1/2 arc second, about 4.2 by 15 m), its `.orbtiming`, `elevation.dem.rsc`, `bounds` and a parameter `.txt`.

Memory and time:
- **Memory: plan on a 32 GB worker.** The CPU processor decodes all three IW sub-swaths of a frame into memory before processing them, whatever `--bounds` says: about 16.5 GB for these frames (estimate from the buffer sizes; three sub-swath buffers of 3.9 to 4.4 GB plus the range-compressed data). It does not fit in 16 GB. hyp3-srg's README also asks for at least 32 GB.
- With `--bounds` as above, the back-projection itself is small: about 8 minutes of CPU per date for the sub-swath that covers the box, a few minutes of wall time on 4 cores, plus the 3.2 GB download. Disk: about 3.5 GB in the work folder (the `.SAFE` folders stay).
- **Without `--bounds`**, the DEM covers the whole frame pair, upsampled 6 by 2 from 30 m: a 79,272 by 28,368 grid, so about 18 GB per complex SLC file (one per frame plus the merged one) and an 18 GB zip, and hours of CPU back-projection (estimate: 6 hours or more on 4 cores). That size is meant for the GPU build (`hyp3-srg/Dockerfile.gpu`, `--gpu`).

Checked on 2026-10-25 from a fresh `git clone --recurse-submodules`, with the environment built from `environment.yml` (conda-forge GCC and gfortran 16.2.0, FFTW 3.3.11, Python 3.12): the build completes with snaphu, the tests pass (13 of 13), and the example above, run in a work folder with a 231-character path, gets through decoding without the crash the upstream SRG code had there. On the 14 GB test machine the unmodified processor then runs out of memory holding the three sub-swaths, and the workflow stops with an error naming the failed step. The values were therefore checked with a test build that processes only sub-swath IW2, which holds the whole box (8.5 GB peak, 3.6 minutes on 4 cores with the inputs already downloaded, 4.6 minutes including the download), for this pair and for the next pass (2024-01-10 frames `S1A_IW_RAW__0SDV_20240110T134338_20240110T134410_052045_064A30_E6DA` and `S1A_IW_RAW__0SDV_20240110T134403_20240110T134435_052045_064A30_76BE`). The interferogram of the two geocoded SLCs matches the official ARIA S1 GUNW `S1-GUNW-D-R-173-tops-20240110_20231229-134406-00117W_00035N-PP-2a77-v3_0_1` with a median phase difference of 0.12 rad (mean resultant length 0.970, 96% of 71,999 cells within 0.5 rad) and the interferogram from the official OPERA CSLC-S1 v1.1 bursts T173-370303/04/05-IW2 with 0.25 rad; amplitude correlates with OPERA at r = 0.84. These are the same numbers, to every digit, as from the same IW2 test build of the unchanged upstream code in a short work folder.
