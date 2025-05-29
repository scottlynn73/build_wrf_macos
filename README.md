# How to build and run WRF on M series Macs
Disclaimer- this is just what I did, it worked for me, I'm not saying it will work for you but if it does, well great!

**Dr Scott Hamilton**,
scott.hamilton@ricardo.com 
28th May 2025

This guide provides the necessary steps to build and then run the WRF and WPS systems on M series Apple Macs. The build was developed and tested on a M2 Macbook Pro, with 8GB RAM. 

The guide offers a recipe for installing the model, followed by a basic run configuration to try in WPS and WRF after you have acquired the relevant files to run a simulation.

# Installing WRF in MacOS
**NOTE: Do all of these steps in the same terminal session!** 

**If you close it you will have to reset the environment variables as they are deliberately transient.**

## Install step 1: install Homebrew
Homebrew is the preferred package manager in MacOS, but it is not installed by default. Install it by running the line below in the Terminal app in MacOS.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Install step 2: install necessary compilers and tools with Homebrew
We need the install compilers that don't interfere with the system tools in MacOS. Homebrew keeps the installed libraries separate so they can be replaced/updated/deleted easily.

Install the main dependencies like this:
```bash
brew install gcc
brew install wget
brew install git
brew install libtool
brew install automake
brew install autoconf
brew install make
brew install m4
brew install java
brew install ksh
brew install tcsh
brew install python@3.10
brew install cmake
brew install xorgproto
brew install xorgrgb
brew install xauth
brew install curl
brew install flex
brew install byacc
brew install bison
brew install gnu-sed
```

## Install step 3: Verify compiler installations
If the step above completed without error, we now check if gcc, g++, and gfortran are correctly installed

```bash
which gcc
which g++
which gfortran-12
```

The command above should show the path to the Homebrew installations of the compilers, e.g:

```bash 
/opt/homebrew/bin/gcc
/opt/homebrew/bin/g++
/opt/homebrew/bin/gfortran-12
```

## Install step 4: create a symbolic link for gfortran
This ensures that when WRF calls for 'gfortran', it uses the installed gfortran-12

```bash
rm -f /opt/homebrew/bin/gfortran
ln -s /opt/homebrew/bin/gfortran-12 /opt/homebrew/bin/gfortran
```

## Install step 5: set compiler environment variables
These tell the build system which compilers to use and with what flags

```bash
export CC=gcc                  # C compiler
export CXX=g++                 # C++ compiler
export FC=gfortran          # Fortran compiler (explicitly set)
export F77=gfortran         # Fortran 77 compiler
export CFLAGS="-fPIC -fPIE -Wno-implicit-function-declaration -Wall"  # C compiler flags
```

## Install step 6: set up the directory structure and environment variables

Now set WRF_FOLDER and DIR to your system specific paths

**in the example below I build in `/Users/sh34/WRF_multistage/` but it can be anywhere on your system**

```bash
# WRF_FOLDER - directory where WRF will be built- set this for your own system
export WRF_FOLDER=/Users/sh34/WRF_multistage/WRF
```

and 

```bash
# LIB_DIR - base directory for all libraries - set this for your own system
export LIB_DIR=/Users/sh34/WRF_multistage/LIBRARIES
mkdir $LIB_DIR # if not done already
```

Now set these compiler and library related environment variables.

```bash
# Various flags needed for compilation and linking
export FCFLAGS=-m64            # 64-bit compilation for Fortran
export FFLAGS=-m64             # 64-bit compilation for Fortran
export JASPERLIB=$LIB_DIR/grib2/lib     # Jasper library location (for GRIB2 support)
export JASPERINC=$LIB_DIR/grib2/include # Jasper include files location
export NCDIR=$DIR/netcdf            # NetCDF directory
export LDFLAGS=-L$LIB_DIR/netcdf/lib    # Linker flags for NetCDF
export CPPFLAGS=-I$LIB_DIR/netcdf/include # Preprocessor flags for NetCDF
export FCFLAGS="-w -fallow-argument-mismatch -O2"  # Fortran compiler flags
export FFLAGS="-w -fallow-argument-mismatch -O2"   # Fortran compiler flags
export PATH=$LIB_DIR/netcdf/bin:$PATH   # Add NetCDF binaries to PATH
export NETCDF=$LIB_DIR/netcdf           # NetCDF location for WRF configure
export PATH=/opt/homebrew/bin:$PATH # Add Homebrew binaries to PATH
export LD_LIBRARY_PATH=${LIB_DIR}/grib2/lib  # Library path for GRIB2
export DYLD_LIBRARY_PATH=${NCDIR}/lib:${LD_LIBRARY_PATH}  # Dynamic library path (macOS specific)
export NFDIR=$DIR/netcdf            # NetCDF Fortran directory
```

