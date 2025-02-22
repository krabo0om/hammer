Sky130 Technology Library
=========================
Hammer supports the Skywater 130nm Technology process. The [SkyWater Open Source PDK](https://skywater-pdk.readthedocs.io/) is a collaboration between Google and SkyWater Technology Foundry to provide a fully open source Process Design Kit (PDK) and related resources, which can be used to create manufacturable designs at SkyWater’s facility.


PDK Setup
---------

All Sky130 PDK files required for the Hammer VLSI flow can now be [installed via conda](https://anaconda.org/litex-hub/open_pdks.sky130a) by running:

```shell
conda install -c litex-hub open_pdks.sky130a
```

We recommend using the conda install, but the [manual PDK setup is documented below](#manual-pdk-setup) (although it may be outdated).

Now in your Hammer YAML configs, point to the location of this install:

```yaml
technology.sky130.sky130A: "/path/to/conda/env/share/pdk/sky130A"
```


SRAM Macros
-----------

If you are using SRAMs in your design, such as for the [Chipyard Sky130 tutorial](https://chipyard.readthedocs.io/en/stable/VLSI/Sky130-Commercial-Tutorial.html),
you will require a set of SRAM macros.
The Sky130 PDK did not come with it's own SRAM macros, so these have been generated by various third-party contributors.
Hammer is compatible with Sky130 SRAM macros generated by 
[Sram22](https://github.com/rahulk29/sram22_sky130_macros)
and [OpenRAM](https://github.com/efabless/sky130_sram_macros),
but starting with Hammer v1.0.2 only Sram22 macros will be supported.
To obtain these macros, clone their github repo:

```shell
git clone https://github.com/rahulk29/sram22_sky130_macros
```

Then set the respective Hammer YAML key:

```yaml
technology.sky130.sram22_sky130_macros: "/path/to/sram22_sky130_macros"
```

Note that the various configurations of the SRAMs available are encoded in the file ``sram-cache.json``.
To modify this file to include different configurations, or switch to using the OpenRAM SRAMs,
navigate to ``./extra/[sram22|openram]`` and run the script ``./sram-cache-gen.py`` for usage information.

NDA Files
---------
The NDA version of the Sky130 PDK is only required for Siemens Calibre to perform DRC/LVS signoff with the commercial VLSI flow.
It is NOT REQUIRED for the remaining commercial flow, as well as the open-source tool flow.
Therefore this NDA PDK is not used to generate a GDS.

If you have access to the NDA repo, you should add this path to your Hammer YAML configs:

```yaml
technology.sky130.sky130_nda: "/path/to/skywater-src-nda"
```

We use the Calibre decks in the ``s8`` PDK, version ``V2.0.1``, 
see [here for the DRC deck path](https://github.com/ucb-bar/hammer/blob/612b4b662a774b1cab5cf25e8f41c6a771388e47/hammer/technology/sky130/sky130.tech.json#L16) 
and [here for the LVS deck path](https://github.com/ucb-bar/hammer/blob/612b4b662a774b1cab5cf25e8f41c6a771388e47/hammer/technology/sky130/sky130.tech.json#L24).



Resources
---------
The good thing about this process being open-source is that most questions about the process are answerable through a google search. 
The tradeoff is that the documentation is a bit of a mess, and is currently scattered over a few pages and Github repos. 
We try to summarize these below.

SRAMs:

* [Sram22 pre-compiled macros](https://github.com/rahulk29/sram22_sky130_macros)

  * Various pre-compiled sizes of SRAM macros to support Hammer Sky130 flow

* [Sram22](https://github.com/rahulk29/sram22)

  * Open-source SRAM generator
  * Currently only supports the Sky130 process
  * Very much under development, and some parts currently require commercial tools (for LEF and LIB generation)

* [OpenRAM pre-compiled macros](https://github.com/efabless/sky130_sram_macros)

  * Precompiled sizes are 1kbytes, 2kbytes and 4kbytes

* [OpenRAM](https://github.com/VLSIDA/OpenRAM/)

  * Open-source static random access memory (SRAM) compiler


Git repos:

* [SkyWater Open Source PDK](https://github.com/google/skywater-pdk/)

  * Git repo of the main Skywater 130nm files

* [Open-PDKs](https://github.com/RTimothyEdwards/open_pdks/)

  * Git repo of Open-PDKs tool that compiles the Sky130 PDK

* [Git repos on foss-eda-tools](https://foss-eda-tools.googlesource.com/)

  * Additional useful repos, such as Berkeley Analog Generator (BAG) setup

Documentation:

* [SkyWater SKY130 PDK's documentation](https://skywater-pdk.readthedocs.io)

  * Main documentation site for the PDK

* [Join the SkyWater PDK Slack Channel](https://join.skywater.tools/)

  * By far the best way to have questions about the process answered, with 80+ channels for different topics

* [Skywater130 Standard Cell and Primitives Overview](http://diychip.org/sky130/)

  * Additional useful documentation for the PDK


## Manual PDK Setup

### PDK Structure

OpenLANE expects a certain file structure for the Sky130 PDK. 
We recommend adhering to this file structure for Hammer as well.
All the files reside in a root folder (named something like `skywater` or `sky130`).
The environment variable `$PDK_ROOT` should be set to this folder's path:

```shell
export PDK_ROOT=<path-to-skywater-directory>
```

`$PDK_ROOT` contains the following:

* `skywater-pdk`

  * Original PDK source files

* `open_pdks`

  * install of Open-PDKs tool

* `share/pdk/sky130A`

  * output files from Open-PDKs compilation process


### Prerequisites for PDK Setup

* [Magic](http://opencircuitdesign.com/magic/)

  * required for `open_pdks` file generation
  * tricky to install, closely follow the directions on the `Install` page of the website
  
    * as the directions indicate, you will likely need to manually specify the location of the Tcl/Tk package installation using `--with-tcl` and `--with-tk`
 
 If using conda, these installs alone caused the Magic install to work:

```shell
conda install -c intel tcl 
conda install -c anaconda tk 
conda install -c anaconda libglu
```

### PDK Install

In `$PDK_ROOT`, clone the skywater-pdk repo and generate the liberty files for each library:

```shell
git clone https://github.com/google/skywater-pdk.git
cd skywater-pdk
# Expect a large download! ~7GB at time of writing.
git submodule init libraries/*/latest
git submodule update
# Regenerate liberty files
make timing
```

Again in `$PDK_ROOT`, clone the open_pdks repo and run the install process to generate the `sky130A` directory:

```shell
git clone https://github.com/RTimothyEdwards/open_pdks.git
cd open_pdks
./configure \
    --enable-sky130-pdk=$PDK_ROOT/skywater-pdk/libraries \
    --prefix=$PDK_ROOT \
make
make install
```

This generates all the Sky130 PDK files and installs them to `$PDK_ROOT/share/pdk/sky130A`




