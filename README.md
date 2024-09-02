# Python Packages - The Easy Way

## Why?

Packaging up Python code allows you to create reusable tools for yourself and the community. Using Cookiecutter templates makes it simple to follow best practices for software development and handles lots of boring set-up for you.
## What?

This tutorial shows you how to create a minimal Python package using a great [Cookiecutter](https://github.com/cookiecutter/cookiecutter) [template from Simon Boothroyd](https://github.com/SimonBoothroyd/python-template/tree/main). A more common option is the fantastic [MolSSI Cookiecutter template](https://github.com/MolSSI/cookiecutter-cms), which I've used for most of my previous projects, but Boothroyd's template comes with some nice features such as a Makefile to simplify running common commands. 

For the sake of a basic example, our minimal package will convert a standard free energy of binding (in kcal / mol) into a dissociation constant. The CLI will look something like:

```
> kcalc 10 # kcal/ mol, where the CLI takes the negative of DeltaG^o
Kd = 4.68e-08 M
```

## Prerequisites

You should have:

- [VSCode installed](https://code.visualstudio.com/download) (or your favourite editor)
- A [GitHub](https://github.com/) account
- [Mamba installed](https://mamba.readthedocs.io/en/latest/installation/mamba-installation.html) (a faster version of conda)
- **Optional**: A [CodeCov](https://about.codecov.io/) account linked to your GitHub

## 1. Create a fresh mamba environment

First, decide on a name for your project. I've gone with `kcalc` (Kd-Calculator), but I'm sure you or ChatGPT can do better.

Now, create a virtual environment including cookiecutter and a recent version of git, making sure to substitute in your package name for `kcalc`

```bash
mamba create -n kcalc python=3.11 git cookiecutter
mamba activate kcalc
```

## 2. Create a skeleton package with Cookiecutter

Navigate to the directory where you want to develop your package (a subdirectory will be created by Cookiecutter for your package) and run

```bash
cookiecutter gh:SimonBoothroyd/python-template
```

From the options,

- Include CLI
- Do not include data
- Include docs
- Do not include `griffe_insiders`

## 3. Take a tour of the skeleton 

Open the new package directory in VSCode and take a look at a few important files and directories:

- `<your package name>`: This directory will contain your code and tests. It comes pre-populated with a `tests` subdirectory.
- `LICENSE`: By default, this is the highly permissive MIT license ([tldrlegal](https://www.tldrlegal.com/) is a great site to learn about licenses).
- `README.md`: This will be the first thing a user sees when the package is hosted on GitHub, so it should be brief and informative.
- `pyproject.toml`: A configuration file which handles various tools and settings. For example, it contains project details under `[project]` and has a `[build-system]` section. This file allows you to install the project with `pip install .`.
- `docs`: Contains files required to create the documentation
- `devtools/envs/base.yaml`: Lists your packages dependencies. `mamba` is used to install these. 
- `.github/workflows`: Contains yaml files specifying workflows which will be run through GitHub actions when you push your package to GitHub. `ci.yaml` specifies the continuous integration workflow (including linting, testing, building the docs, and uploading a code coverage report) and `docs.yaml` specifies a workflow for deploying the documentation to GitHub pages.
- `.gitignore`: A file specifying which files/ directories git should ignore.

## 4. Install your (empty) package into your environment

To install your (empty) package, simply run

```bash
make env
```

in the package's base directory. Select Yes when asked if you want to remove the existing environment.

There's quite a lot going on behind the scenes here. [Make](https://www.gnu.org/software/make/manual/make.html#Overview) is a tool commonly used to build software according to rules specified in a "makefile". Looking in `Makefile` shows the rule:

```bash
env:
    mamba create     --name $(PACKAGE_NAME)
    mamba env update --name $(PACKAGE_NAME) --file devtools/envs/base.yaml
    $(CONDA_ENV_RUN) pip install --no-deps -e .
    $(CONDA_ENV_RUN) pre-commit install || true
```

where "env" is the "target", and the "recipe" is shown in the following lines. Note that because "env" doesn't correspond to a file that we want to build (which is expected by Make), we need to specify it as "phony" in the line

```bash
.PHONY: env lint format test docs-build docs-deploy
```

In fact, all of the targets in this makefile are phony, because the file is used to simplify running common commands, like those shown for installation. 

When we run `make env`, we first create an environment with the correct name, then the line

```bash
mamba env update --name $(PACKAGE_NAME) --file devtools/envs/base.yaml
```

installs all required dependencies listed in `devtools/envs/base.yaml`. Then, we install our Python package into the environment using `pip`

```bash
$(CONDA_ENV_RUN) pip install --no-deps -e .
```

The `-e` tells pip that this is an editable install, meaning that when we change the code, those changes will be reflected in the package installed in our virtual environment.

To see the packages installed, type `mamba list` in the terminal. This should include your package, listed as installed from channel "pypi" and all requirements from `base.yaml`, installed from `conda-forge`.

We can also see that our package has been installed by typing in the terminal e.g.

```bash
python
>>> import kcalc
>>> kcalc.__version__
```

Your package should import without errors, showing that it's successfully installed. Helpfully, the package already has versioning info. This is generated in e.g. `kcalc/__init__.py` using the `get_versions` function from `_version.py` . We'll come back to this later.

## 5. Write your python code

Create a file inside the `kcalc` directory called e.g. `conversion.py`. In this, write a function to compute a Kd value from a standard free energy change of binding (in kcal/mol). As an example:

```python
"""
Functions for converting between free energy and dissociation constant.
"""

import numpy as _np
from scipy import constants as _const

def calc_k(dg: float, temp: float = 298.15) -> float:
    """
    Calculate the dissociation constant (Kd) from the standard free energy of binding (ΔG).

    Args:
        dg (float): Standard free energy of binding in kcal/mol.
        temp (float, optional): Temperature in Kelvin. Default is 298.15 K.

    Returns:
        float: Dissociation constant Kd in M.
    """
    return _np.exp(dg * 1000 / ((_const.R / _const.calorie) * temp))
```

It's likely that your function will need some libraries that aren't installed in your environment already. I've used numpy and scipy; to install these, add them to `devtools/envs/base.yaml` and run `make env` again (keeping the environment which is already present).

You can have a quick play with your function by starting an interpreter on the command line

```bash
python
>>> from kcalc.conversion import calc_k
>>> calc_k(-9) # kcal / mol
```

We should add a test, which we'll run using [Pytest](https://docs.pytest.org/en/stable/). Inside the `tests` directory, `conftest.py` sets up some useful Pytest [fixtures](https://docs.pytest.org/en/latest/explanation/fixtures.html), and e.g. `test_kcalc.py` contains a test which checks that your package can be imported. Add a test for your conversion function. For example

```python
import importlib
  
from ..conversion import calc_k

def test_kcalc():
    assert importlib.import_module("kcalc") is not None
  
def test_calc_k():
    assert calc_k(0) == 1
```

where `..` specifies that the `conversion` module is located up a directory. You could also do `from kcalc.conversion import calc_k`, but relative imports make it easier to change the package name. We can run the tests from the base directory with

```bash
make test
```

Looking in `Makefile`, we can see that this runs

```bash
test:
    $(CONDA_ENV_RUN) pytest -v $(TEST_ARGS) $(PACKAGE_DIR)/tests/
```

with `TEST_ARGS`

```bash
TEST_ARGS := -v --cov=$(PACKAGE_NAME) --cov-report=term --cov-report=xml --junitxml=unit.xml --color=yes
```

meaning that Pytest will generate a code coverage report - this tells us the percentage of lines which are run by the tests. We should aim for high coverage, but be aware that high coverage is not a guarantee of good tests. 

If your test(s) pass, you now have a functional package, and you're almost ready to "save" our code by creating a git commit. However, we should enforce consistent style and avoid semantic and stylistic errors by formatting and linting before we commit. Do this with

```bash
make format
make lint
```

Inspecting `Makefile` shows that both commands use [Ruff](https://github.com/astral-sh/ruff), a very fast linter and formatter written in rust. After fixing any errors highlighted and rerunning the commands, we're not ready to "save" our code by committing.

Typing

```bash
git status
```

will show a list of new and modified files. You can "stage" modified files with

```bash
git add -u
```

You can add any desired new files with e.g.

```bash
git add kcalc/conversion.py
```

All of these "staged" changes can now be "commited" with

```bash
git add -A
```

Running `git status` again should now show many files staged to commit. We can now commit them with

```bash
git commit -m "Add dG to Kd conversion function"
```

using `-m` to add a short description of the changes. To write a more detailed description, simply omit `-m "Add dG to Kd conversion function"` and you'll be able to edit the commit message with a text editor. A commit is a unit of change, which records a snapshot of your code and specifies the changes made since the last commit to reach the current state. You can see a list of commits with `git log`.

Note that because we have installed and configured the [pre-commit](https://pre-commit.com/) package (see `.pre-commit-config.yaml`), our commit would have been blocked if our code was not correctly formatted. This helps us avoid lots of small commits after each time we format. 

Finally, let's create a command line interface (CLI) so that we can easily run conversions without starting Python. In the template-generated `_cli.py`, add a simple interface to your conversion function using [click](https://click.palletsprojects.com/en/latest/quickstart/). Do this in the `main` function. An example is

```python
"""A CLI for ``kcalc``."""

import click
  
from .conversion import calc_k
  

@click.command
@click.argument("neg_dg", type=float)
@click.option(
    "--temp",
    type=float,
    default=298.15,
    help="Temperature in Kelvin. Default is 298.15 K.",
)
def main(neg_dg: float, temp: float):
    """
    Calculate Kd from a standard free energy of binding (in kcal/mol).
    The standard free energy of binding should be entered as its negative
    (e.g. enter -10 kcal /mol as '10').
    """
    kd = calc_k(-neg_dg, temp)
    print(f"Kd = {kd:.2e} M")
```

where I've chosen to input the negative of the standard binding free energy on the command line to avoid issues with click interpreting the `-` as the start of a flag. 

Now, typing e.g.

```bash
kcalc 10
```

outputs `Kd = 4.68e-08 M`.  To see why this works, look in `pyproject.toml` for the lines

```toml
[project.scripts]
kcalc = "kcalc._cli:main"
```

This means that once you've installed your package with pip, inputting `kcalc` on the command line will run the function `main` from the `kcalc._cli` module.

Now, format, lint, stage, and commit your changes as before.

## 6. Commit and Push to GitHub

Now we have a minimal functional package, let's push it to GitHub with the GitHub CLI. First, install the GitHub command line tool

```bash
mamba install gh
```

Now, log in with

```bash
gh auth login
```

(either authentication method is fine).  Now, create and push the repo with 

```bash
gh repo create
```

Select mostly default options, but make sure to choose

- Push an existing local repository to GitHub
- Add a remote
- Push commits from local branch

Now, take a look at your repo on GitHub. Click on the "Actions" tab and have a look at the workflow runs. There should be two workflows, corresponding to the two yaml files in the package `.github/workflows` directory. Make sure that the "CI" workflow has completed successfully, and have a look at the logs/ yaml file to see what it's doing. At this stage, the "docs" workflow will fail - we'll fix that next.

## 7. Publish and Expand Documentation

First, let's get the documentation GitHub Actions workflow working. The documentation is built using [Material for MKDocs](https://squidfunk.github.io/mkdocs-material/) to generate a documentation site from Markdown. However, by default, the template will attempt to use some packages which are only available to sponsors of the project (see [Insiders](https://squidfunk.github.io/mkdocs-material/insiders/)). To avoid this, edit the "Build and Deploy Documentation" section in `.github/workflows/docs.yaml` to remove the offending lines, so that it looks like

```yaml
    - name: Build and Deploy Documentation
      run: |
        git config --global user.name 'GitHub Actions'
        git config --global user.email 'actions@github.com'
        git config --global --add safe.directory "$PWD"
        git remote set-url origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/${{ github.repository }}

        git fetch --all --prune

        make env

        make docs-deploy VERSION="$VERSION"
```

Next, on your GitHub repository, go to Settings -> Actions -> General -> Workflow Permissions, and select "Read and write permissions" - this will allow the actions run to publish your documentation. Now, commit your changes to `docs.yaml` and push them to GitHub with

```bash
git push
```

Check that the "docs" workflow run completes successfully. Click on the "documentation" link at the bottom of `README.md` - this should take you to your newly built site! The homepage will display the content of your `README.md`. There will also be and "API reference" tab, with auto-generated docs for your modules and conversion function, and a "Development" tab, documenting some of the useful `make` commands.

Where have the docs been built? On your GitHub repo, take a look at your branches. A new branch called `gh-pages` has been generated. Looking at this, we can see that "actions-user" has deployed our docs.

Let's improve and expand the documentation slightly. The important files are those in `docs` and the `mkdocs.yaml` configuration file. `docs/index.md` is responsible for creating the site home page. It's content is simply `--8<-- "README.md"`, which instructs the `markdown-include` plugin for MkDocs to insert the README file. 

It's convenient to host the documentation locally while we're editing, so we don't have to push to GitHub and wait for the CI to run every time we want to inspect a change. To do this, type

```bash
mkdocs serve
```

and follow the link. Now, when you edit and save a file, this will be immediately reflected in the documentation. You can test this by making some improvements to `README.md` (for example, your package cannot be installed through conda). 

Next, let's improve the documentation for the conversion function. Navigate to the auto-generated documentation on the locally-hosted site (e.g. API reference -> conversion). Then, in your code, update the docstring for your conversion function to include a "Notes"
section which displays the equation we've used. Ensure that the equation is formatted to work with [MathJax](https://squidfunk.github.io/mkdocs-material/reference/math/?h=mathjax#mathjax-mkdocsyml) (which is configured in `mkdocs.yml)`. For example

```python
def calc_k(dg: float, temp: float = 298.15) -> float:
    """
    Calculate the dissociation constant (Kd) from the standard free energy of binding (ΔG).

    Args:
        dg (float): Standard free energy of binding in kcal/mol.
        temp (float, optional): Temperature in Kelvin. Default is 298.15 K.

    Returns:
        float: Dissociation constant Kd in M.

    Notes:
        The equation used to calculate Kd is:

        $$
        K_d = e^{\\frac{\\Delta G}{RT}}
        $$
  
        where $\\Delta G$ is the standard free energy of binding,
        $R$ is the gas constant, and $T$ is the temperature in Kelvin.
    """
```

Check that this is rendered nicely on your site. 

Finally, let's add a new section to the site, called "Getting Started". First, edit the `nav` section in `mkdocs.yml` to be

```yaml
nav:
- Home:
  - Overview: index.md
- API reference: reference/
- Development: development.md
- Getting Started: getting_started.md
```

Now, create the corresponding `getting_started.md` file in the `docs` directory

```markdown
# Getting Started

Add some helpful documentation here...
```

and check that a new tab has appeared in your site next to "Development". 

If everything looks fine on your local site, you can now commit your changes, push them to GitHub, and let your docs be auto-generated by the "docs" Actions workflow.

## 8. Versioning

Code versioning is important for reproducibility. We'll use [semantic versioning](https://semver.org/), a widely-adopted versioning scheme which uses the MAJOR.MINOR.PATCH format. We'll version our code `0.1.0`: 

- Major version 0: Indicates initial development phase and that our package is not yet stable.
- Minor version 1: First minor release with some initial functionality.
- Patch version 0: No patches or bug fixes have been applied to this version.

To release this version, go to the GitHub repo -> Releases (in right-hand column) -> Create a new release. Select "Choose a tag", enter "0.1.0", and select "Create new tag: 0.1.0 on publish". Add a title, describe the release, and hit "Publish release". 

As well as creating a release, we've added the tag "0.1.0" to our last commit. To see this, run 

```bash
git pull
```

on the command line, then

```bash
git log
```

and look for the tag. A nice feature of this Cookiecutter template is that the `_version.py` script will use this git tag to determine the version, and this is dynamically used to set the version in when we install with pip - in `pyproject.toml`, see the lines

```toml
[tool.setuptools.dynamic]
version = {attr = "kcalc.__version__"}
```

To see this, first reinstall the package with

```
pip install --no-deps -e .
```

Now, check the version displayed for your package by

```bash
conda list
```

and when you run

```bash
python
>>> import kcalc
>>> kcalc.__version__
```

A nice feature of this template the use of [mike](https://github.com/jimporter/mike) to manage multiple version of your docs. This means that if you change the version and rebuild the docs, the old version of the docs will be accessible through a drop-down menu at the top of the site.
## 9. Code Coverage

The final thing we need to get working is the code coverage display - the badge on your README will currently display "unknown". To get this working, we need to configure our repo to work with CodeCov. Sign up to CodeCov with your GitHub account, go to the CodeCov list of your GitHub repos, and select "Configure" for the tutorial repo (you may need to hit "Resync" to get your repo to show up). Under the "Coverage" tab, from "Step 2", copy the CODECOV_TOKEN. On your GitHub repo, go to Settings -> Secrets and variables -> Actions and add this as a repository secret with the name "CODECOV_TOKEN".

Now, make sure that this token is used in the CI workflow by modifying the "CodeCov" section in `ci.yaml` to

```
    - name: CodeCov
      uses: codecov/codecov-action@v3.1.1
      with:
        token: ${{ secrets.CODECOV_TOKEN }}
        file: ./coverage.xml
        flags: unittests
```

Now, commit and push your changes. In a few minutes, interactive code coverage reports should be generated on the CodeCov site and your README badge should display the coverage.

We're done! 

## Final Thoughts

We now have a minimal Python package which follows several best practices for software development, with continuous integration and continuous deployment of the documentation. However, there are many things we've skipped over or done poorly. For one, we've skipped almost all discussion of the code in favour of how to package it. A couple of areas for improvement are handling units properly (e.g. with [pint](https://pint.readthedocs.io/en/stable/) ) and making our tests more comprehensive (for example see how to [parameterise test functions](https://docs.pytest.org/en/latest/how-to/parametrize.html)). Some great resources for software design with Python are:

- [ArjanCodes YouTube Channel](https://www.youtube.com/watch?v=SjUQLryotAk&list=PLC0nd42SBTaNuP4iB4L6SJlMaHE71FG6N)
- [Software Design by Example Online Book](https://third-bit.com/sdxpy/)

Also, to fully streamline installation, we could make our package installable from the Python Package Index (PyPI) with pip, or from the `conda-forge` conda channel with conda or mamba. Note that pip is specifically designed for managing Python packages, while conda/mamba can manage multiple languages. If you just want to install a Python package to run it from the command line directly (e.g. if we only cared about using the CLI from our package), installing with [pipx](https://github.com/pypa/pipx) is a good option. This installs the dependencies in an isolated environment and makes the command (e.g. `kcalc`) globally available (you don't have to have a specific environment activated). However, note that for both pip and pipx, you need to specify the dependencies in `pyproject.toml`, which we haven't done.

An example of a nice package written with this template is [Simon Boothroyd's ABSOLV](https://github.com/SimonBoothroyd/absolv) for absolute solvation free energy calculations. For an example of a slightly more complete command line converter between standard free energies and dissociation constants, you could see my [k2dg package](https://github.com/fjclark/k2dg) which is [available on PyPI](https://pypi.org/project/k2dg/) and can be installed with `pipx install k2dg` or `pip install k2dg`.

As well as building tools for others, Python packages can help you organise your personal code. For example, you could create a package containing utility functions you use on a regular basis. Importing your functions from a single package rather than repeatedly typing then out in each script makes bugs easier to fix and improvements easier to apply. An example of a package like this is Thomas Lohr's [kitchensink](https://github.com/tlhr/kitchensink). 