## Install step 7: Compiler tests
These tests verify that the compilers work correctly with the WRF requirements. The steps test fixed and free fortran compilation, C compilation, and a mix of all of these in a series of 4 tests. The last check is to run the shell environments we'll be using.

```bash
# Create test directory
mkdir $LIB_DIR/TESTS
cd $LIB_DIR/TESTS

# Download test suite from NCAR
wget https://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/Fortran_C_tests.tar

# expand the archive
tar -xf Fortran_C_tests.tar

# Test 1: Fixed-format Fortran
$FC TEST_1_fortran_only_fixed.f
# execute the build artefact
./a.out 

# Test 2: Free-format Fortran
$FC TEST_2_fortran_only_free.f90
./a.out

# Test 3: C compiler
$CC TEST_3_c_only.c
./a.out

# Test 4: Mixed C and Fortran
$CC -c -m64 TEST_4_fortran+c_c.c
$FC -c -m64 TEST_4_fortran+c_f.f90 
$FC -m64 TEST_4_fortran+c_f.o TEST_4_fortran+c_c.o
./a.out

# Shell tests
./TEST_csh.csh 
./TEST_perl.pl
./TEST_sh.sh
```
If all the compiler tests pass, we're ready to begin building dependencies for WRF, and the model itself after that.

## Install step 8: More environment variables
Set these like so:

```bash

# Set Jasper directories again for clarity
export JASPERLIB=${LIB_DIR}/grib2/lib
export JASPERINC=${LIB_DIR}/grib2/include

# Step 9: Set up MPI (Message Passing Interface) for parallel computing
# Link to the Homebrew MPI installation
export MPICC=/opt/homebrew/bin/mpicc        # MPI C compiler
export MPICXX=/opt/homebrew/bin/mpicxx      # MPI C++ compiler
export MPIF90=/opt/homebrew/bin/mpifort     # MPI Fortran 90 compiler
export MPIF77=/opt/homebrew/bin/mpifort     # MPI Fortran 77 compiler

# MPI library and include paths
export MPILIB=/opt/homebrew/lib
export MPIINC=/opt/homebrew/include
export MPIINC2=/opt/homebrew/include # Redundant but kept for compatibility

# Update PATH with all necessary binary directories
export PATH=${LIB_DIR}/grib2/bin:/opt/homebrew/bin:$NETCDF/bin:$PATH

# Set additional library paths and compiler flags
export LD_LIBRARY_PATH=${LIB_DIR}/grib2/lib:$LD_LIBRARY_PATH
export LDFLAGS="-L${LIB_DIR}/grib2/lib -L${MPILIB} -lmpi ${LDFLAGS}"
export CPPFLAGS="-I${LIB_DIR}/grib2/include -I${MPIINC} -I${MPIINC2}"
```

## Install step 9: Download WRF dependencies

```bash
# HDF5 - hierarchical data format, used by NetCDF4
cd $DIR
wget https://docs.hdfgroup.org/archive/support/ftp/HDF5/releases/hdf5-1.12/hdf5-1.12.0/src/hdf5-1.12.0.tar.gz

# NetCDF-C - Network Common Data Form C library
wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v4.8.1.tar.gz

# NetCDF-Fortran - Fortran interface to NetCDF
wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v4.5.4.tar.gz

# Jasper - JPEG-2000 library for GRIB2 file support
wget https://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/jasper-1.900.1.tar.gz

# libpng - PNG library, dependency for plotting
wget https://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/libpng-1.2.50.tar.gz

# zlib - compression library, dependency for NetCDF and PNG
wget https://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/zlib-1.2.11.tar.gz
```

