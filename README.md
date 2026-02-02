# HToWWAnalyzer
## Setting up the framework

You can install the framework with all its dependencies via:

```
git clone https://github.com/theumesvl/HToWWAnalyzer
cd HToWWAnalyzer
git submodule init
git submodule update
```

To use the framework, you will need to setup a local environment.

1. Make sure you are on m9 or m10 (e.g. ```ssh user@m9.iihe.ac.be```). You can always also use e.g. ```ssh m9``` from any other m-machine
2. Go to topmost directory and setup python virtual env with ```python -m venv env```
3. Activate the env with ```source env/bin/activate``` 
4. Install coffea and xrootd with ```pip install coffea``` and ```pip install xrootd``` 

Everytime you want to use the analyser in your interactive shell, make sure to activate the environment ```source env/bin/activate```. To submit jobs, we will later have to provide the env path to the submission script so that your condor jobs can also activate the env. 

##Framework introduction

The framework is written entirely in python primarily using the Coffea [1] and Awkward [2] packages. Make sure to check the documention of these to gain further understanding of how things work.

[1] https://coffea-hep.readthedocs.io/en/latest/
[2] https://awkward-array.org/doc/main/

To produce nTuples that are ready for user analysis. two steps are performed:

1. Skim: this script skims over the entire datasets and via some simple cuts on the physics objects of interest (e.g. muons and jets), aims to drastically reduce the amount of data that is passed to the next, more computationally intensive step
2. Analysis: this script takes the skim as input and analyses each event to reconstruct a Higgs candidate (if possible), select a jet candidate as well as perform some gen to reco matching

The skim step outputs ```.parquet``` files that contain Awkward arrays containing the skimmed events. The analysis step finally outputs everything as a ```.root``` which can be browsed and used for easy plotting. 

## Running everything locally

First, make sure your environment is set up, as described above. You can then run either the skim or analysis scripts directly via

```python analysis/myana.py```

and 

```python skim/myskim.py```

both of which require some arguments to be passed to run. These are:

- ```--infile```: path to an input file to run on (should be either NanoAOD for skim, or parquet for analysis). This option is mutually exclusive to the --inputlist option. 
- ```--inputlist```: path to a text file containing, line per line, a list of input files to run over. This option is mutually exclusive to the --infile option. 
- ```--outdir```: directory in which all the output should be written into
- ```--tag```: a descriptive name of the directory under which the output is saved

Lastly, the skim has an exlusive tag ```--islocal``` that tells the script whether it needs to look for the input files you specify, locally or on the grid. Valid values are ```"True"``` or ```"False"``` (NB: make sure they are passed as strings!).

When submitting to the cluster, these will be passed to the scripts automatically based on a configuration file. 

## Running everything on the cluster

To submit jobs to the cluster, you need to first make sure your voms proxy is set up. You can setup your proxy by running 

```source submission/initProxy.sh```.

You can check your proxy has been set up correctly with ```voms-proxy-info```. Make sure it is valid and located under ```/user/username```, which is where condor will look for it.

Next, you need to setup working directories for your jobs that contains the information for what each job should do, which is passed on the corresponding condor job. This is done with the ```submission/write_workdirs.py``` script. Instead of passing many arguments to this script, we use a config file ```submission/submission_config.py``` to setup everything with the relevant details instead. Currently there are 9 variables that are declared in the config:


- ```filesPerJob```: declares the number of files that each job should run over. The entire fileset is then automatically split so that each job (apart from potentially the last) gets the specified number of input files
- ```processes```: this is a (python) list of all the processes that should be run over. Each process is identified by a unique tag through which the scrip interfaces with the ```HJetSamples``` repository. This repository contains all the metadata for each process as well as inclusive lists of all the NanoAOD files available for each process (these are mostly available via the grid save for the signal H+c files, which we store locally). An example list of all the currently relevant process tags is provided in the config example.
- ```tag```: a tag to uniquely identify your workdirs and output files
- ```workdirBasePath```: the path to where all your workdirs should be written. The easier option is to just write to the ```./workdirs``` directory, as in the example/
- ```envPath```: the path to your python env, so that the condor jobs can activate the env
- ```outdir```: directory where all your output is stored
- ```jobFlavour```: defines the walltime of your job
- ```shift```: an additional tag to differentiate different shifts related to systematics. currently only "nominal" is available
-```era```: an additional era tag. currently only "2018" is available

The ```write_workdirs.py``` script directly pulls all this information to setup the workdirs. It should be run with the following options:

- ```--mode```: specifies whether to run the skim or analysis. Should read either ```skim``` or ```analysis``` accordingly
- ```--skimdir```: Only to be used for the analysis step. Should specify the output directory of the skim files that you want to use as input for the analysis step (i.e. the directory where all your ```.parquet``` files are written to). Each output directory contains a ```.txt``` file containing, line by line, each of the procces tags that were processed in the skim step. The analysis step will then perform the analysis step for these same processes

Once you have created your workdirs, you can run 

```
python submission/submit_jobs.py --indir /path/to/analysis/or/skim/dir
```

It will automatically loop over a ```processes.txt``` that is copied to the skim or analysis workdir (for some given tag and shift) file that contains all your specified process tags to submit the jobs for each process. 

Finally, the output root files can be merged into one for each process via:

```hadd targetfile.root /path/tp/root/files/*root```


