# TauMLTools

## Introduction

Tools to perform machine learning studies for tau lepton reconstruction and identification at CMS.

## How to install

### Setup for pre-processing
Root-tuples production steps (both big-tuple and training tuple) require CMSSW environment
```sh
export SCRAM_ARCH=slc7_amd64_gcc700
cmsrel CMSSW_10_6_20
cd CMSSW_10_6_20/src
cmsenv
git cms-merge-topic -u cms-tau-pog:CMSSW_10_6_X_tau-pog_boostedTausMiniFix
git clone -o cms-tau-pog -b master git@github.com:cms-tau-pog/TauMLTools.git
scram b -j8
```

### Setup for training and testing
The following steps (training, testing, etc.) should be run using LCG or conda-based environment setup.

#### LCG environment
On sites where LCG software distribution is available (e.g. lxplus) it is enough to source `setup.sh`:
```sh
source /cvmfs/sft.cern.ch/lcg/views/LCG_97apython3/x86_64-centos7-clang10-opt/setup.sh
```
Currently supported LCG distribution version is LCG_97apython3.

#### conda environment
In cases when LCG software is not available, we recommend to setup a dedicated conda environment:
```sh
conda create --name tau-ml python=3.7
conda install tensorflow==1.14 pandas scikit-learn matplotlib statsmodels scipy pytables root==6.20.6 uproot lz4 xxhash cython
```

To activate it use `conda activate tau-ml`.

N.B. All of the packages are available through the conda-forge channel, which should be added to conda configuration before creating the environment:
```sh
conda config --add channels conda-forge
```

## How to produce inputs

Steps below describe how to process input datasets starting from MiniAOD up to the representation that can be directly used as an input for the training.

### Big root-tuples production