## Install step 10: build all libraries in the correct order
Each library depends on previously built ones, so order is important

**Zlib**
```bash
# Build zlib first (required by libpng, HDF5, and NetCDF)
cd $LIB_DIR
tar xzvf zlib-1.2.11.tar.gz 
cd zlib-1.2.11 
./configure --prefix=${LIB_DIR}/grib2
make -j 6     # -j 6 uses 6 parallel jobs to speed up compilation
make install -j 6 
```

**libpng**
```bash
# Build libpng (required for some WRF utilities)
cd $LIB_DIR
tar xzvf libpng-1.2.50.tar.gz
cd libpng-1.2.50 
export ac_cv_header_fp_h=no
perl -pi -e 's{^\s*#\s*include\s*<fp\.h>}{# include <math.h>}g' pngconf.h  # Fix for macOS compatibility
./configure --prefix=${LIB_DIR}/grib2 
make -j 6 
make install -j 6
```

**HDF5**
```bash
# Build HDF5 (required by NetCDF4)
mkdir $LIB_DIR/netcdf
cd $LIB_DIR
tar xzvf hdf5-1.12.0.tar.gz
cd hdf5-1.12.0
./configure --prefix=$LIB_DIR/netcdf --with-zlib=$LIB_DIR/grib2 --enable-fortran --enable-shared
make -j 6
make install -j 6
```

**NetCDF-C**
```bash
# Build NetCDF-C
cd $LIB_DIR
tar xzvf v4.8.1.tar.gz
cd netcdf-c-4.8.1
./configure --prefix=$LIB_DIR/netcdf --disable-dap --enable-netcdf4 --enable-hdf5 --enable-shared
make -j 6
make install
```

**NetCDF-F**
```bash
# Build NetCDF-Fortran
cd $LIB_DIR
export LIBS="-lnetcdf -lz"  # Link with NetCDF and zlib
tar xzvf v4.5.4.tar.gz
cd netcdf-fortran-4.5.4
./configure --prefix=$LIB_DIR/netcdf --disable-hdf5 --enable-shared
make -j 6
make install
```

**MPICH**
```bash
# Build MPICH (MPI implementation)
cd $LIB_DIR
export CFLAGS="$CFLAGS -fcommon"  # Needed for newer gcc compatibility
export CXXFLAGS="$CXXFLAGS -fcommon"
wget https://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/mpich-3.0.4.tar.gz
tar xzvf mpich-3.0.4.tar.gz
cd mpich-3.0.4
./configure --prefix=$LIB_DIR/mpich
make -j 6 2>&1
make install
```

**Jasper**
Jasper is tricky to build because it ships with an old config.sub/config.guess that doesn't list the ARM64 architecture, which means the compilers can't tell which type of system we're building against. The steps below acquire updated versions of these definitions to allow compilation to proceed.

```bash
# Build Jasper (for GRIB2 file support)
cd $LIB_DIR
tar xzvf jasper-1.900.1.tar.gz
cd jasper-1.900.1

# Install automake
if ! command -v automake &> /dev/null; then     brew install automake; fi

# Copy required configuration files for macOS build
cp $(brew --prefix)/share/automake-*/config.{guess,sub} acaux
chmod +x acaux/config.guess acaux/config.sub
ls -l acaux/config.*

# Debug automake location
brew list automake && echo "Automake location:" && find $(brew --prefix)/share -name "config.guess" -o -name "config.sub" 2>/dev/null

# Copy exact files from known location
cp /opt/homebrew/Cellar/automake/1.17/share/automake-1.17/config.{guess,sub} acaux
chmod +x acaux/config.guess acaux/config.sub
ls -l acaux/config.*

# Now configure
./configure --prefix=$LIB_DIR/grib2

# And make / make install
make
make install
```

