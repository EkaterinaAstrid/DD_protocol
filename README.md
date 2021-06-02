# The Deep Docking protocol

Deep docking (DD) is a deep learning-based tool developed to accelerate docking-based virtual screening. Using a docking program of choice, the method allows to virtually screem extensive chemical libraries 50 times faster than conventional docking. For further details into the processes behind DD, please refer to our paper (https://doi.org/10.1021/acscentsci.0c00229). This repository provides all the scripts required to run different stages of DD, and also slurm programs to automatically perform the protocol on a computing cluster using either Glide SP or FRED docking. The protocol can be trivially adapted to any other docking program.

If you use DD in your research, please cite:

Gentile F, Agrawal V, Hsing M, Ton A-T, Ban F, Norinder U, et al. *Deep Docking: A Deep Learning Platform for Augmentation of Structure Based Drug Discovery.* ACS Cent Sci 2020:acscentsci.0c00229.


## Requirements
The only installation step required for DD is to create a conda environment and install the following packages within it:
* rdkit
* tensorflow >= 1.14.0
* pandas
* numpy
* keras
* matplotlib
* scikit-learn

The majority of the DD steps can be run on regular CPU cores; for model training and inference however, the use of GPUs is recommended. To run the automated version (see below, B section), a computing cluster with slurm job scheduler is required.

# Test data
A complete example of DD iteration for testing the protocol is available from https://drive.google.com/drive/folders/1s1wjoNXZR1wijVABgLl6HeKzokAlxcAG?usp=sharing. A DD-prepared version of the ZINC20 library (as available in March 2021) that can readily be screened with the protocol is available at https://drive.google.com/drive/folders/1-XlN3spOI-bAKhRdfIhA_Q0l2EsITeaD?usp=sharing



## Help
For help with the options of a specific script, type

```
python script.py -h
```


## A. Run Deep Docking manually
Here we present in details how to use each individual script constituting DD. If you have access to an HPC cluster, you may want to consider running the automated protocol using a job scheduler (see Run automated Deep Docking on HPC clusters section below).

### i. Preparing a database
In order to be prepared for DD virtual screening, the chemical library must be in SMILES format. DD requires to calculate the Morgan fingerprint of radius 2 and size 1024 bits of each molecule, represented as the list of the indexes of bits that are set to 1. It is recommended to split the library of SMILES into a number of evenly populated files to facilitate other steps such as random sampling and inference, and place these files into a new folder. This reorganization can be achieved for example with the `split` bash command. For example, consider a `smiles.smi` file with a billion compounds, to obtain 1000 files of 1 million molecules each you can run:

```bash
split -d -l 1000000 smiles.smi smile_all_ --additional-suffix=.smi
```

Ideally the number of files should be equal to the number of CPUs used for random sampling. After this step, stereoisomers, tautomers and protomers should be calculated for each molecule and stored in txt files (e.g. smiles_all_1.txt).

Once the SMILES have been prepared, activate the conda environemnt with rdkit and calculate Morgan fingerprints using the `morgan_fing.py` script (located in the `utilities` folder):

```bash
python morgan_fing.py --smile_folder_path path_smiles_folder/smiles_folder --folder_name path_output_morgan_folder/output_morgan_folder --tot_process num_cpus
```
which will create all the fingerprints and place them in `path_output_morgan_folder/output_morgan_folder`. --tot_process controls how many files will be processed in parallel using multiprocessing.

### ii. Phase 1. Random sampling of molecules
In phase 1 molecules are randomly sampled from the database to generate or augment the training set. During the first iteration, this phase also generates validation and test sets.

Create a project folder. To run phase 1, run the following scripts (form *scripts_1* folder) sequentially with the activated conda environment:

```bash
python molecular_file_count_updated.py --project_name project_name --n_iteration current_iteration --data_directory  left_mol_directory --tot_process num_cpus --tot_sampling  molecules_to_dock
python sampling.py --project_name  project_name --file_path path_to_project_without_project_name --n_iteration current_iteration --data_director left_mol_directory --tot_process num_cpus --train_size train_size --val_size val_size
python sanity_check.py  --project_name project_name --file_path path_to_project_without_project_name --n_iteration current_iteration
python extracting_morgan.py --project_name project_name --file_path path_to_project_without_project_name --n_iteration current_iteration --morgan_directory morgan_directory --tot_process num_cpus
python extracting_smiles.py -pt project_name -fp path_to_project_without_name -it current_iteration -smd smile_directory -t_pos num_cpus
```

* `molecular_file_count_updated.py` determines the number of molecules to be sampled from each file of the database. The sample sizes (per million) are stored in `Mol_ct_file_updated_project_name.csv` file created in the `left_mol_directory` directory.

* `sampling.py` randomly samples the specified number of molecules for the training, validation and testing sets (the validation and testing sets are generated only in the first iteration). 

* `sanity_check.py` removes overlaps between the sets.

* `extracting_morgan.py` and `extracting_smiles.py` extract Morgan fingerprints and SMILES for the molecules that were randomly sampled, and save them in `morgan` and `smiles` folders inside the directory of the current iteration.

**IMPORTANT:** For `molecular_file_count_updated.py` AND `sampling.py` the option `left_mol_directory` must be the directory from where molecules are sampled; for iteration 1, `left_mol_directory` is the directory where the Morgan fingerprints of the database are stored; BUT for successive iterations this must be the path to `morgan_1024_predictions` folder of the previous iteration. This will ensure that sampling is done progressively on better scoring subsets of the database over the course of DD.

### iii. Phase 2 and phase 3. Ligand preparation and docking
After phase 1 is completed, molecules grouped in the *smiles* folder can be prepared and docked to the target. Use your favourite tools for this step. It is important that docking results are saved as SDF files in a *docked* folder in the current iteration directory, keeping the same name convention of the files in the *smile* folder (the name of the originating set (training, validation, testing) must always be present in the name of the respective SDF file with docking results).

### iv. Phase 4. Neural network training
In phase 4, deep neural network models are trained with the docking scores from the previous phase. Run the following scripts from *scripts_2*, after activating the environment:

```bash
python extract_labels.py --project_name project_name --file_path path_to_project_without_project_name --iteration_no current_iteration --tot_process 3 -score score_keyword
python simple_job_models_manual.py --iteration_no current_iteration --morgan_directory morgan_directory --time 00-04:00 --file_path project_path --number_of_hyp num_hyperparameters --total_iterations number_total_iterations --is_last is_this_last_iteration (False/True) --number_mol num_molecules_test/valid --percent_first_mols percent_first_molecules --percent_last_mols percent_last_mols -recall recall_value 
```
* `extract_labels.py` extracts docking scores for model training. It should generate three comma-spaced files, `training_labels.txt`, `validation_labels.txt` and `testing_labels.txt` inside the current iteration folder.

* `simple_job_models_manual.py` creates bash scripts to run model training using the `progressive_docking.py` script. These scripts are generated inside the `simple_job` folder in the current iteration. Note that if `--recall` is not specified, the recall value will be set to 0.9.

The bash scripts generated by `simple_job_models.py` in the iteration directory, *simple_job* foldder, should be then run on GPUs to train DD models. The resulting models will be saved in the `all_models` folder in the current iteration.

### v. Phase 5. Selection of best model and prediction of virtual hits
In phase 5 the models are evalauted with a grid search, and the model with the highest precision is selected for predicting scores of all the molecules in the database. This step will create a `morgan_1024_predictions` folder in the iteration directory, which will contain all the molecules that are predicted as virtual hits. To run phase 3, use the following scripts from *scripts_2* with the environment activated:

```bash
python hyperparameter_result_evaluation.py --n_iteration current_iteration --data_path project_path --morgan_directory morgan_directory --number_mol num_molecules --recall recall_value
python simple_job_predictions_manual.py --project_name project_name --file_path path_to_project_without_name --n_iteration current_iteration --morgan_directory morgan_directory

```

* `hyperparameter_result_evaluation.py` evaluates the models generated in phase 2 and select the best (most precise) one.

* `simple_job_predictions.py` creates bash scripts to run the predictions over the full database using the `Prediction_morgan_1024.py` script. Inference scripts will be saved in the `simple_job_predictions` folder of the current iteration, and they should be run on GPU nodes in order to predict virtual hits from the full database. Prospective hits will be saved in `morgan_1024_predictions` folder of the current iteration, together with their virutal hit likeness.

### vi. Final docking phase
After the last iteration of DD is complete, SMILES of all or a ranked subset of the predicted virtual hits can be obtained for the final docking. Ranking is based on the probabilities of being virtual hits. Use the following script (availabe in `utilities`).

```bash
python final_extraction.py -smile_dir path_to_smile_dir -prediction_dir path_to_predictions_last_iteration -processors n_cpus -mols_to_dock num_molecules_to_dock
```

Executing this script will return the SMILES of all the predicted virtual hits of the last iteration or the top `num_molecules_to_dock` molecules ranked by their virtual hit likeness. If *mols_to_dock* is not specified, all the prospective hits will be extracted. Virtual hit likeness will also be returned in a separated file. These molecules can be then docked into the target of interest.


## B. Run automated Deep Docking on HPC clusters
As part of this repository, we provide a series of scripts to run the process on a slurm cluster using the preparation and docking tools that we regularly employ in virtual screening campaigns. The workflow can be trivially adapted to any other set of tools by modifying the scripts of phase 2, 3 and 4. Additionally, the user will need to either modify the headers of the slurm scripts or pass the #SBATCH values from command line in order to satisfy the requirements of the cluster that is being used. 

### i. Automated library preparation
In our DD automated version, SMILES preparation is performed using OpenEye tools (flipper, https://docs.eyesopen.com/applications/omega/flipper.html and tautomers, https://docs.eyesopen.com/applications/quacpac/tautomers/tautomers.html, both require license). Use the `compute_states.sh` script provided in `utilities` to submit a preparation job for each original SMILES file:

```bash
for i in $(ls smiles/smile_all_*.smi); do sbatch compute_states.sh $i library_prepared; done
```

THe SMILES will be prepared and saved into the `library_prepared` folder. Note that this program will enumerate all the unspecified chiral centers of the molecules and assign unique names to each isomer, and then calculate the dominant tautomer at pH 7.4. The next step is the calculation of Morgan fingerprints; run:

```bash
sbatch --cpus-per-task num_cpu compute_morgan_fp.sh library_prepared library_prepared_fp num_cpu name_conda_environment
```
to calculate the Morgan fingerprints and save them in `library_prepared_fp`.

### ii. Project file preparation
Create a `logs.txt` file with parameters that will be read by the slurm scripts, and save it in the project folder. An example file is provided in `utilities` (remove the comments delimited by #).

### iii. Automated phase 1
Run:

```bash
sbatch --cpus-per-task n_cpus phase_1.sh n_iteration n_cpus path_to_project_without_name project_name training_sample_size conda_env_name
```

### iv. Automated phase 2
The provided automated version of DD works with Glide docking or FRED docking; the choice of the program (listed in `logs.txt`) influences both how the SMILES are translated to 3D structures and how they are successively docked. For creating conformations for Glide docking, run:

```bash
sbatch phase_2_glide.sh 1 60 ~/DeepDocking/projects protein_test_automated cpu-partition
```

OR for creating conformations for FRED docking, run:

```bash
sbatch phase_2_fred.sh 1 60 ~/DeepDocking/projects protein_test_automated cpu-partition
```

### v. Automated phase 3
For Glide docking, run:

```bash
sbatch phase_3_glide.sh 1 600 ~/DeepDocking/projects protein_test_automated
```

For FRED docking, run:

```bash
sbatch phase_3_fred.sh 1 60 ~/DeepDocking/projects protein_test_automated cpu-partition
```

### iv. Automated phase 4
Run:

```bash
sbatch phase_4.sh current_iteration 3 ~/DeepDocking/projects protein_test_automated gpu-partition 11 1 0.01 0.90 00-04:00 dd-env
```

### v. Automated phase 5
Run:

```bash
sbatch phase_5.sh 1 ~/DeepDocking/projects protein_test_automated 0.90 gpu-partition dd-env
```

### vi. Automated final phase
Run:

```bash
sbatch --cpus-per-task 60 final_extraction.sh ~/DeepDocking/library_prepared /DeepDocking/projects/protein_test/iteration_11/morgan_1024_predictions 60 'all_mol' dd-env
```
