# T3 Track

## Table Of Contents

- [Introduction](#introduction)  
- [For Participants](#for_participants) 
  - [Getting Started](#getting_started) 
  - [Starting Your Development](#starting_your_development)
  - [Developing Your Dockerfile](#developing_your_dockerfile)
  - [Developing Your Algorithm](#developing_your_algorithm) 
  - [How To Get Help](#how_to_get_help)
  - [Leaderboard Ranking](#leaderboard_ranking)
    - [Baseline Thresholds](#baseline_thresholds)
    - [Leaderboard](#leaderboard)
- [For Evaluators](#for_organizers)  

## Introduction

The T1 and T2 tracks evaluate algorithms on standardized Azure CPU servers.

**Track 1**: In-memory indices with [FAISS](https://github.com/facebookresearch/faiss) as the baseline. Search would use Azure [Standard_F32s_v2 VMs](https://docs.microsoft.com/en-us/azure/virtual-machines/fsv2-series) with 32 vCPUs and 64GB RAM.

**Track 2:** Out-of-core indices with [DiskANN](https://github.com/Microsoft/diskann) as the baseline. In addition to the limited DRAM in T1, index can use an SSD for search. Search would use Azure [Standard_L8s_v2 VMs](https://docs.microsoft.com/en-us/azure/virtual-machines/lsv2-series) with 8 vCPUS, 64GB RAM and a local SSD Index constrained to 1TB.

Index construction for both tracks would use Azure [Standard_F64s_v2 VM](https://docs.microsoft.com/en-us/azure/virtual-machines/fsv2-series) with 64vCPUs, 128GB RAM and an additional 4TB of SSD to be used for storing the data, index and other intermediate data. There is a **time limit for 4 days per dataset** for index build. 

Queries will be supplied in one shot and the algorithm can execute the queries in any order. 

We will release plots for recall vs QPS separately for tracks T1 and T2.
Additionally, we will release leaderboards for T1 and T2. The  metric for the leaderboard in each track will be the sum of improvements in recall over the baseline at the target QPS over all datasets. **The target recall
for T1 is 10000 QPS and for T2 is 1500 QPS.**

Participants must submit their algorithm via a pull request and (optionally) index file(s) upload (one per participating dataset).  


## For_Participants

### Requirements

You will need the following installed on your machine:
* Python ( we tested with Anaconda using an environment created for Python version 3.8 ) and Docker.
* Note that we tested everything on Ubuntu Linux 18.04 but other environments should be possible.

### Getting_Started

This section will present a small tutorial about how to use this framework and several of the key scripts you will use throughout the development of your algorithm and eventual submission.

First, clone this repository and cd into the project directory:
```
git clone <REPO_URL>
```
Install the python package requirements:
```
pip install -r requirements_py38.txt
```
Create a small, sample dataset.  For example, to create a dataset with 10000 20-dimensional random floating point vectors, run:
```
python create_dataset.py --dataset random-xs
```
To create a smaller slice of the competition datasets (e.g. 10M slice of deep-1B), run:
```
python create_dataset.py --dataset deep-10M
```
To see a complete list of datasets, run the following:
```
python create_dataset.py --help
```

Build the docker container for the T1 or T2 baselines:
```
#for T1
python install.py --algorithm faiss  
#for T2
python install.py --algorithm diskann
```
Run a benchmark evaluation using the algorithm's definition file:
```
python run.py --algorithm faiss_t1 --dataset random-xs
python run.py --algorithm diskann-t2 --dataset random-xs
```

For the competition dataset (e.g. deep-1B), running the following command downloads a prebuilt index and runs te queries locally.
```
python run.py --algorithm faiss_t1 --dataset deep-1B
python run.py --algorithm diskann-t2 --dataset deep-1B
```

Now plot QPS vs recall:
```
python plot.py --algorithm algorithm --dataset random-xs
```
This will place a plot into the *results/* directory.

### Starting_Your_Development

First, please create a short name for your team without spaces or special characters.  Henceforth in these instructions, this will be referenced as [your_team_name].

Create a custom branch off main in this repository:
```
git checkout -b t1/[your_team_name]
```


### Developing_Your_Dockerfile

This framework evaluates algorithms in Docker containers by default.  Your algorithm's Dockerfile should live in *install/Docker.[your_team_name]*.  Your Docker file should contain everything needed to install and run your algorithm on a system with the same hardware. 

Please consult the Dockerfile [here](faiss_t3/Dockerfile) for an example.

To build your Docker container, run:
```
python install.py --dockerfile [your_team_name]
```

### Developing_Your_Algorithm

Develop and add your algorithm's python class to the [benchmark/algorithms](../benchmark/algorithms) directory.
* You will need to subclass from the [BaseANN class](../benchmark/algorithms/base.py) and implement the functions of that parent class.
* You should consult the examples already in the directory.


When you are ready to test on the competition datasets, use the create_dataset.py script as follows:
```
python create_dataset.py --dataset [sift-1B|bigann-1B|text2image-1B|msturing-1B|msspacev-1B|ssnpp-1B]
```
To benchmark your algorithm, first create an algorithm configuration yaml in your teams directory called *algos.yaml.*  This file contains the index build parameters and query parameters that will get passed to your algorithm at run-time.  Please look at [algos.yaml](../algos.yaml).

If your machine is capable of both building and searching an index, you can benchmark your algorithm using the run.py script:
```
python run.py --t3  --algorithm diskann-t2 --dataset deep-1B
```
This will write the results to the toplevel [results](../results) directory.

To build the index and upload it to Azure cloud storage without querying it:
```
python run.py --t3  --algorithm diskann-t2 --dataset deep-1B --upload-index --blob-prefix <your Azure blob container path> --sas-string <your sas string to authenticate to blob>
```
To download the index from cloud storage and query it on another machine:
```
python run.py --t3  --algorithm diskann-t2 --dataset deep-1B --download-index --blob-prefix <your Azure blob container path> --sas-string <your sas string to authenticate to blob>
```

Now you can analyze the results by running:
```
python plot.py --algorithm [your_team_name] --dataset deep-1B
```
This will place a plot of the algorithms performance into the toplevel [results](../results) directory.

The plot.py script supports other benchmarks.  To see a complete list, run:
```
python plot.py --help
```

### Submitting_Your_Algorithm

A submission is composed of the following. 
* For each dataset you are participating in, add to [algos.yaml](../algos.yaml)
  * 1 index build configuration 
  * 10 search configuration
  * Optionally an URL to download any prebuilt indices. This would help us evaluate faster, although we would build your index to verify the time limit.
* Your algorithm's python class ( placed in the [benchmark/algorithms/](../benchmark/algorithms) directory.)

If you are unable to host the index on your own Azure blob storage, please let us know and we can arrange to have it copied to organizer's account.

We will run early PRs on organizer's machine to the extent possible and provide any feedback necessary.

### How_To_Get_Help

There are several ways to get help as you develop your algorithm using this framework:
* You can submit an issue at this github repository.
* Send an email to the competition's T1/T2 organizer, harsha.v.simhadri@gmail.com
* Send en email to the competition's googlegroup, big-ann-organizers@googlegroups.com

### Leaderboard_Ranking

T3 will maintain four different leaderboards 1) one based on recall 2) one based on throughput 3) one based on power consumption and 4) one based on cost.  The details of the ranking metrics are described here.

#### Baseline_Thresholds

Thresholds of performance have been put in place for this competition, based on both queries per second (qps) and recall measured as recall@10.  For the recall leaderboard, we will rank participants by recall@10 at 2K qps.  The table below shows the baseline recall@10 for all the (knn search type) datasets near 2K qps.

|   dataset    |    qps   | recall@10 |
| ------------ | -------- | --------- |
| msturing-1B  | 2011.542 |   0.936   |
| bigann-1B    | 2058.950 |   0.949   |
| text2image-1B| 2120.635 |   0.488   |
| deep-1B      | 2002.490 |   0.937   |
| msspacev-1B  | 2190.829 |   0.901   |


Here are all the baseline recall@10 vs throughput plots for the (knn search type) datasets:
* [msturing-1B]()
* [bigann-1B]()
* [text2image-1B]()
* [deep-1B]()
* [msspacev-1B]()

#### Leaderboard

This leaderboard leverages the standard recall@10 vs throughput benchmark that has become a standard benchmark when evaluating and comparing approximate nearest neighbor algorithms.  We will rank participants based on recall@10 at track-specific target recall qps for each dataset.  The evaluation framework allows for 10 different search parameter sets and we will use the best value of recall@10 from the set.

Participants are required to commit to at least 3 datasets, and ideally more. Improvements in recall at target QPS are added to assign a score to a team. Algorithms that work on more datasets are at an advantage as they can benefit from additional scores. Any recall regression compared to the baseline on the datasets committed will be subtracted from the final score.


## For_Evaluators

