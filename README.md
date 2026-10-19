# Installing a new version of LAYNII+LayNii_IDA in Neurodesk's Neurocommand
This guide describes how to install a custom or newer [**LAYNII**](https://github.com/layerfMRI/LAYNII) release together with **LayNii_IDA** and expose it through [**Neurocommand**](https://neurodesk.org/getting-started/neurocommand/linux-and-hpc/) as a normal Lmod module.

## Why Neurocontainers + Neurocommand?

[Neurodesk](https://neurodesk.org/) is the overall ecosystem rather than a single container-build command. Its desktop and application interfaces normally consume software that has already been packaged as Neurocontainers.

For a LAYNII + IDA version that is newer than, or different from, the one currently distributed by Neurodesk, it is simpler to use the underlying components directly:

- [**Neurocontainers**](https://neurodesk.org/getting-started/neurocontainers/) builds the software container from a recipe.
- [**Neurocommand**](https://neurodesk.org/getting-started/neurocommand/linux-and-hpc/) makes the resulting Apptainer container available through Lmod, so it can be used with commands such as `module load laynii/<version>`.

This avoids rebuilding the full Neurodesk desktop environment, gives direct control over the LAYNII/IDA version being used, and works well on systems without a local graphical desktop, including HPC systems.

`LayNii_IDA` itself is graphical. On a remote or headless system, its interface can be forwarded to a graphical client over a network connection, for example through an X11-based setup. That configuration is intentionally left outside the scope of this guide.

## Operating system

The commands below are written for **Linux** and use a Bash-compatible shell. They are especially suited to Linux workstations, virtual machines, WSL environments, and HPC systems, which are also the main environments targeted by Neurocommand and Apptainer.

Apptainer runs natively on Linux. On macOS or Windows, use an appropriate Linux environment or virtualization approach and adapt paths, package installation, container-runtime setup, and module initialization as required. See the [Apptainer installation documentation](https://apptainer.org/docs/admin/main/installation.html) for supported approaches on other operating systems.

## Prerequisites

The procedure requires a Linux environment with:

- [Git](https://git-scm.com/)
- [Python](https://www.python.org/)
- [uv](https://docs.astral.sh/uv/) **(recommended)** — used to manage the Python environment for Neurocontainers
- [Docker Engine](https://docs.docker.com/engine/install/) — used to build the custom Neurocontainer
- [Apptainer](https://apptainer.org/docs/user/main/quick_start.html) — used to create and run the final `.simg` container
- [Lmod](https://lmod.readthedocs.io/en/latest/) — used by Neurocommand to expose containers as loadable modules

## Set up Neurocontainers

Clone the [Neurocontainers repository](https://github.com/NeuroDesk/neurocontainers):

```bash
git clone https://github.com/NeuroDesk/neurocontainers.git
cd neurocontainers
```

Using `uv` **(recommended)**:

```bash
uv sync
```

Then run Neurocontainers commands through `uv`:

```bash
uv run sf-build --help
```

Alternatively, use a standard virtual environment:

```bash
python3 -m venv env
source env/bin/activate

python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip install -e .
```

Confirm that the build tools are available:

```bash
sf-build --help
```

See the [Neurocontainers documentation](https://neurodesk.org/getting-started/neurocontainers/) for additional information.

## Set up Neurocommand

Clone the [Neurocommand repository](https://github.com/NeuroDesk/neurocommand):

```bash
git clone https://github.com/NeuroDesk/neurocommand.git
cd neurocommand
```

Initialize the command-line setup:

```bash
bash build.sh --cli
```

See the official [Neurocommand Linux/HPC installation guide](https://neurodesk.org/getting-started/neurocommand/linux-and-hpc/) for alternative configurations.

After initialization, Neurocommand stores downloaded containers, generated wrappers, and Lmod modules under its `local/` directory.

The LAYNII and LayNii_IDA binaries do not need to be downloaded manually. The `laynii+ida` Neurocontainers recipe specifies the required files from the [LAYNII releases](https://github.com/layerfMRI/LAYNII/releases), and Neurocontainers retrieves them while preparing the build.

## 1. Add the `laynii+ida` recipe

Copy the complete `laynii+ida` recipe folder into the `recipes/` directory of your Neurocontainers checkout:

```bash
cp -r /path/to/laynii+ida /path/to/neurocontainers/recipes/
```

The recipe should already contain the desired LAYNII/IDA release URLs and required dependencies.

The resulting structure should look like:

```text
neurocontainers/
└── recipes/
    └── laynii+ida/
        └── build.yaml
```

## 2. Build the Docker image

Go to your Neurocontainers checkout:

```bash
cd /path/to/neurocontainers
```

If you are using `uv`, no manual activation of the Neurocontainers Python environment is required because the commands can be run through `uv run`.

### Docker permissions

Before running `sf-build`, make sure your user can access Docker without `sudo`:

```bash
docker info
```

On Linux systems using Docker Engine, if this returns a permission error for `/var/run/docker.sock`, add your user to the `docker` group:

```bash
sudo usermod -aG docker "$USER"
```

Apply the new group membership by logging out and back in, restarting your Linux/WSL session, or for the current shell:

```bash
newgrp docker
```

Then verify that Docker works:

```bash
docker info
```

> **Note:** Membership in the `docker` group grants privileges effectively equivalent to root access on the machine. See the [Docker post-installation documentation](https://docs.docker.com/engine/install/linux-postinstall/) for more information.

Build the recipe with Docker:

```bash
uv run sf-build laynii+ida \
    --architecture x86_64 \
    --recreate \
    --method docker
```

If you installed Neurocontainers using a manually activated virtual environment instead of `uv`, run the equivalent command without `uv run`:

```bash
sf-build laynii+ida \
    --architecture x86_64 \
    --recreate \
    --method docker
```

The Docker image name and tag are derived from the `name` and `version` fields in `build.yaml`. For example:

```text
laynii:2.10.0
```

Confirm that the image exists:

```bash
docker images laynii
```

Optionally check that LayNii_IDA has no unresolved shared libraries:

```bash
docker run --rm laynii:<version> \
    sh -lc 'ldd "$(command -v LayNii_IDA)" | grep "not found" || echo "All shared libraries found"'
```

You may see a message such as:

```
mesg: ttyname failed: Inappropriate ioctl for device
```

This can be ignored in this context. It is emitted because the container shell is running non-interactively and does not have a normal terminal attached. It does not indicate a problem with LayNii_IDA or its shared libraries.

A successful check should still show the path to `LayNii_IDA` followed by:

```
All shared libraries found
```

## 3. Convert the Docker image to an Apptainer image

Choose your Neurocommand checkout and a build date. Neurocommand expects container names in the form:

```text
name_version_YYYYMMDD.simg
```

For example:

```bash
NEUROCOMMAND_DIR=/path/to/neurocommand
VERSION=<laynii-version>
BUILD_DATE=$(date +%Y%m%d)

CONTAINER_ID="laynii_${VERSION}_${BUILD_DATE}"
CONTAINER_DIR="$NEUROCOMMAND_DIR/local/containers/$CONTAINER_ID"

mkdir -p "$CONTAINER_DIR"
```

Convert the local Docker image to an Apptainer image:

```bash
apptainer build --force \
    "$CONTAINER_DIR/$CONTAINER_ID.simg" \
    "docker-daemon:laynii:$VERSION"
```

See the [Apptainer build documentation](https://apptainer.org/docs/user/main/build_a_container.html) for more information.

## 4. Register the container with Neurocommand

Copy Neurocommand's transparent-Singularity helper files into the container directory:

```bash
cp -a "$NEUROCOMMAND_DIR/local/transparent-singularity/." "$CONTAINER_DIR/"
```

Then enter the container directory and run the Neurocommand helper used to inspect the image and generate command wrappers and the Lmod module:

```bash
cd "$CONTAINER_DIR"

bash run_transparent_singularity.sh \
    --container "$CONTAINER_ID.simg"
```

For containers installed through Neurocommand's normal download mechanism, this registration step is performed automatically. Because this custom LAYNII + IDA image was built locally rather than downloaded through Neurocommand, `run_transparent_singularity.sh` is run manually.

The script detects the executables exposed by the container, generates host-side wrapper commands that invoke them through Singularity/Apptainer, and creates the corresponding Lmod module.

Neurocommand will generate a module such as:

```text
laynii/<version>
```

## 5. Load and run LAYNII + IDA

Make sure Lmod is installed at this point. If you would like for the module command to available every time you open a new terminal, you can run the following code:

```bash
echo 'source /usr/share/lmod/lmod/init/bash' >> ~/.bashrc
```

To enable laynii to be executed through Lmod, add the Neurocommand module directory to Lmod:

```bash
module use "$NEUROCOMMAND_DIR/local/containers/modules"
```

Load LAYNII:

```bash
module load laynii/<version>
```

Run a LAYNII command:

```bash
LN2_LAYERS -help
```

Run IDA:

```bash
LayNii_IDA
```

Verify the loaded module with:

```bash
module list
```

## Extra: Using packages already available in Neurodesk

For software that is already distributed as a Neurodesk container, there is no need to create a custom Neurocontainers recipe or manually run `run_transparent_singularity.sh`. Neurocommand handles the installation and registration automatically.

From the Neurocommand directory, search for an available package:

```bash
bash containers.sh <package>
```

For example, to look for ITK-SNAP:

```bash
bash containers.sh itksnap
```

Install the available container using the command specified in the output of the previous command. For example, the command might print something like this:

```bash
---------------------------------------------------
| itksnap | 4.4.0 | 20260117 | visualization | Run:
---------------------------------------------------
/ilpakoz/mri/tutorial/neurocommand/local/fetch_containers.sh itksnap 4.4.0 20260117
```

Run the command under the container description to install that specific version of the container.
After installation, make the Neurocommand modules visible if necessary:

```bash
module use /path/to/neurocommand/local/containers/modules
```

Check which versions are available:

```bash
module avail itksnap
```

Load the desired version:

```bash
module load itksnap/<version>
```

Then run it normally:

```bash
itksnap
```

The same workflow applies to other software already provided by Neurodesk:

```bash
bash containers.sh <package>
bash containers.sh --<package>

module avail <package>
module load <package>/<version>
```

Neurocommand automatically downloads the corresponding Apptainer/Singularity image, generates the command wrappers, and creates the Lmod module. A custom Neurocontainers recipe is only needed when the desired software or version is not already available through Neurodesk.

## Reproducibility

Keep the complete `laynii+ida/` recipe folder under version control together with the code or documentation for the analysis that uses it. The recipe records the base image, package dependencies, LAYNII/IDA download URLs, environment setup, and deployed paths.

For a new build, use a new `YYYYMMDD` build date instead of silently replacing an older container. This keeps separate builds distinguishable while preserving the same `laynii/<version>` workflow through Neurocommand.

## AI use

This repository was written with the assistance of AI, but it is based on the actual procedure I used to install and expose my own LAYNII + LayNii_IDA build through Neurodesk tooling. The commands and workflow described here were tested in practice and worked in my setup, rather than being generated purely from AI suggestions.

## References

- [Neurodesk](https://neurodesk.org/)
- [Neurocontainers documentation](https://neurodesk.org/getting-started/neurocontainers/)
- [Neurocontainers GitHub repository](https://github.com/NeuroDesk/neurocontainers)    
- [Neurocommand Linux/HPC documentation](https://neurodesk.org/getting-started/neurocommand/linux-and-hpc/)
- [Neurocommand GitHub repository](https://github.com/NeuroDesk/neurocommand)
- [LAYNII GitHub repository](https://github.com/layerfMRI/LAYNII)
- [LAYNII releases](https://github.com/layerfMRI/LAYNII/releases)
- [Docker installation documentation](https://docs.docker.com/engine/install/)
- [Docker Linux post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/)
- [Apptainer Quick Start](https://apptainer.org/docs/user/main/quick_start.html)
- [Apptainer installation documentation](https://apptainer.org/docs/admin/main/installation.html)
- [Apptainer build documentation](https://apptainer.org/docs/user/main/build_a_container.html)
- [Lmod documentation](https://lmod.readthedocs.io/en/latest/)
