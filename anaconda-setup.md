# Setting Up Anaconda and Jupyter Notebooks

This guide will walk you through the process of installing Anaconda and running Jupyter Notebook/Lab on your system.

## 1. Download Anaconda

1. Visit the official Anaconda website: [https://www.anaconda.com/download](https://www.anaconda.com/download)
2. Download the appropriate version for your operating system (Windows, macOS, or Linux)
3. Choose the latest Python version installer

## 2. Install Anaconda

### Windows
1. Double-click the downloaded `.exe` file
2. Click "Next" to begin the installation
3. Read and accept the license agreement
4. Choose "Just Me" for installation type (recommended)
5. Select an installation location (default is recommended)
6. Advanced Options:
   - ✅ Add Anaconda to PATH (recommended)
   - ✅ Register Anaconda as default Python
7. Click "Install" and wait for the installation to complete

### macOS/Linux
1. Open Terminal
2. Navigate to the directory containing the downloaded file
3. Run the installer:
   ```bash
   bash Anaconda3-[version]-[platform].sh
   ```
4. Follow the prompts and accept the license agreement
5. Choose the installation location (default is recommended)

## 3. Verify Installation

Open a new terminal/command prompt and run:
```bash
conda --version
```
You should see the Conda version number if installation was successful.

## 4. Running Jupyter Notebook

There are several ways to launch Jupyter Notebook:

### Method 1: Using Anaconda Navigator
1. Open Anaconda Navigator
2. Click on the "Launch" button under Jupyter Notebook

### Method 2: Using Command Line
1. Open terminal/command prompt
2. Run:
   ```bash
   jupyter notebook
   ```
This will open Jupyter Notebook in your default web browser.

## 5. Running JupyterLab

JupyterLab is the next-generation web-based interface for Project Jupyter.

### Method 1: Using Anaconda Navigator
1. Open Anaconda Navigator
2. Click on the "Launch" button under JupyterLab

### Method 2: Using Command Line
1. Open terminal/command prompt
2. Run:
   ```bash
   jupyter lab
   ```

## 6. Managing Environments

### Create a new environment
```bash
conda create --name myenv python=3.9
```

### Activate an environment
```bash
conda activate myenv
```

### Install packages in the environment
```bash
conda install package_name
# or
pip install package_name
```

## 7. Common Issues and Solutions

### Jupyter not found
If Jupyter commands are not recognized, install them in your environment:
```bash
conda install jupyter
conda install jupyterlab
```

### Kernel not found
If you get kernel errors, install ipykernel:
```bash
conda install ipykernel
python -m ipykernel install --user
```

## 8. Useful Tips

1. Always activate your environment before installing new packages
2. Use `conda list` to see installed packages
3. Keep Anaconda updated:
   ```bash
   conda update --all
   ```
4. Create environment.yml files to share your environment:
   ```bash
   conda env export > environment.yml
   ```

## Additional Resources

- [Anaconda Documentation](https://docs.anaconda.com/)
- [Jupyter Documentation](https://jupyter.org/documentation)
- [Conda Cheat Sheet](https://docs.conda.io/projects/conda/en/latest/user-guide/cheatsheet.html)