## Install step 11: Build WRF
```bash
cd $LIB_DIR
# Set some additional environment variables for WRF
export MPICH_HOME=$LIB_DIR/mpich
export PATH="$MPICH_HOME/bin:$PATH"
export NETCDF_classic=1
# build with CHEM enabled
export WRF_CHEM=1
export CC=mpicc
export CXX=mpicxx
export FC=mpifort
export F90=$FC
export F77=$FC
export NETCDF=$DIR/netcdf

# Download the WRF model source
cd $WRF_FOLDER
wget -q https://github.com/wrf-model/WRF/releases/download/v4.5.2/v4.5.2.tar.gz
tar -zxf v4.5.2.tar.gz
# Move it to WRF build folder
mv WRFV4.5.2 WRF
cd WRF

# Clean any previous builds (OPTIONAL)
./clean -a

# Configure WRF - this will present an interactive menu
./configure

# !!! IMPORTANT USER INPUT REQUIRED !!!
#!!! IN FIRST MENU SELECT OPTION 17 !!!!! Option 17 is for gfortran compiler with dmpar (distributed memory parallel)

#!!! IN SECOND MENU SELECT OPTION 1 = BASIC NESTING !!!!

# Compile WRF for real-data cases
./compile em_real -j 6  # -j 6 uses 6 parallel processes for faster compile
```

## Install step 12: Build WPS (WRF Preprocessing System)

```bash
cd $WRF_FOLDER
wget -q https://github.com/wrf-model/WPS/archive/refs/tags/v4.5.tar.gz \
tar -zxf v4.5.tar.gz \
mv WPS-4.5 WPS && rm v4.5.tar.gz \
cd WPS

# Configure WPS - this will present an interactive menu
./configure
#!!! SELECT OPTION 17 !!!  # Option 17 matches the compiler choice for WRF

# Compile WPS
./compile
```

## Install step 13: Check the build
In the `$WRF_FOLDER/WPS` directory, you should have .exe files:

* geogrid.exe
* ungrib.exe
* metgrid.exe

In the `$WRF_FOLDER/WRF/main` directory, you should have:
* real.exe
* wrf.exe

## Install step 14: obtain mandatory WRF static input data (geography files etc)
First make the following folders inside your defined `$WRF_FOLDER`

This is the headline folder
`$WRF_FOLDER/SHARED/`

then inside that, make:

`$WRF_FOLDER/SHARED/GRIB_DATA`
`$WRF_FOLDER/SHARED/BACKUPS`
`$WRF_FOLDER/SHARED/GEOG`

In the GEOG folder, download and untar the geography data.

Go to http://www2.mmm.ucar.edu/wrf/users/download/get_sources_wps_geog.html

Download this tar.gz for now- this is the minimal data required. You can download the high res data later at your leisure.

https://www2.mmm.ucar.edu/wrf/src/wps_files/geog_low_res_mandatory.tar.gz

When it is unarchived, make sure the folder containing the files is called `WPS_GEOG_LOW_RES`.  

So our finished geography folder, should be 

`$WRF_FOLDER/SHARED/GEOG/WPS_GEOG_LOW_RES` which should contain all the various geography files.


## Install step 15: obtain optional files to run dust simulations
The dust scheme in WRF-Chem needs 3 more geography files to work.

https://www2.mmm.ucar.edu/wrf/src/wps_files/clayfrac_5m.tar.bz2

https://www2.mmm.ucar.edu/wrf/src/wps_files/erod.tar.bz2

https://www2.mmm.ucar.edu/wrf/src/wps_files/sandfrac_5m.bz2

Download these to the `$WRF_FOLDER/SHARED/GEOG/WPS_GEOG_LOW_RES` folder and untar them- the folder names for these should be `clayfrac_5m`, `erod`, `sandfrac_5m`.

**You will need the paths to the geography data for your WPS namelist**.

## Running WRF step 1: Preparing datasets
To reiterate, make sure the WRF_FOLDER variable is set before proceeding.
```bash
# WRF_FOLDER - directory where WRF will be built- set this for your own system
export WRF_FOLDER=/Users/sh34/WRF_multistage/WRF
```

