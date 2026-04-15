# ECE 232E Project 1: Random Graphs and Random Walks

This repository contains the source code and the final report for ECE 232E Project 1. 

## Environment Setup

It is highly recommended to use `conda` to manage the environment and dependencies. The primary library used for network generation and analysis is `igraph`.

1. Create a new conda environment:
   ```bash
   conda create -n ece232_proj1 python=3.9 -y
   ```
2. Activate the environment:
   ```bash
   conda activate ece232_proj1
   ```
3. Install the required libraries:
   ```bash
   conda install -c conda-forge python-igraph numpy matplotlib jupyter -y
   ```
