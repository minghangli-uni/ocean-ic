# ocean-ic

Create ocean initial conditions by regridding GODAS or ORAS4 reanalysis to MOM or NEMO grids.

Build status: [![Build Status](https://travis-ci.org/nicjhan/ocean-ic.svg?branch=master)](https://travis-ci.org/nicjhan/ocean-ic)

# Install

This tool is written in Python and depends a few different Python packages. It also depends on
 [ESMF_RegridWeightGen](https://www.earthsystemcog.org/projects/regridweightgen/) program to perform regridding between non-rectilinear grids.

## Download

Download ocean-ic:
```{bash}
$ git clone --recursive https://github.com/nicjhan/ocean-ic.git
```

## Python dependencies

Use Anaconda as described below or an existing Python setup.

1. Download and install [Anaconda](https://www.continuum.io/downloads) for your platform.
2. Setup the Anaconda environment. This will download all the necessary Python packages.
```{bash}
$ cd ocean-ic
$ conda env create -f regridder/regrid.yml
$ source activate regrid
```

## ESMF dependencies

Install ESMF_RegridWeightGen. ESMF releases can be found [here](http://www.earthsystemmodeling.org/download/data/releases.shtml).

There is a bash script regridder/contrib/build_esmf.sh which the testing system uses to build ESMF. This may be useful in addition to the ESMF installation docs.

# Use

Download ORAS4 or GODAS reanalysis dataset, data can be found here:

- GODAS: http://www.esrl.noaa.gov/psd/data/gridded/data.godas.html
- ORAS4: ftp://ftp.icdc.zmaw.de/EASYInit/ORA-S4/monthly_orca1/

For ORAS4 it is also necessary to download the grid definition file at:

- ftp://ftp.icdc.zmaw.de/EASYInit/ORA-S4/orca1_coordinates/

In addition the horizontal and vertical model grid definitions and land-sea mask are also needed. These should be a part of your model installation.

The examples below use preprepared inputs and outputs.

## MOM IC from GODAS

```
$ cd test
$ wget http://s3-ap-southeast-2.amazonaws.com/dp-drop/ocean-ic/test/test_data.tar.gz
$ tar zxvf test_data.tar.gz
$ cd test_data/input
$ ../../../makeic.py GODAS pottmp.2016.nc pottmp.2016.nc pottmp.2016.nc salt.2016.nc \
    MOM ocean_hgrid.nc ocean_vgrid.nc mom_godas_ic.nc --model_mask ocean_mask.nc
$ ncview mom_godas_ic.nc
```

To verify your output:

```
$ mkdir -p test/example_output
$ cd test/example_output
$ wget http://s3-ap-southeast-2.amazonaws.com/dp-drop/ocean-ic/test/example_output/mom_godas_ic.nc
$ ncdiff mom_godas_ic.nc ../test_data/input/mom_godas_ic.nc
```

Notice that since GODAS does not have horizontal and vertical grid definition files we just use the pottmp.nc file.

## NEMO IC from GODAS

There's no need to download the test data if you already did so above.

```
$ cd test
$ wget http://s3-ap-southeast-2.amazonaws.com/dp-drop/ocean-ic/test/test_data.tar.gz
$ tar zxvf test_data.tar.gz
$ cd test_data/input
$ ./makeic.py GODAS pottmp.2016.nc pottmp.2016.nc pottmp.2016.nc salt.2016.nc  \
    NEMO coordinates.nc data_1m_potential_temperature_nomask.nc nemo_godas_ic.nc
$ ncview nemo_godas_ic.nc
```

## MOM IC from ORAS4

```
$ cd test
$ wget http://s3-ap-southeast-2.amazonaws.com/dp-drop/ocean-ic/test/test_data.tar.gz
$ tar zxvf test_data.tar.gz
$ cd test_data/input
$ ./makeic.py ORAS4 coords_T.nc coords_T.nc thetao_oras4_1m_2014_grid_T.nc so_oras4_1m_2014_grid_T.nc \
    MOM ocean_hgrid.nc ocean_vgrid.nc mom_oras4_ic.nc --model_mask ocean_mask.nc
$ ncview mom_oras4_ic.nc
```

## NEMO IC from ORAS4

```
$ cd test
$ wget http://s3-ap-southeast-2.amazonaws.com/dp-drop/ocean-ic/test/test_data.tar.gz
$ tar zxvf test_data.tar.gz
$ cd test_data/input
$ ./makeic.py ORAS4 coords_T.nc coords_T.nc thetao_oras4_1m_2014_grid_T.nc so_oras4_1m_2014_grid_T.nc \
    NEMO coordinates.nc data_1m_potential_temperature_nomask.nc nemo_oras4_ic.nc
$ ncview nemo_oras4_ic.nc
```

Above we're using a NEMO data file, data_1m_potential_temperature_nomask to specify the vertical grid. The levels variable in the NEMO coordinates.nc is incomplete.

## All of the above tests in one go

```
$ python -m pytest
$ ls test/test_data/output/
```

# How to use the output

## MOM

Overwrite the output from the tool to the MOM initial condition file in the INPUT directory:

```{bash}
$ cp mom_oras4_ic.nc INPUT/ocean_temp_salt.res.nc
```

## NEMO

Overwrite the output from the tool to the NEMO initial condition file in the model run directory:

```{bash}
$ cp nemo_oras4_ic.nc data_1m_potential_temperature_nomask.nc
$ cp nemo_oras4_ic.nc data_1m_salinity_nomask.nc
```

Then check the following namelist options:

```{fortran}
&namrun        !   parameters of the run
!-----------------------------------------------------------------------
    ln_rstart   = .false.   !  start from rest (F) or from a restart file (T)

&namtsd    !   data : Temperature  & Salinity
!-----------------------------------------------------------------------
!          !  file name                            ! frequency (hours) ! variable  ! time interp. !  clim  ! 'yearly'/ ! weights  ! rotation ! land/sea mask !
!          !                                       !  (if <0  months)  !   name    !   (logical)  !  (T/F) ! 'monthly' ! filename ! pairing  ! filename      !
    sn_tem  = 'data_1m_potential_temperature_nomask',         -1        ,'votemper' ,    .false.    , .true. , 'yearly'   , ''       ,   ''    ,    ''
    sn_sal  = 'data_1m_salinity_nomask'             ,         -1        ,'vosaline' ,    .false.    , .true. , 'yearly'   , ''       ,   ''    ,    ''
    ln_tsd_init   = .true.    !  Initialisation of ocean T & S with T &S input data (T) or not (F)
    ln_tsd_tradmp = .false.   !  damping of ocean T & S toward T &S input data (T) or not (F)

!-----------------------------------------------------------------------
&namtra_dmp    !   tracer: T & S newtonian damping
!-----------------------------------------------------------------------
    ln_tradmp   =  .false.   !  add a damping termn (T) or not (F)
```

Note that nudging / Newtownian damping (ln_tsd_tradmp and ln_tradmp) has been turned off and there is no time interpolation done on the input. The model output should then contain something like:

```
dta_tsd: deallocte T & S arrays as they are only use to initialize the run
```

If you do wish to do nudging / Newtownian damping then the initial condition must contain a time-series. One way to create this is using the [ocean-nudge](https://github.com/nicjhan/ocean-nudge.git) tool.

# How it works

1. The reanalysis/obs dataset is regridded in the vertical to have the same depth and levels as the model grid. Linear interpolation is used for this. If the model is deeper than the obs then the deepest value is extended.

2. In the case of GODAS since the obs dataset is limited latitudinally it is extended to cover the whole globe. This is done based on nearest neighbours.

3. The obs dataset is then regridded onto the model grid using weights calculated with ESMF_RegridWeightGen. Various regridding schemes are supported includeing distance weighted nearest neighbour, bilinear and conservative.

4. The model land sea mask is applied and initial condition written out.

# Limitations

* When using GODAS reanalysis the values at high latitudes are unphysical due to limited observations.
* Only 'cold-start' initial conditions are created consisting of temperature and salt fields. This means that the model will need to be spun up.

# Example output

## MOM IC temperature field based on ORAS4 reanalysis
![Temp from MOM IC based on ORAS4 reanalysis](https://raw.github.com/nicjhan/ocean-ic/master/doc/MOM_IC_TEMP_ORAS4.png)

## MOM IC salt field based on GODAS reanalysis
![Salt from MOM IC based on GODAS reanalysis](https://raw.github.com/nicjhan/ocean-ic/master/doc/MOM_IC_SALT_GODAS.png)

Note that because GODAS has a limited domain the salt in the Arctic has been filled with a 'representational value', in this case taken from the Bering Strait.

# Developer notes

## Building ESMF_RegridWeightGen

```{bash}
$ export ESMF_DIR=<dir>
$ export ESMF_SHARED_LIB_BUILD=OFF
$ export ESMF_NETCDF="split"
$ export ESMF_NETCDF_INCLUDE=$NETCDF_BASE/include/GNU/
$ export ESMF_NETCDF_LIBPATH=$NETCDF_BASE/lib/GNU
$ export ESMF_NETCDF_LIBS="-lnetcdff -lnetcdf"
$ cd $ESMF_DIR
$ make
$ cd src/apps/ESMF_RegridWeightGen
$ make
```

## Package ocean-ic into a tarball using PyInstaller

```{bash}
$ pyinstaller makeic.spec
```

Upload tarball to s3:

```{bash}
$ s3put -b dp-drop -p $(pwd) ./makeic-0.0.1.tar.gz
$ s3cmd setacl --acl-public --guess-mime-type s3://dp-drop/makeic-0.0.1.tar.gz
```

Download tarball and test data from s3:

```{bash}
$ wget http://s3-ap-southeast-2.amazonaws.com/dp-drop/makeic-0.0.1.tar.gz
$ tar zxvf makeic-0.0.1.tar.gz
$ wget http://s3-ap-southeast-2.amazonaws.com/dp-drop/ic_test_data.tar.gz
$ tar zxvf ic_test_data.tar.gz
```

Try it out:

```{bash}
$ export PATH=$(pwd)/makeic-0.0.1/:$PATH
$ cd ic_test_data
$ makeic --help
$ makeic GODAS pottmp.2016.nc pottmp.2016.nc pottmp.2016.nc salt.2016.nc NEMO coordinates.nc data_1m_potential_temperature_nomask.nc nemo_godas_ic.nc
$ ncview nemo_godas_ic
```