Make sure you have these folders
`$WRF_FOLDER/SHARED/GRIB_DATA`
`$WRF_FOLDER/SHARED/BACKUPS`

In the BACKUPS folder, prepare the `namelist.wps`, `namelist.input` files, and copy the `Vtable.GFS2` in there (copies are provided in the Appendices at the bottom of this guide).

## Running WRF step 2: Check the WRF build and input data/files
Checklist:
- WRF and WPS have been compiled, using `$WRF_FOLDER` as the default location ✓
.
- `ungrib.exe`, `geogrid.exe`, `metgrid.exe`, `real.exe` and `wrf.exe` in the `$WRF_FOLDER`, in the respective sub directories? (`$WRF_FOLDER/WRF/main` for wrf exe files, `$WRF_FOLDER/WPS` for WPS exe files) ✓
.
- `namelist.wps` and `namelist.input` have been populated and placed in `$WRF_FOLDER/SHARED/BACKUPS`  ✓
.
- `Vtable.GFS2` is also in `$WRF_FOLDER/SHARED/BACKUPS` 
.
- geographical files are downloaded from NCAR and unarchived in `$WRF_FOLDER/SHARED/GEOG/WPS_GEOG_LOW_RES`, including optional files if running dust simulations ✓
.
- GDAS or GFS files have been acquired for the simulation, and placed in `$WRF_FOLDER/SHARED/GRIB_DATA` ✓

**If all these checks are passed, we are ready to run WRF.**


## Running WRF step 3: run WPS
```bash
# make sure you're in the directory with the WPS .exe files
cd $WRF_FOLDER/WPS/

# Edit the namelist.wps for the case beforehand, and link it into the WPS folder
ln -sf  $WRF_FOLDER/SHARED/BACKUPS/namelist.wps

# The Geogrid table ships with WPS, its in the WPS/geogrid folder
# link the Geogrid table so that dust can be modelled, this over writes the default with the CHEM enabled table
ln -sf  $WRF_FOLDER/WPS/geogrid/GEOGRID.TBL.ARW_CHEM $WRF_FOLDER/WPS/geogrid/GEOGRID.TBL
```

Now open your namelist.wps file in a text editor and make sure the `geog_data_path` line is correctly set

You need the full **ABSOLUTE** path for this, it can't read the `$WRF_FOLDER` variable

Mine looks like this:
`geog_data_path = '/Users/sh34/WRF_multistage/SHARED/GEOG/WPS_GEOG_LOW_RES',`

Back in the terminal session:

```bash
cd $WRF_FOLDER/WPS/
# Run geogrid
./geogrid.exe

# Link grib files for the forecast period- likely in a directory called SHARED/GRIB_DATA
# (if GFS files change the last part to gfs*)
./link_grib.csh $WRF_FOLDER/SHARED/GRIB_DATA/gdas*

# Link the Vtable (usually kept in the same directory as the namelists)
ln -sf $WRF_FOLDER/SHARED/BACKUPS/Vtable.GFS2 Vtable

# Run ungrib
./ungrib.exe

#Run metgrid
./metgrid.exe
```

If WPS ran without error, you're ready to run WRF.

## Running WRF step 4: set up and run WRF
Below we setup and run WRF using 6 cores, you can change this to whatever you like on your system by updating the `-np 6` argument in the `mpirun` commands.

In the same terminal session:
```bash
# Go to the WRF run directory (e.g. for my setup below)
cd $WRF_FOLDER/WRF/run

# Link to the WRF namelist
ln -sf $WRF_FOLDER/SHARED/BACKUPS/namelist.input

# edit the namelist.input to set the output directory,
history_outname = "$WRF_FOLDER/WRF/outputs/wrfout_d<domain>_<date>.nc",

# Link to the met_em* outputs from metgrid
ln -sf $WRF_FOLDER/WPS/met_em* .

# Run real.exe on 6 cores- change -np 6 to whatever you like.
mpirun -np 6 ./real.exe

# Run wrf.exe on 6 cores- change -np 6 to whatever you like.
mpirun -np 6 ./wrf.exe
```

