
# Managing an ML project with DVC

A simple demo repository that shows how to use DVC to track data and model binaries. We will also see how to manage experiments with DVC. 

This repo has three import tags:

- `blank`: dvc has been not configured
- `dvc-tracked` : dvc configured to track data 
- `dvc-experiment`: dvc configured to track data and manage experiments

If you want to follow this tutorial step by step I suggest branching from the blank tag:

    git checkout blank
    git switch -c my-own-dvc-journey

The DVC documentation at https://dvc.org/ is the only reference for this tutorial. The documentation is excellent, please check it. 

## Setting up for DVC

To run the codes in this repo we need a Python environment with turbofats, scikit-learn and DVC. You can set the environment quickly with:

    conda create -n demo python=3.9 pip cython numba numpy pandas scikit-learn matplotlib statsmodels
    conda activate demo
    pip install -r requirements.txt

This will install DVC with pip. 

If you already have an working environment and you only need DVC, then:

    pip install dvc[ssh]

In this tutorial I will be using an *SSH remote* to store binaries, hence why you see `dvc[ssh]` in `requirements.txt`

**Note for other remote options:** See here: https://dvc.org/doc/install/linux#install-with-pip, e.g. S3, Azure, google drive, etc. If you are not sure which remote to use you can do `pip install dvc[all]`


## Start [tracking data with DVC](https://dvc.org/doc/start/data-management/data-versioning)

**Important:** DVC requires a git tracked folder. 

This repo is already git tracked. If you want to start using DVC on a project that is not tracked you would need to do `git init`

To initialize DVC in a git tracked repo run:

    dvc init
    
This will create a hidden .dvc folder for the DVC configuration file and the `cache` folder in which the different versions of our binaries will live.

To track data we use the [`dvc add`](https://dvc.org/doc/command-reference/add) command. You can either track a single binary or a directory. In the latter case DVC treats the folder as a single data artifact.

Example:

    unzip explorer_ztf_lcs.zip
    dvc add raw_data
    git add .gitignore raw_data.dvc
    git commit -m "raw_data tracked with dvc"
    git push

DVC tracks artifacts by their MD5 hashes which are stored in a `.dvc` file. These files should be git tracked.

## [Pulling and pushing data from/to cache and remote](https://dvc.org/doc/start/data-management/data-versioning#storing-and-sharing)

Try removing a file from the `raw_data` and then calling [`dvc pull`](https://dvc.org/doc/command-reference/pull). Missing data is pulled from cache. If cache does not exists, e.g. a cleanly cloned repo, `dvc pull` will pull from the remote.

(You can pull a single .dvc file or pull all dvc files in the repo).

Github is our remote for git tracked code. To backup and share large binaries we have to set a remote with DVC. Remotes are added/configured/deleted with [`dvc remote`](https://dvc.org/doc/command-reference/remote#remote)

For a shared server which is accessed via SSH:

    dvc remote add -d ssh-server ssh://my-server-url/an-absolute-path
    dvc remote modify ssh-server port my-server-port

This created an default remote entry in `.dvc/config`. If this is a shared server we may only need to set our user/key, this can be done using the --local flag

    dvc remote modify --local ssh-server user my-ssh-user-name
    dvc remote modify --local ssh-server keyfile path-to-my-key-file

With all set you then send data to the remote using

    dvc push
    
This will create a several dvc folders at `an-absolute-path` in the ssh server. 

**Note:** if you run into problems when pushing/pulling data use the -v flag

## Checking out between "data commits"

Simply change git branches or move to a commit with a different dvc file and checkout with dvc, example:

    git checkout <...>
    dvc checkout

## Making a [data pipeline](https://dvc.org/doc/start/data-management/data-pipelines)

DVC pipelines are `dvc.yaml` files which describe stages that are run sequentially, e.g. data wrangling, feature extraction, model training and metrics computation. Stating this process as a pipeline allows for easier reproducibility and task automation.

You can run the pipeline with the command

    dvc repro

Running this will create a `dvc.lock` file that should be git-tracked, also the results of the pipeline can be dvc-pushed to be shared.

The stages of the pipeline may define dependencies on certain python scripts and data artifacts. DVC detects these changes and execute the needed stages. Stages are defined using the `dvc stage add` command

The commands used to create the stages in this demo were:

    dvc stage add -n parse_raw_data -d src/parse_raw_data.py -d raw_data/ -o data python src/parse_raw_data.py 

    dvc stage add -n compute_features -p features.list -d src/compute_features.py -d data/ -o features/ python src/compute_features.py

    dvc stage add -n train_model -p train.max_depth,train.criterion -d src/train_classifier.py -d features/ -o models/ python src/train_classifier.py

    dvc stage add -n evaluate_model -d src/evaluate_classifier.py -d models/ -d features/ -M metrics.json python src/evaluate_classifier.py


Each flag in `dvc stage add` means:

- `-n` : name of the stage
- `-d` : dependencies of the stage (can be many)
- `-o` : outputs of the stage (can be many and will be dvc-tracked automatically)
- `-p` : parameters for the stage, they should appear in a `params.yaml` file
- `-M` : metrics file (more in this later)

The `params.yaml` might contain model hyperparameters, dataset proportions, random seeds, etc

**Note:** There can be several pipelines in a dvc repo, but only one per folder.

## [Metrics](https://dvc.org/doc/start/data-management/metrics-parameters-plots)

The last stage (evaluate model) set up a git tracked file called `metrics.json` 

In this case `src/evaluate_model.py` computes the f1-score of the trained model and saves this value in that file

We can check the performance of the current evaluation using

    dvc metrics show 

In general we want to iteratively improve our results by tuning and comparing models. We will now see DVC capabilities to track experiments using pipelines and metrics.

**Note:** Metrics can be numerical tables and also plots, see the examples in the dvc documentation

## [Experiment management](https://dvc.org/doc/start/experiment-management/experiments)

An ML experiment in DVC is build from pipeline, parameter file and data artifacts. 

The commands for managing experiments start with `dvc exp`

For example:

    dvc exp show

Presents a table with the results commited to `main` and the current head (`workspace`). The table includes the metrics, parameters and artifact versions.

You can run an experiment with:

    dvc exp run --set-param train.criterion='entropy'

In this case we modified the DT hyperparameter `criterion` at run time. If you do `dvc exp show` again you will say the experiment under `main`.

We can set a queue of experiments using the `--queue` flag, for example

    dvc exp run --set-param train.max_depth=1,10,100 --queue

**Note:** Parameters support ranges, in this case we have set three experiments in the queue. 

**Note:** Several parameters can be changed calling the `--set-param` (or `-S` for short) flag multiple times. We can do hyperparameter grid search with DVC in one line.

Once we have filled the queue with our desired experiments we can run them with:

    dvc queue start --jobs P

where `P` is the amount of parallel tasks. We can stop the processes with `dvc queue stop` and check if jobs have finished succesfully with `dvc queue status`.


Experiments are not tracked (non-permament). At this point we should inspect which experiment came up better using `dvc exp show` and then we can either:

- Update our workspace with a particular experiment using `dvc exp apply` and commit its results
- Create a permament branch for an experiment with `dvc exp branch`. 

Once we have commited the ones we are interested in, we can clean-up the rest with `dvc exp gc`. 

**Note:** If you need to share the full set of experiments you can use `dvc exp push/pull`


