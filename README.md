# SPEAR: Uniting Single-cell Gene Expressions with Protein Sequences for Cross-Species Integration

PyTorch implementation of SPEAR, a deep learning approach that couples gene expression with protein representations learnt using large protein language models. The key idea in SPEAR is to map cells from all datasets to a shared space of functionally related genes that we name macrogenes.

SPEAR takes as an input:
- multiple scRNA-seq *count* datasets from different species (AnnDatas), with labels.
- protein embeddings generated by large language models (TorchDicts)

SPEAR is composed of three modules:
- Macrogene initialization with Kmeans (scipy)
- Pretraining conditional autoencoder (scVI ZINB loss)
- Fine tuning cell clusters with weakly supervised metric learning

`Vignettes/frog_zebrafish_embryogenesis/Train Spear.ipynb` has an example of running SPEAR, scoring the results, and running differential expression on Macrogenes.

`protein_embeddings/Generate Protein Embeddings.ipynb` has an example of creating and formatting protein embeddings.


## Setting up SPEAR

### Protein Embeddings
To run SPEAR you will need protein embeddings. To generate your own protein embeddings, please see instructions in the `protein_embeddings` directory. 

Alternatively, you can use one of the publicly available protein embedding datasets we have provided.

### Requirements

SPEAR requires installation of a number of python modules. Please install them via:

```
pip install -r requirements.txt
pip install torch==1.10.2+cu113 -f https://download.pytorch.org/whl/cu113/torch_stable.html
```

## Running SPEAR

To run SPEAR, use the `train-spear.py` file.

`Vignettes/frog_zebrafish_embryogenesis/Train Spear.ipynb` has an example of running SPEAR, scoring the results, and running differential expression on Macrogenes.

`train-spear.py` accepts a number of required arguments:

- `in_data`: a csv file with the following columns

`path`: path to an Scanpy h5 file with scRNA-seq count values in the `X` field, and at least one column of cell type labels.
`species`: name of the Adata species
`embedding_path`: optionally, you can specify a path to gene embedding torch files per adata. Otherwise, SPEAR will map protein embeddings based on the default paths in `data/gene_embeddings.py`

- `in_label_col`: the name of a column present in all input AnnDatas that contains cell type labels that should be used for metric learning.
- `ref_label_col`: the name of a column present in all input AnnDatas that contains an additional cell type label that can be used to aid metric learning if the additional argumetn `--use_ref_labels` is also passed, otherwise this column will just be included in SPEAR's output AnnData. If you don't have an additional column, just set this value to the same column as `in_label_col` and do not pass the `--use_ref_labels` argument.

### Optional Arguments

`train-spear.py` also contains a number of optional arguments, a number of which are detailed below:

- `hv_genes`: Number of highly variable genes each AnnData should be subset to. Defaulted to 8000
- `num_macrogenes`: The number of macrogenes.
- `score_adatas`: should the pretraining AnnData and metric learning AnnDatas be scored? See the section on scoring AnnDatas for more info.
- `centroids_init_path`: This is a path to a `.pkl` file that contains a copy of gene to macrogene scores after HV subset and centroids initialization, but before SPEAR pretraining. When you pass this argument, SPEAR will look for a file at this location and use that file to initialize macrogene weights. If the file does not exist, SPEAR will instead create centroids and save to this location. This can be used to skip the centroids creation step if running SPEAR multiple times on the same datasets with the same genes (after HV gene selection).
- `device_num` which GPU to use

## SPEAR Output

`train-spear.py` will output a number of files during and following training.

All of these files will be in the same directory, and have the same prefix, `run_name`, which SPEAR will display at the conclusion of training.

**AnnDatas:**
- Final integrated AnnData: `{run_name}.h5ad`

This AnnData will have SPEAR embeddings in the `.X` slot.
In `.obs`, there will be columns for the original unmodified labels from `in_label_col` in the slot `labels2`, and the slot `labels` will contain those labels but with their corresponding `species` value preprended. The `ref_labels` column will contain values from the `ref_label_col`, and the `species` column will contain species values.

In the AnnData's `.obsm`, there will be a slot called `macrogenes` that will contain the macrogene values for each cell.

- Pretraining AnnData: `{run_name}_pretrain.h5ad`

The pretraining AnnData has the same format the final AnnData.

**Final Macrogene Weights:**
- Gene to macrogene final weights file: `{run_name}_genes_to_macrogenes.pkl`

**Log Files:**

There are a number of additional log files outputted:
- `{run_name}_triplets.csv` A csv with information about which triplets were mined during metric learning
- `{run_name}_epoch_scores.csv` A csv with information about scoring during metric learning
- `{run_name}_celltype_id.pkl` A pkl of a dictionary containing cell type to categorical codings used for interpreting the other log files

## Scoring SPEAR outputs

To score SPEAR outputs, either after training or during training, you will need a csv file that maps cell types between each species.

This csv should have the columns:

`{species 1 name}_cell_type`, `{species 2 name}_cell_type`... `{species n name}_cell_type`

and each row should contain the cell types from each species that should be mapped to eachother.

If a cell type is unique, you can just leave the other species' values blank.

To score while training SPEAR, add the argument `--score_adatas` and pass the name of this csv file to `--ct_map_path`. 

To score SPEAR outputs after training is finished, use the `score_adata.py` file.

`score_adata.py` takes the following arguments:

- `adata`: path to the SPEAR formatted AnnData to score
- `species1`: The species whose embeddings will be used to train a simple cell type classifier
- `species2`: The species whose embeddings will be used to test a simple cell type classifier
- `ct_map_path`: path to the csv file mapping cell types between species
- `label`: the label column for cell types. If scoring a SPEAR output, this should be `labels2`
- `scores`: the number of scores that should be calculated.