The big root-tuples, `TauTuple`, contain an extensive information about reconstructed tau, seeding jet, and all reco objects within tau signal and isolation cones: PF candidates, pat::Electrons and pat::Muons, and tracks.
`TauTuple` also contain the MC truth that can be used to identify the tau decay.
Each `TauTuple` entry corresponds to a single tau candidate.
Branches available within `TauTuple` are defined in [Analysis/interface/TauTuple.h](https://github.com/cms-tau-pog/TauMLTools/blob/master/Analysis/interface/TauTuple.h)
These branches are filled in CMSSW module [Production/plugins/TauTupleProducer.cc](https://github.com/cms-tau-pog/TauMLTools/blob/master/Production/plugins/TauTupleProducer.cc) that converts information stored in MiniAOD events into `TauTuple` format.

[Production/python/Production.py](https://github.com/cms-tau-pog/TauMLTools/blob/master/Production/python/Production.py) contains the configuration that allows to run `TauTupleProducer` with `cmsRun`.
Here is an example how to run `TauTupleProducer` on 1000 DY events using one MiniAOD file as an input:
```sh
cmsRun TauMLTools/Production/python/Production.py sampleType=MC_18 inputFiles=/store/mc/RunIIAutumn18MiniAOD/DYJetsToLL_M-50_TuneCP5_13TeV-madgraphMLM-pythia8/MINIAODSIM/102X_upgrade2018_realistic_v15-v1/00000/A788C40A-03F7-4547-B5BA-C1E01CEBB8D8.root maxEvents=1000
```

In order to run a large-scale production for the entire datasets, the CMS computing grid should be used via CRAB interface.
Submission and status control can be performed using [crab_submit.py](https://github.com/cms-tau-pog/TauMLTools/blob/master/Production/scripts/crab_submit.py) and [crab_cmd.py](https://github.com/cms-tau-pog/TauMLTools/blob/master/Production/scripts/crab_cmd.py) commands.

#### 2018 root-tuple production steps
1. Go to `$CMSSW_BASE/src` and load CMS and environments:
   ```sh
   cd $CMSSW_BASE/src
   cmsenv
   source /cvmfs/cms.cern.ch/common/crab-setup.sh
   ```
1. Enable VOMS proxy:
   ```sh
   voms-proxy-init -rfc -voms cms -valid 192:00
   export X509_USER_PROXY=`voms-proxy-info -path`
   ```
1. Submit task in a config file (or a set of config files) using `crab_submit.py`:
   ```sh
   crab_submit.py --workArea work-area --cfg TauMLTools/Production/python/Production.py --site T2_CH_CERN --output /store/group/phys_tau/TauML/prod_2018_v2/crab_output TauMLTools/Production/crab/configs/2018/CONFIG1.txt TauMLTools/Production/crab/configs/2018/CONFIG2.txt ...
   ```
   * For more command line options use `crab_submit.py --help`.
   * For big dataset file-based splitting should be used
   * For Embedded samples a custom DBS should be specified
1. Regularly check task status using `crab_cmd.py`:
   ```sh
   crab_cmd.py --workArea work-area --cmd status
   ```
1. Once all jobs within a given task are finished you can move it from `work-area` to `finished` folder (to avoid rerunning status each time) and set `done` for the dataset in the production google-doc.
1. If some jobs are failed: try to understand the reason and use standard crab tools to solve the problem (e.g. `crab resubmit` with additional arguments). In very problematic cases a recovery task could be created.
1. Once production is over, all produced `TauTuples` will be moved in `/eos/cms/store/group/phys_tau/TauML/prod_2018_v2/full_tuples`.



### Big tuples splitting

In order to be able to uniformly mix tau candidates from various datasets with various pt, eta and ground truth, the big tuples should be splitted into binned files.

For each input dataset run [CreateBinnedTuples](https://github.com/cms-tau-pog/TauMLTools/blob/master/Analysis/bin/CreateBinnedTuples.cxx):
```sh
DS=DY; CreateBinnedTuples --output tuples-v2/$DS --input-dir raw-tuples-v2/crab_$DS --n-threads 12 --pt-bins "20, 25, 30, 35, 40, 45, 50, 55, 60, 70, 80, 90, 100, 120, 140, 160, 180, 200, 225, 250, 275, 300, 350, 400, 450, 500, 600, 700, 800, 900, 1000" --eta-bins "0., 0.575, 1.15, 1.725, 2.3"
```

Alternatively, you can use python script that iterates over all datasets in the directory:
```sh
python -u TauMLTools/Analysis/python/CreateBinnedTuples.py --input raw-tuples-v2 --output tuples-v2 --n-threads 12
```

Once all datasets are split, the raw files are no longer needed and can be removed to save the disc space.

In the root directory (e.g. `tuples-v2`) txt file with number of tau candidates in each bin should be created:
```sh
python -u TauMLTools/Analysis/python/CreateTupleSizeList.py --input /data/tau-ml/tuples-v2/ > /data/tau-ml/tuples-v2/size_list.txt
```

In case you need to update `size_list.txt` (for example you added new dataset), you can specify `--prev-output` to retrieve bin statistics from the previous run of `CreateTupleSizeList.py`:
```sh
python -u TauMLTools/Analysis/python/CreateTupleSizeList.py --input /data/tau-ml/tuples-v2/ --prev-output /data/tau-ml/tuples-v2/size_list.txt > size_list_new.txt
mv size_list_new.txt /data/tau-ml/tuples-v2/size_list.txt
```

Each split root file represents a signle bin.
The file naming convention is the following: *tauType*\_pt\_*minPtValue*\_eta\_*minEtaValue*.root, where *tauType* is { e, mu, tau, jet }, *minPtValue* and *minEtaValue* are lower pt and eta edges of the bin.
Please, note that the implementation of the following steps requires that this naming convention is respected, and the code will not function properly otherwise.

### Shuffle and merge

The goal of this step create input root file that contains uniform contribution from all input dataset, tau types, tau pt and tau eta bins in any interval of entries.
It is needed in order to have a smooth gradient descent during the training.

Steps described below are defined in [TauMLTools/Analysis/scripts/prepare_TauTuples.sh](https://github.com/cms-tau-pog/TauMLTools/blob/master/Analysis/scripts/prepare_TauTuples.sh).


#### Training inputs

Due to considerable number of bins (and therefore input files) merging is split into two steps.
1. Inputs from various datasets are merged together for each tau type. How the merging process should proceed is defined in `TauMLTools/Analysis/config/training_inputs_{e,mu,tau,jet}.cfg`.
   For each configuration [ShuffleMerge](https://github.com/cms-tau-pog/TauMLTools/blob/master/Analysis/bin/ShuffleMerge.cxx) is executed. E.g.:
   ```sh
   ShuffleMerge --cfg TauMLTools/Analysis/config/training_inputs_tau.cfg --input tuples-v2 \
                --output output/training_preparation/all/tau_pt_20_eta_0.000.root \
                --pt-bins "20, 25, 30, 35, 40, 45, 50, 55, 60, 70, 80, 90, 100, 120, 140, 160, 180, 200, 225, 250, 275, 300, 350, 400, 450, 500, 600, 700, 800, 900, 1000" \
                --eta-bins "0., 0.575, 1.15, 1.725, 2.3" --mode MergeAll --calc-weights true --ensure-uniformity true \
                --max-bin-occupancy 500000 --n-threads 12  --disabled-branches "trainingWeight"
   ```
   Once finished, use `CreateTupleSizeList.py` to create `size_list.txt` in `output/training_preparation`.
1. Merge all 4 tau types together.
   ```sh
   ShuffleMerge --cfg TauMLTools/Analysis/config/training_inputs_step2.cfg --input output/training_preparation \
                --output output/training_preparation/training_tauTuple.root --pt-bins "20, 1000" --eta-bins "0., 2.3" --mode MergeAll \
                --calc-weights false --ensure-uniformity true --max-bin-occupancy 100000000 --n-threads 12
   ```

#### Testing inputs

For testing inputs there is no need to merge different datasets and tau types. Therefore, merging can be done in a single step.
```sh
ShuffleMerge --cfg TauML/Analysis/config/testing_inputs.cfg --input tuples-v2 --output output/training_preparation/testing \
             --pt-bins "20, 25, 30, 35, 40, 45, 50, 55, 60, 70, 80, 90, 100, 120, 140, 160, 180, 200, 225, 250, 275, 300, 350, 400, 450, 500, 600, 700, 800, 900, 1000" \
             --eta-bins "0., 0.575, 1.15, 1.725, 2.3" --mode MergePerEntry \
             --calc-weights false --ensure-uniformity false --max-bin-occupancy 20000 \
             --n-threads 12 --disabled-branches "trainingWeight"
```
### ShuffleMergeSpectral (Alternative shuffle and merge)

This step represents alternative Shuffle and Merge procedure that aims to minimize usage of physicical memory as well as provide direct control over the shape of final (pt,eta) spectrum in a stochastic manner. Before running the following step on HTCondor for all the samples it is essential to analyze and tune the parameters of ShuffleMergeSpectral.

Spectrum of the initial data is needed to calculate the probability to take tau candidate of some genuine tau-type and from some pt-eta bin. To generate the specturum histograms for a dataset and lists with number of entries per datafile, run:
```sh
CreateSpectralHists --output "spectrum_file.root" \
                    --output_entries "entires_file.txt" \
                    --input-dir "path/to/dataset/dir" \
                    --pt-hist "n_bins_pt, pt_min, pt_max" \
                    --eta-hist "n_bins_eta, |eta|_min, |eta|_max" \
                    --n-threads 1
```
on this step it is important to have high granularity binning to be able later to re-bin into custom, non-uniform pt-eta bins on the next step.

Alternatively, if one wants to process several datasets the following python script can be used (parameters of pt and eta binning to be hardcoded in the script):
```sh
python Analysis/python/CreateSpectralHists.py --input /path/to/input/dir/ \
                                              --output /path/to/output/dir/ \
                                              --filter ".*(DY).*" \
                                              --rewrite
```
After the following step spectrums and .txt files with the number of entries will be created in the output folder. To merge all the .txt files into one and mix the lines:
```
cat <path_to_spectrums>/*.txt | shuf - > filelist_mix.txt
```
Mixing is needed to have shuffled files within one data group. After this step, filelist_mix.txt should contain filenames and the number of entries in the corresponding files:
```
./DYJetsToLL_M-50/eventTuple_1-70.root 249243
./TTJets_ext1/eventTuple_1-308.root 59414
./TTToHadronic/eventTuple_467.root 142781
./WJetsToLNu/eventTuple_25.root 400874
...
```
After spectrums are created for all datasets, the final procedure of Shuffle and Merge can be performed with:
```sh
ShuffleMergeSpectral --cfg Analysis/config/2018/training_inputs_MC.cfg
                     --input filelist_mix.txt
                     --prefix prefix_string
                     --output <path_to_output_file.root>
                     --mode MergeAll
                     --n-threads 1
                     --disabled-branches "trainingWeight, tauType"
                     --input-spec /afs/cern.ch/work/m/myshched/public/root-tuple-v2-hists
                     --pt-bins "20, 30, 40, 50, 60, 70, 80, 90, 100, 1000"
                     --eta-bins "0., 0.6, 1.2, 1.8, 2.4"
                     --tau-ratio "jet:1, e:1, mu:1, tau:1"
                     --lastbin-disbalance 100.0
                     --lastbin-takeall true
                     --refill-spectrum true
                     --enable-emptybin true
                     --job-idx 0 --n-jobs 500
```
- `--cfg` is a configuration file where data groups are constructed, each line represents data group in the format of (name, dataset_dir_regexp, file_regexp, tau_types to consider in this group), e.g: `Analysis/config/2018/training_inputs_MC.cfg`
- `--input` is the file containing the list of input files and entries (read line-by-line). The abdolute path is not taken from this file, only `../dataset/datafile.root n_events` is read. The rest of the path (the path to the dataset folders) should be specified in the `--prefix` argument.
- `--prefix` is the prefix which will be placed before the path of each file read form `--input`. Please note that this prefix will *not* be placed before the `--input-spec` value. This value can include a remote path compatible with xrootd.
- the last pt bin is taken as a high pt region, all entries from it are taken without rejection.
- `--tau-ratio "jet:1, e:1, mu:1, tau:1"` defines proportion of TauTypes in final root-tuple.
- `--refill-spectrum` to recalculated spectrums of the input data on flight, only events and files that correspond to the current job `--job-idx` will be considered. It was studied that in case of heterogeneity within a data group, the common spectrum of the whole data group poorly represents the spectrum of the sub-chunk of this data group if we use `--n-jobs 500`, so it is needed to fill spectrum histograms in the beginning of every job, to obtain required uniformity of the output. 
- `--lastbin-takeall` due to the poor statistic in the high-pt region the option to take all candidates from last pt-bin is included (contrary to `--lastbin-takeall false`, in this case we put the requirement on n_entries in the last bin to be equal to n_entries in other bins)
- `--lastbin-disbalance` the argument is relevant in case of `-lastbin-takeall true`, it put the requirement on the ratio of (all entries up to the last bin) to the (number of entries in the last bin) not to be greater than required number, otherwise less events will be taken from all bins up to the last.
- `--enable-emptybin` in case of empty pt-eta bin, the probability in this bin will be set to 0 (that is needed in cases when datasets are statistically poor or the number of jobs `--n-jobs` is big in case of `--refill-spectrum true` mode)
- `--n-jobs 500 --job-idx 0` defines into how many parts to split the input files and the index of corresponding job

In order to find appropriate binning and `--tau-ratio` in correspondence to the present statistics it might be useful to execute one job in `--refill-spectrum false --lastbin-takeall false` mode and study the output of `./out/*.root` files. In the \<DataGroupName>_n.root files the number of entries in required  `--pt-bins --eta-bins` can be found. \<DataGroupName>.root files show the probability of accepting candidate from corresponding pt-eta bin. 

#### ShuffleMergeSpectral on HTCondor
ShuffleMergeSpectral can be executed on condor through the [law](https://github.com/riga/law) package. To run it, first install law following [this](https://github.com/riga/law/wiki/Usage-at-CERN) instructions. Then, set up the environment 
```sh
cd $CMSSW_BASE/src
cmsenv
cd TauMLTools/Analysis/law
source setup.sh
law index
```
Jobs can be submitted running
```
law run ShuffleMergeSpectral --version vx --params --n-jobs N
```
where *--params* are the [ShuffleMergeSpectral](https://github.com/cms-tau-pog/TauMLTools/blob/master/Analysis/bin/ShuffleMergeSpectral.cxx#L28-L52) parameters. In particular

   - *--input* has been renamed to *--input-path*
   - *--output* has been renamed to *--output-path*

**NOTA BENE**: *--output-path* should be a full path, otherwise the output will be lost
The full list of parameters accepted by law and ShuffleMergeSpectral can be printed with the commands

```
ShuffleMergeSpectral --help
law run ShuffleMergeSpectral --help
```

Jobs are created by the script using the *--start-entry* and *--end-entry* parameters.

Additional arguments can be used to control the condor submission:

   - *--workflow local* will run locally. If omitted, condor will be used
   - *--max-runtime* condor runtime in hours
   - *--max-memory* condor RAM request in MB
   - *--batch-name* batch name to be used on condor. Default is "TauML_law"

At this point, the worker will start reporting the job status. As long as the worker is alive, it will automatically resubmit failed jobs. The worker can be killed with **ctrl+C** once all jobs have been submitted. Failed jobs can be resubmitted running the same command used in the first submission (from the same working directory).

A *data* directory is created. This directory contains information about the jobs as well as the log, output and erorr files created by condor.

#### Validation
A validation can be run on shuffled samples to ensure that different parts of the training set have compatible distributions.
To run the validation tool, a ROOT version greater or equal to 6.16 is needed:
```
source /cvmfs/sft.cern.ch/lcg/views/LCG_97apython3/x86_64-centos7-clang10-opt/setup.sh
```
Then, run:
```
python TauMLTools/Production/scripts/validation_tool.py  --input input_directory \
                                                         --id_json /path/to/dataset_id_json_file \
                                                         --group_id_json /path/to/dataset_group_id_json_file \
                                                         --output output_directory \
                                                         --n_threads n_threads \
                                                         --legend > results.txt
```
The *id_json*  (*group_id_json*) points to a json file containing the list of datasets names (dataset group names) and their hash values, used to identify them inside the shuffled ROOT tuples. These files are needed in order to create a unique identifier which can be handled by ROOT. These files are produced at the shuffle and merge step.
The script will create the directory "output_directory" containing the results of the test.
Validation is run on the following ditributions with a Kolmogorov-Smirnov test:

- dataset_id, dataset_group_id, lepton_gen_match, sampleType
- tau_pt and tau_eta for each bin of the previous
- dataset_id for each bin of dataset_group_id

If a KS test is not successful, a warning message is print on screen.

Optional arguments are available running:
```
python TauMLTools/Production/scripts/validation_tool.py --help
```

A time benchmark is available [here](https://github.com/cms-tau-pog/TauMLTools/pull/31#issue-510206277).

### Production of flat inputs

In this stage, `TauTuple`s are transformed into flat [TrainingTuples](https://github.com/cms-tau-pog/TauMLTools/blob/master/Analysis/interface/TrainingTuple.h) that are suitable as an input for the training.

The script to that performs conversion is defined in [TauMLTools/Analysis/scripts/prepare_TrainingTuples.sh](https://github.com/cms-tau-pog/TauMLTools/blob/master/Analysis/scripts/prepare_TrainingTuples.sh).

1. Run [TrainingTupleProducer](https://github.com/cms-tau-pog/TauMLTools/blob/master/Analysis/bin/TrainingTupleProducer.cxx) to produce root files with the flat tuples. In `prepare_TrainingTuples.sh` several instances of `TrainingTupleProducer` are run in parallel specifying `--start-entry` and `--end-entry`.
   ```sh
   TrainingTupleProducer --input /data/tau-ml/tuples-v2-training-v2/training_tauTuple.root --output output/tuples-v2-training-v2-t1-root/training-root/part_0.root
   ```
1. Convert root files into HDF5 format:
   ```sh
   python TauMLTools/Analysis/python/root_to_hdf.py --input output/tuples-v2-training-v2-t1-root/training-root/part_0.root \
                                                    --output output/tuples-v2-training-v2-t1-root/training/part_0.h5 \
                                                    --trees taus,inner_cells,outer_cells
   ```

## Training NN

1. The code to read the input grids from the file is implemented in cython in order to provide acceptable I/O performance.
   Cython code should be compiled before starting the training. To do that, you should go to `TauMLTools/Training/python/` directory and run
   ```sh
   python _fill_grid_setup.py build
   ```
1. DeepTau v2 training is defined in [TauMLTools/Training/python/2017v2/Training_p6.py](https://github.com/cms-tau-pog/TauMLTools/blob/master/Training/python/2017v2/Training_p6.py). You should modify input path, number of epoch and other parameters according to your needs in [L286](https://github.com/cms-tau-pog/TauMLTools/blob/master/Training/python/2017v2/Training_p6.py#L286) and then run the training in `TauMLTools/Training/python/2017v2` directory:
   ```sh
   python Training_p6.py
   ```
1. Once training is finished, the model can be converted to the constant graph suitable for inference:
   ```sh
   python TauMLTools/Analysis/python/deploy_model.py --input MODEL_FILE.hdf5
   ```

## Testing NN performance

1. Apply training for all testing dataset using [TauMLTools/Training/python/apply_training.py](https://github.com/cms-tau-pog/TauMLTools/blob/master/Training/python/apply_training.py):
   ```sh
   python TauMLTools/Training/python/apply_training.py --input "TUPLES_DIR" --output "PRED_DIR" \
           --model "MODEL_FILE.pb" --chunk-size 1000 --batch-size 100 --max-queue-size 20
   ```
1. Run [TauMLTools/Training/python/evaluate_performance.py](https://github.com/cms-tau-pog/TauMLTools/blob/master/Training/python/evaluate_performance.py) to produce ROC curves for each testing dataset and tau type:
   ```sh
   python TauMLTools/Training/python/evaluate_performance.py --input-taus "INPUT_TRUE_TAUS.h5" \
           --input-other "INPUT_FAKE_TAUS.h5" --other-type FAKE_TAUS_TYPE \
           --deep-results "PRED_DIR" --deep-results-label "RESULTS_LABEL" \
           --prev-deep-results "PREV_PRED_DIR" --prev-deep-results-label "PREV_LABEL" \
           --output "OUTPUT.pdf"
   ```
   Or modify [TauMLTools/Training/scripts/eval_perf.sh](https://github.com/cms-tau-pog/TauMLTools/blob/master/Training/scripts/eval_perf.sh) according to your needs to produce plots for multiple datasets.

#### Examples

Evaluate performance for Run 2:
```sh
python TauMLTools/Training/python/evaluate_performance.py --input-taus /eos/cms/store/group/phys_tau/TauML/DeepTau_v2/tuples-v2-training-v2-t1/testing/tau_HTT.h5 --input-other /eos/cms/store/group/phys_tau/TauML/DeepTau_v2/tuples-v2-training-v2-t1/testing/jet_TT.h5 --other-type jet --deep-results /eos/cms/store/group/phys_tau/TauML/DeepTau_v2/predictions/2017v2p6/step1_e6_noPCA --output output/tau_vs_jet_TT.pdf --draw-wp --setup TauMLTools/Training/python/plot_setups/run2.py
```

Evaluate performance for Phase 2 HLT:
```sh
python TauMLTools/Training/python/evaluate_performance.py --input-taus DeepTauTraining/training-hdf5/even-events/even_pt_20_eta_0.000.h5 --other-type jet --deep-results DeepTauTraining/training-hdf5/even-events-classified-by-DeepTau_odd --output output/tau_vs_jet_Phase2.pdf --setup TauMLTools/Training/python/plot_setups/phase2_hlt.py
```
