[![Project Status: Active – The project has reached a stable, usable
state and is being actively
developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![Python Version](https://img.shields.io/pypi/pyversions/cellshape.svg)](https://pypi.org/project/cellshape)
[![PyPI](https://img.shields.io/pypi/v/cellshape.svg)](https://pypi.org/project/cellshape)
[![Downloads](https://pepy.tech/badge/cellshape)](https://pepy.tech/project/cellshape)
[![Wheel](https://img.shields.io/pypi/wheel/cellshape.svg)](https://pypi.org/project/cellshape)
[![Development Status](https://img.shields.io/pypi/status/cellshape.svg)](https://github.com/Sentinal4D/cellshape)
[![Tests](https://img.shields.io/github/workflow/status/Sentinal4D/cellshape/tests)](
    https://github.com/Sentinal4D/cellshape/actions)
[![Coverage Status](https://coveralls.io/repos/github/Sentinal4D/cellshape/badge.svg?branch=master)](https://coveralls.io/github/Sentinal4D/cellshape?branch=master)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

<img src="https://github.com/Sentinal4D/cellshape/blob/main/img/cellshape.png" 
     alt="Cellshape logo by Matt De Vries">

# 3D single-cell shape analysis of cancer cells using geometric deep learning


This is a package for **automatically learning** and **clustering** cell
shapes from 3D images. Please refer to our preprint on bioRxiv [here](https://www.biorxiv.org/content/10.1101/2022.06.17.496550v1)

**cellshape** is available for everyone.

## Graph neural network
<https://github.com/Sentinal4D/cellshape-cloud> Cellshape-cloud is an
easy-to-use tool to analyse the shapes of cells using deep learning and,
in particular, graph-neural networks. The tool provides the ability to
train popular graph-based autoencoders on point cloud data of 2D and 3D
single cell masks as well as providing pre-trained networks for
inference.

## Clustering
<https://github.com/Sentinal4D/cellshape-cluster>

Cellshape-cluster is an easy-to-use tool to analyse the cluster cells by
their shape using deep learning and, in particular,
deep-embedded-clustering. The tool provides the ability to train popular
graph-based or convolutional autoencoders on point cloud or voxel data
of 3D single cell masks as well as providing pre-trained networks for
inference.

<https://github.com/Sentinal4D/cellshape-voxel>

## Convolutional neural network
Cellshape-voxel is an easy-to-use tool to analyse the shapes of cells
using deep learning and, in particular, 3D convolutional neural
networks. The tool provides the ability to train 3D convolutional
autoencoders on 3D single cell masks as well as providing pre-trained
networks for inference.  

## Point cloud generation
<https://github.com/Sentinal4D/cellshape-helper>

<figure>
<img src="img/github_cellshapes.png" style="width:100.0%" alt="Fig 1: cellshape workflow" />
</figure>

## Data structure

Our data is structured in the following way:

```
Data/
    all_data_stats.csv
    Plate1/
        stacked_pointcloud/
            Binimetinib/
                0010_0001_accelerator_20210315_bakal01_erk_main_21-03-15_12-37-27.ply
                ...
            Blebbistatin/
            ...
    Plate2/
    Plate3/
```

## Usage
```python
import torch
from torch.utils.data import DataLoader
from datetime import datetime
import logging

import cellshape_cloud as cscloud
import cellshape_cluster as cscluster
from cellshape_cloud.vendor.chamfer_distance import ChamferLoss
from cellshape_cloud.helpers.reports import get_experiment_name


input_dir = "/home/mvries/Documents/CellShape/DatasetForTesting/"
batch_size = 20
learning_rate_autoencoder = 0.00001
learning_rate_clustering = 0.000001
num_features = 128
num_clusters = 3
num_epochs_autoencoder = 1
num_epochs_clustering = 3
k=20
encoder_type="dgcnn"
decoder_type = "foldingnetbasic"
output_dir = "/home/mvries/Documents/Testing_output/"
gamma = 1
alpha = 1.0
divergence_tolerance = 0.01
update_interval = 1


autoencoder = cscloud.CloudAutoEncoder(num_features=num_features, 
                         k=k,
                         encoder_type=encoder_type,
                         decoder_type=decoder_type)

dataset = cscloud.PointCloudDataset(input_dir)

dataloader = DataLoader(dataset, batch_size=batch_size, shuffle=True)

criterion = ChamferLoss()

optimizer = torch.optim.Adam(
    autoencoder.parameters(),
    lr=learning_rate_autoencoder * 16 / batch_size,
    betas=(0.9, 0.999),
    weight_decay=1e-6,
)

name_logging, name_model, name_writer, name = get_experiment_name(
        model=autoencoder, output_dir=output_dir
    )

logging_info = name_logging, name_model, name_writer, name

now = datetime.now().strftime("%d/%m/%Y %H:%M:%S")
logging.basicConfig(filename=name_logging, level=logging.INFO)
logging.info(f"Started training model {name} at {now}.")

output_cloud = cscloud.train(autoencoder, 
                             dataloader,
                             num_epochs_autoencoder, 
                             criterion, 
                             optimizer,
                             logging_info)

autoencoder = output_cloud[0]


model = cscluster.DeepEmbeddedClustering(autoencoder=autoencoder, 
                               num_clusters=num_clusters)

dataloader = DataLoader(dataset, batch_size=batch_size, shuffle=False) 
# it is very important that shuffle=False here!
dataloader_inf = DataLoader(dataset, batch_size=1, shuffle=False) 
# it is very important that batch_size=1 and shuffle=False here!

optimizer = torch.optim.Adam(
    model.parameters(),
    lr=learning_rate_clustering * 16 / batch_size,
    betas=(0.9, 0.999),
    weight_decay=1e-6,
)

reconstruction_criterion = ChamferLoss()
cluster_criterion = torch.nn.KLDivLoss(reduction="sum")

cscluster.train(
    model,
    dataloader,
    dataloader_inf,
    num_epochs_clustering,
    optimizer,
    reconstruction_criterion,
    cluster_criterion,
    update_interval,
    gamma,
    divergence_tolerance,
    logging_info
)
```

## For developers
* Fork the repository
* Clone your fork
```bash
git clone https://github.com/USERNAME/cellshape
```
* Install an editable version (`-e`) with the development requirements (`dev`)
```bash
cd cellshape
pip install -e .[dev] 
```
* To install pre-commit hooks to ensure formatting is correct:
```bash
pre-commit install
```

* To release a new version:

Firstly, update the version with bump2version (`bump2version patch`, 
`bump2version minor` or `bump2version major`). This will increment the 
package version (to a release candidate - e.g. `0.0.1rc0`) and tag the 
commit. Push this tag to GitHub to run the deployment workflow:

```bash
git push --follow-tags
```

Once the release candidate has been tested, the release version can be created with:

```bash
bump2version release
```

## References
[1] An Tao, 'Unsupervised Point Cloud Reconstruction for Classific Feature Learning', [GitHub Repo](https://github.com/AnTao97/UnsupervisedPointCloudReconstruction), 2020