## Appendices:
These are example files for a single domain with a basic working configuration.

1. **namelist.wps** (save this to the `$WRF_FOLDER/SHARED/BACKUPS` directory)
```
&share
wrf_core = 'ARW',
max_dom = 1,
start_date = '2024-07-17_00:00:00',
end_date = '2024-07-18_00:00:00',
interval_seconds = 21600,
io_form_geogrid = 2,
/

&geogrid
parent_id = 1,
parent_grid_ratio = 1,
i_parent_start = 1,
j_parent_start = 1,
e_we = 50,
e_sn = 50,
geog_data_res = 'lowres',
dx = 27000,
dy = 27000,
map_proj = 'mercator',
ref_lat = 20.4,
ref_lon = 50.4,
truelat1 = 20.4, 
truelat2 = 20.4,
stand_lon = 50.4,
geog_data_path = '/Users/sh34/WRF_multistage/SHARED/GEOG/WPS_GEOG_LOW_RES',
/

&ungrib
out_format = 'WPS',
prefix = 'FILE',
/

&metgrid
fg_name = 'FILE',
io_form_metgrid = 2,
/
```

2. **namelist.input** (save this to the `$WRF_FOLDER/SHARED/BACKUPS` directory)
```
&time_control
run_days = 1,
run_hours = 0,
run_minutes = 0, 
run_seconds = 0,
start_year = 2024,
start_month = 07,
start_day = 17,
start_hour = 00,
start_minute = 00,
start_second = 00,
end_year = 2024,
end_month = 07,
end_day = 18,
end_hour = 00,
end_minute = 00,
end_second = 00,
interval_seconds = 21600,
history_interval = 60,
frames_per_outfile = 24,
restart = .false.,
restart_interval = 21600,
io_form_history = 2,
io_form_restart = 2,
io_form_input = 2,
io_form_boundary = 2,
debug_level = 0,
history_outname = /Users/sh34/WRF_multistage/WRF/outputs/wrfout_d<domain>_<date>.nc,
nocolons = .false.,
nwp_diagnostics = 1,
force_use_old_data= .true., 
write_hist_at_0h_rst= .true., 
override_restart_timers = .true.,
/

&domains
time_step = 162,
time_step_fract_num      = 0,
time_step_fract_den      = 1,
max_dom                  = 1,
e_we                     = 50,
e_sn                     = 50,
e_vert                   = 40,
num_metgrid_levels       = 34,
num_metgrid_soil_levels  = 4,
dx                       = 27000,
dy                       = 27000,
grid_id                  = 1,
parent_id                = 1,
i_parent_start           = 1,
j_parent_start           = 1,
parent_grid_ratio        = 1,
parent_time_step_ratio   = 1,
feedback                 = 1,
smooth_option            = 0,
/

&physics
mp_physics               = 4,
ra_lw_physics            = 4,
ra_sw_physics            = 4,
radt                     = 27,
sf_sfclay_physics        = 4,
sf_surface_physics       = 2,
bl_pbl_physics           = 4,
bldt                     = 0,
cu_physics               = 1,
cudt                     = 5,
isfflx                   = 1,
ifsnow                   = 1,
icloud                   = 1,
surface_input_source     = 1,
num_soil_layers          = 4,
num_land_cat             = 21,
sf_urban_physics         = 0,
do_radar_ref             = 1,
/

&dynamics
w_damping                = 1,
diff_opt                 = 1,
km_opt                   = 4,
diff_6th_opt             = 0,
diff_6th_factor          = 0.12,
base_temp                = 290.,
damp_opt                 = 0,
zdamp                    = 5000.,
dampcoef                 = 0.2,
khdif                    = 0,
kvdif                    = 0,
non_hydrostatic          = .true.,
moist_adv_opt            = 1,
scalar_adv_opt           = 1,
epssm                    = 0.3,
/

&bdy_control
spec_bdy_width            = 5,
spec_zone                 = 1,
relax_zone                = 4,
specified                 = .true.,
nested                    = .false.,
/

&grib2
/

&namelist_quilt
nio_tasks_per_group      = 2,
nio_groups               = 1,
/

&chem
kemit                               = 1,
chem_opt                            = 401,
bioemdt                             = 0,
photdt                              = 0,
chemdt                              = 10,
io_style_emissions                  = 0,
emiss_opt                           = 3,
emiss_opt_vol                       = 0,
chem_in_opt                         = 0,
phot_opt                            = 0,
gas_drydep_opt                      = 0,
aer_drydep_opt                      = 1,
bio_emiss_opt                       = 0,
dust_opt                            = 3,
dmsemis_opt                         = 0,
seas_opt                            = 0,
gas_bc_opt                          = 0,
gas_ic_opt                          = 0,
aer_bc_opt                          = 1,
aer_ic_opt                          = 1,
gaschem_onoff                       = 0,
aerchem_onoff                       = 1,
wetscav_onoff                       = 0,
cldchem_onoff                       = 0,
vertmix_onoff                       = 1,
chem_conv_tr                        = 0,
conv_tr_wetscav                     = 0,
conv_tr_aqchem                      = 0,
biomass_burn_opt                    = 0,
plumerisefire_frq                   = 30,
have_bcs_chem                       = .false.,
aer_ra_feedback                     = 0,
aer_op_opt                          = 0,
opt_pars_out                        = 0,
diagnostic_chem                     = 0,
/
```

