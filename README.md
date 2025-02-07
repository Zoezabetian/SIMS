# **SIMS**: Scalable, Interpretable Modeling for Single-Cell RNA-Seq Data Classification

SIMS is a pipeline for building interpretable and accurate classifiers for identifying any target on single-cell rna-seq data. The SIMS model is based on [a sequential transformer](https://arxiv.org/abs/1908.07442), a transformer model specifically built for large-scale tabular datasets.

SIMS takes in a list of arbitrarily many expression matrices along with their corresponding target variables. We assume the matrix form `cell x gene`, and NOT `gene x cell`, since our training samples are the transcriptomes of individual cells.

The code is run with `python`. To use the package, we recommend using a virtual environment such as [miniconda](https://docs.conda.io/en/latest/miniconda.html) which will allow you to install packages without harming your system `python`.  

## Installation
If using conda, run 
1. Create a new virtual environment with `conda create --name=<NAME> python=3.9`
2. Enter into your virtual environment with `conda activate NAME`

Otherwise, enter your virtual environment of choice and
1. Install the SIMS package with `pip install --use-pep517 git+https://github.com/braingeneers/SIMS.git`
2. Set up the model training code in a `MYFILE.py` file, and run it with `python MYFILE.py`. A tutorial on how to set up training code is shown below.

## Training and inference
The sims library uses a cell-by-gene matrix. This means our input data to the model should be 
an (M, N) matrix of M cells with expression levels across N different genes. The data should be log1p normalized before model training and model inference. 

To train a model, we can set up a SIMS class in the following way:

```python 
from scsims import SIMS
from pytorch_lightning.loggers import WandbLogger
import scanpy as sc

logger = WandbLogger(offline=True)

data = an.read_h5ad('mydata.h5ad')
#Perform some light filtering
sc.pp.filter_cells(adata, min_genes=100)
sc.pp.filter_genes(adata, min_cells=3)
#Transform the data for model ingestion
sc.pp.normalize_total(adata)#Normalize counts per cell
sc.pp.log1p(adata) ### Logarithmizing the data
sc.pp.scale(adata) #Scale mean to zero and variance to 1

sims = SIMS(data=data, class_label='class_label')
sims.setup_trainer(accelerator="gpu", devices=1, logger=logger)
sims.train()
```

This will set up the underlying dataloaders, model, model checkpointing, and everything else we need. Model checkpoints will be saved every training epoch. 

To load in a model to infer new cell types on an unlabeled dataset, we load in the model checkpoint, point to the label file that we originally trained on, and run the `predict` method on new data.

```python
sims = SIMS(weights_path='myawesomemodel.ckpt')# If the model has been trained on GPU move the weights to CPU, this is the case for our pretrained models
#SIMS(weights_path=checkpoint_path,map_location=torch.device('cpu'))

unlabeled_data = an.read_h5ad('my/new/unlabeled.h5ad')
#Process the data the same way you processed the training data. For all our pretrained models we followed this steps.
sc.pp.filter_cells(unlabeled_data, min_genes=100)
sc.pp.filter_genes(unlabeled_data, min_cells=3)
#Transform the data for model ingestion
sc.pp.normalize_total(unlabeled_data)#Normalize counts per cell
sc.pp.log1p(unlabeled_data) ### Logarithmizing the data
sc.pp.scale(unlabeled_data) #Scale mean to zero and variance to 1
#Perform the predictions
cell_predictions = sims.predict(unlabeled_data)
```

Finally, to look at the explainability of the model, we similarly run 
```python
explainability_matrix = sims.explain('my/new/unlabeled.h5ad') # this can also be labeled data, of course 
```

## Custom training jobs / logging
To customize the underlying `pl.Trainer` and SIMS model params, we can initialize the SIMS model like 
```python 
from pytorch_lightning.loggers import WandbLogger
from pytorch_lightning.callbacks import EarlyStopping, LearningRateMonitor
from scsims import SIMS
import anndata as an

adata = an.read_h5ad("my_labeled_data.h5ad")  # can read h5 using anndata as well
wandb_logger = WandbLogger(project=f"My Project", name=f"SIMS Model Training") # set up the logger to log data to Weights and Biases

sims = SIMS(data=adata, class_label='class_label')
sims.setup_model(n_a=64, n_d=64, weights=sims.weights)  # weighting loss inversely proportional by label freq, helps learn rare cell types (recommended)
sims.setup_trainer(
    logger=wandb_logger,
    callbacks=[
        EarlyStopping(
            monitor="val_loss",
            patience=50,
        ),
        LearningRateMonitor(logging_interval="epoch"),
    ],
    num_epochs=100,
)
sims.train()
```
This will train the SIMS model on the given expression matrices with target variable given by the `class_label` column in each label file.

## Using SIMS inside github codespaces
If you are using SIMS only for predictions using an already trained model, github codespaces is the recommended way to use this tool. You can also use this pipeline to train it in smaller datasets as the computing services offered in codespaces are modest.
To use this tool in github codespaces start by forking the repo in your github account. Then create a new codespace with the SIMS repo as the Repository of choice.
Once inside the newly created environment pull the latest SIMS image:
```docker
docker pull jmlehrer/sims:latest
```
Run the docker container mounting the file folder containing datasets and model checkpoints to the filesystem:
```docker
docker run -it -v /path/to/local/folder:/path/in/container [image_name] /bin/bash
```
Run main.py to check if the installation has been completed. You can alter this file as shown above to perform the different tasks.
```bash
python main.py
```