3. **Vtable.GFS2** (save this to the `$WRF_FOLDER/SHARED/BACKUPS` directory)
```
GRIB1| Level| From |  To  | metgrid  | metgrid | metgrid                                 |GRIB2|GRIB2|GRIB2|GRIB2|
Param| Type |Level1|Level2| Name     | Units   | Description                             |Discp|Catgy|Param|Level|
-----+------+------+------+----------+---------+-----------------------------------------+-----------------------+
  11 | 100  |   *  |      | TT       | K       | Temperature                             |  0  |  0  |  0  | 100 |
  33 | 100  |   *  |      | UU       | m s-1   | U                                       |  0  |  2  |  2  | 100 |
  34 | 100  |   *  |      | VV       | m s-1   | V                                       |  0  |  2  |  3  | 100 |
  52 | 100  |   *  |      | RH       | %       | Relative Humidity                       |  0  |  1  |  1  | 100 |
   7 | 100  |   *  |      | HGT      | m       | Height                                  |  0  |  3  |  5  | 100 |
  11 | 105  |   2  |      | TT       | K       | Temperature       at 2 m                |  0  |  0  |  0  | 103 |
  52 | 105  |   2  |      | RH       | %       | Relative Humidity at 2 m                |  0  |  1  |  1  | 103 |
  33 | 105  |  10  |      | UU       | m s-1   | U                 at 10 m               |  0  |  2  |  2  | 103 |
  34 | 105  |  10  |      | VV       | m s-1   | V                 at 10 m               |  0  |  2  |  3  | 103 |
   1 |   1  |   0  |      | PSFC     | Pa      | Surface Pressure                        |  0  |  3  |  0  |   1 |
 130 | 102  |   0  |      | PMSL     | Pa      | Sea-level Pressure                      |  0  |  3  | 192 | 101 |
 144 | 112  |   0  |  10  | SM000010 | fraction| Soil Moist 0-10 cm below grn layer (Up) |  2  |  0  | 192 | 106 |
 144 | 112  |  10  |  40  | SM010040 | fraction| Soil Moist 10-40 cm below grn layer     |  2  |  0  | 192 | 106 |
 144 | 112  |  40  | 100  | SM040100 | fraction| Soil Moist 40-100 cm below grn layer    |  2  |  0  | 192 | 106 |
 144 | 112  | 100  | 200  | SM100200 | fraction| Soil Moist 100-200 cm below gr layer    |  2  |  0  | 192 | 106 |
 144 | 112  |  10  | 200  | SM010200 | fraction| Soil Moist 10-200 cm below gr layer     |  2  |  0  | 192 | 106 |
  11 | 112  |   0  |  10  | ST000010 | K       | T 0-10 cm below ground layer (Upper)    |  0  |  0  |  0  | 106 |
  11 | 112  |  10  |  40  | ST010040 | K       | T 10-40 cm below ground layer (Upper)   |  0  |  0  |  0  | 106 |
  11 | 112  |  40  | 100  | ST040100 | K       | T 40-100 cm below ground layer (Upper)  |  0  |  0  |  0  | 106 |
  11 | 112  | 100  | 200  | ST100200 | K       | T 100-200 cm below ground layer (Bottom)|  0  |  0  |  0  | 106 |
  85 | 112  |   0  |  10  | ST000010 | K       | T 0-10 cm below ground layer (Upper)    |  2  |  0  |  2  | 106 |
  85 | 112  |  10  |  40  | ST010040 | K       | T 10-40 cm below ground layer (Upper)   |  2  |  0  |  2  | 106 |
  85 | 112  |  40  | 100  | ST040100 | K       | T 40-100 cm below ground layer (Upper)  |  2  |  0  |  2  | 106 |
  85 | 112  | 100  | 200  | ST100200 | K       | T 100-200 cm below ground layer (Bottom)|  2  |  0  |  2  | 106 |
  11 | 112  |  10  | 200  | ST010200 | K       | T 10-200 cm below ground layer (Bottom) |  0  |  0  |  0  | 106 |
  91 |   1  |   0  |      | SEAICE   | proprtn | Ice flag                                | 10  |  2  |  0  |   1 |
  81 |   1  |   0  |      | LANDSEA  | proprtn | Land/Sea flag (1=land, 0 or 2=sea)      |  2  |  0  |  0  |   1 |
  81 |   1  |   0  |      | LANDN    | proprtn |                                         |  2  |  0  | 218 |   1 |
   7 |   1  |   0  |      | SOILHGT  | m       | Terrain field of source analysis        |  0  |  3  |  5  |   1 |
  11 |   1  |   0  |      | SKINTEMP | K       | Skin temperature                        |  0  |  0  |  0  |   1 |
  65 |   1  |   0  |      | SNOW     | kg m-2  | Water equivalent snow depth             |  0  |  1  | 13  |   1 |
     |   1  |   0  |      | SNOWH    | m       | Physical Snow Depth                     |  0  |  1  |     |   1 |
  33 |   6  |   0  |      | UMAXW    | m s-1   | U                 at max wind           |  0  |  2  |  2  |   6 |
  34 |   6  |   0  |      | VMAXW    | m s-1   | V                 at max wind           |  0  |  2  |  3  |   6 |
   2 |   6  |   0  |      | PMAXW    | Pa      | Pressure of max wind level              |  0  |  3  |  0  |   6 |
     |   6  |   0  |      | PMAXWNN  | Pa      | PMAXW, used for nearest neighbor interp |  0  |  3  |  0  |   6 |
   2 |   6  |   0  |      | TMAXW    | K       | Temperature at max wind level           |  0  |  0  |  0  |   6 |
   7 |   6  |   0  |      | HGTMAXW  | m       | Height of max wind level                |  0  |  3  |  5  |   6 |
  33 |   7  |   0  |      | UTROP    | m s-1   | U                 at tropopause         |  0  |  2  |  2  |   7 |
  34 |   7  |   0  |      | VTROP    | m s-1   | V                 at tropopause         |  0  |  2  |  3  |   7 |
   2 |   7  |   0  |      | PTROP    | Pa      | Pressure of tropopause                  |  0  |  3  |  0  |   7 |
     |   7  |   0  |      | PTROPNN  | Pa      | PTROP, used for nearest neighbor interp |  0  |  3  |  0  |   7 |
   2 |   7  |   0  |      | TTROP    | K       | Temperature at tropopause               |  0  |  0  |  0  |   7 |
   7 |   7  |   0  |      | HGTTROP  | m       | Height of tropopause                    |  0  |  3  |  5  |   7 |
-----+------+------+------+----------+---------+-----------------------------------------+-----------------------+
#
```

