# Create Scripts 

The `create_fates_run.sh` script is a set of instructions specifying the configuration of the FATES run.
An example script is found in the fates-tutorial repository under run_scripts (outside the container) or scripts (insdie the container).
To run the create script and launch a FATES simualtion, from inside the elm-fates container, 
change directory to the scripts directory and type: 

```
./create_elm-fates-singlepoint-1pft_bci_invinit.sh
```

In the page below we explain what key parts of the script do. 
Open `create_elm-fates-singlepoint-1pft_bci_invinit.sh` 
with your text editor of choice and then see below for details. 

```
#!/bin/sh
# =======================================================================================
# =======================================================================================
export CIME_MODEL=e3sm
export COMPSET=2000_DATM%QIA_ELM%BGC-FATES_SICE_SOCN_SROF_SGLC_SWAV
export RES=ELM_USRDAT
export MACH=docker                                             # Name your machine
export COMPILER=gnu                                            # Name your compiler
export SITE=bci                                                # Name your site

```

The first line indicates that this is a shell script. 
Next is a block of  code indicating some model configurations. 
The CIME model is the infrastructure that allows different model
components to communicate, for example, the land model and the atmospheric model. 
The COMPSET specifies which components are active. Here we will run land only simulations.
RES is the model resolution. In a global run this would be the size of each model grid cell.
Here we specify that we are running FATES as a single point. 
MACH is the machine being used. In this case we specify docker since we are running FATES
via docker. 
FATES and the host land model code is written in Fortran which is a compiled language. 
The COMPILER option specifies which compiler to use. Here we use gnu as this is 
installed
in the docker environment. 
Finally, SITE is where we tell FATES which site we are going to run the simulation, in this 
case BCI. 

We make some variables to use in the  rest of the script.
```
export TAG=fates-tutorial-${SITE}-inventory_init  # give your run a name
export CASE_ROOT=/output/${SITE}                  # where in scratch should the run go?
export PARAM_FILES=/paramfiles                    # FATES parameter file location  
```


The next code block specifies where the surface and domain files needed to run FATES are 
located. 

```
# surface and domain files
export SITE_BASE_DIR=/sitedata
export ELM_USRDAT_DOMAIN=domain_${SITE}_fates_tutorial.nc
export ELM_USRDAT_SURDAT=surfdata_${SITE}_fates_tutorial.nc
export ELM_SURFDAT_DIR=${SITE_BASE_DIR}/${SITE}
export ELM_DOMAIN_DIR=${SITE_BASE_DIR}/${SITE}
export DIN_LOC_ROOT_FORCE=${SITE_BASE_DIR}
```

The surface dataset contains information on the soil properties and the fraction of the grid cell
that is natural vegetation. The domain file contains information on  the lat and lon of the site
and what fraction of the grid cell is active i.e. running FATES. DIN_LOC_ROOT_FORCE forces
the model to use climate driving data located within the site directory. 

Climate data is stored in the sitedata directory. For simulations at new sites we will use GSWP3 reanalysis data
that has been subset to only the grid cell of each site. Since this data
is only available from the years 2003 to 2014 we tell the model to keep cycling the climate data
so that we can run longer simulations. For BCI, we have meterological data from a tower at the site, and we can use this
data which is available between 2003 and 2016.  

```
# climate data will recycle data between these years
export DATM_START=2003
export DATM_STOP=2016
```

The next code block changes into the directory where the scripts needed to create a FATES case
are located. We make a case name by appending today's date to the tag defined above. 

```
# DEPENDENT PATHS AND VARIABLES (USER MIGHT CHANGE THESE..)
# =======================================================================================
export SOURCE_DIR=/E3SM/cime/scripts  # change to the path where your E3SM/cime/sripts is
cd ${SOURCE_DIR}
export CASE_NAME=${CASE_ROOT}/${TAG}.`date +"%Y-%m-%d"`
```

Within the E3SM/cime/scripts directory is the create_newcase script which we call with 
arguments defined above to set up the FATES simualtion. When the case has been created we 
change into the case directory. 

```
# REMOVE EXISTING CASE IF PRESENT
rm -r ${CASE_NAME}

# CREATE THE CASE
./create_newcase --case=${CASE_NAME} --res=${RES} --compset=${COMPSET} --mach=${MACH} --compiler=${COMPILER} --project=${PROJECT}

cd ${CASE_NAME}

```

The next two blocks of code can be ignored as they relate to the inner workings of the host
land model. 

The next chunk of code relevant for the tutorial is where we specify run type preferences. 

```
# SPECIFY RUN TYPE PREFERENCES (USERS WILL CHANGE THESE)
# =================================================================================

./xmlchange DEBUG=FALSE
./xmlchange STOP_N=10 # how many years should the simulation run
./xmlchange RUN_STARTDATE='1900-01-01'
./xmlchange STOP_OPTION=nyears   
./xmlchange REST_N=10 # how often to make restart files
./xmlchange RESUBMIT=0 # how many resubmits

./xmlchange DATM_CLMNCEP_YR_START=${DATM_START}
./xmlchange DATM_CLMNCEP_YR_END=${DATM_STOP}

``` 

Here we set DEBUG to FALSE as (hopefully) everything should  be running smoothly
in the tutorial. If you are ever encountering errors when trying to get FATES
to compile or run, then setting DEBUG to true might mean you get more
informative error messages. 
STOP_N is how many time steps to run the simulation, with the unit of time
specified by STOP_OPTION - in this case years. 
REST_N specifies how often to write restart files -  these are a snapshot of 
the simulation containing all necessary information on the ecosystem to be
able to restart the run from that point.
RUN_STARTDATE is fairly arbitrary in this case - we chose 1900 for no 
particular reason. 
RESUBMIT is set to 0 here. In very long simulations (decades or centuries)
it can help to run FATES in smaller chunks by setting STOP_N to 
a factor of the total length of the simulation and increasing RESUBMIT such that
RESUBMIT x STOP_N equals the total simulation length. 

The next block of code specifies where to build
the case and where to save model outputs. 

```
# MACHINE SPECIFIC, AND/OR USER PREFERENCE CHANGES (USERS WILL CHANGE THESE)
# =================================================================================

./xmlchange GMAKE=make
./xmlchange RUNDIR=${CASE_NAME}/run                 
./xmlchange EXEROOT=${CASE_NAME}/bld

```

The next bit is where we edit the user_nl_elm file. The user_nl_elm file specifies 
namelist options, such as which optional modules to turn on, and points to the parameter file and inventory data. 
It also defines which history variables to output. 

```
# point to your parameter file
# add any history variables you want
cat >> user_nl_elm <<EOF
fsurdat = '${ELM_SURFDAT_DIR}/${ELM_USRDAT_SURDAT}'
fates_paramfile='${PARAM_FILES}/fates_params_default-1pft.nc'
use_fates=.true.
use_fates_inventory_init = .true.
fates_inventory_ctrl_filename = '/inventorydata/inventory_ctrl/fates_${SITE}_inventory_ctrl'
fluh_timeseries=''
hist_fincl1=
'FATES_VEGC_PF', 'FATES_VEGC_ABOVEGROUND',
'FATES_NPLANT_SZ', 'FATES_CROWNAREA_PF',
'FATES_LAI', 'FATES_BASALAREA_SZPF', 'FATES_CA_WEIGHTED_HEIGHT', 'Z0MG',
'FATES_MORTALITY_CSTARV_CFLUX_PF', 'FATES_MORTALITY_CFLUX_PF',
'FATES_MORTALITY_HYDRO_CFLUX_PF', 'FATES_MORTALITY_BACKGROUND_SZPF',
'FATES_MORTALITY_HYDRAULIC_SZPF', 'FATES_MORTALITY_CSTARV_SZPF',
'FATES_MORTALITY_IMPACT_SZPF', 'FATES_MORTALITY_TERMINATION_SZPF',
'FATES_MORTALITY_FREEZING_SZPF', 'FATES_MORTALITY_CANOPY_SZPF',
'FATES_MORTALITY_USTORY_SZPF', 'FATES_NPLANT_SZPF',
'FATES_NPLANT_CANOPY_SZPF', 'FATES_NPLANT_USTORY_SZPF', 
'FATES_NPP_PF', 'FATES_GPP_PF', 'FATES_NEP', 'FATES_FIRE_CLOSS',
'FATES_ABOVEGROUND_PROD_SZPF', 'FATES_ABOVEGROUND_MORT_SZPF',
'FATES_NPLANT_CANOPY_SZ', 'FATES_NPLANT_USTORY_SZ',
'FATES_DDBH_CANOPY_SZ', 'FATES_DDBH_USTORY_SZ',
'FATES_MORTALITY_CANOPY_SZ', 'FATES_MORTALITY_USTORY_SZ'
EOF
```

More information on history outputs is in the 'FATES History Outputs' section of the Read the Docs. 

```
use_fates_inventory_init  = .true. 
fates_inventory_ctrl_filename = '/inventorydata/inventory_ctrl/fates_${SITE}_inventory_ctrl'
```

These two lines tell FATES that we want to start the run by reading in the forest size structure from inventory data,
which is detailed in the control file. 
hist_fincl1 is followed by a list of  the history files to output. Some files are output by default but larger files we
have to specify here.  These are the outputs  that we will use to plot results of the simulation. Think carefully about these
because it is a real pain to wait days or hours for a simulation to finish and then realize it didn't include the
history variables you need. 

In the next bit of code we tell the model to recycle the three climate data streams (solar, precipitation and wind)
 so that we can run simulations longer than the length of the climate  time series. 

```
cat >> user_nl_datm <<EOF
taxmode = "cycle", "cycle", "cycle"
EOF
```

We then set up the case and check the  namelist options with the two following commands

```
./case.setup
./preview_namelists
```

We copy the climate data information to the user_datm.streams.txt.CLM1PFT.ELM_USRDAT so that 
the simulation will know where to look for it and modify it using sed so it is in the correct format. 

```
cp  run/datm.streams.txt.CLM1PT.ELM_USRDAT user_datm.streams.txt.CLM1PT.ELM_USRDAT
`sed -i '/FLDS/d' user_datm.streams.txt.CLM1PT.ELM_USRDAT`
```

And that is it! We are ready to build the run and launch it with the following two commands: 

```
./case.build --skip-provenance-check  # skipping provenance avoids calling git (for this tutorial only)
./case.submit
```

To kick off the simulation we run the create script from the command line. In terminal, 
from inside the scripts directory (within the container) run the following: 

```
./create_elm-fatese-singlepoint-1pft_bci_invinit.sh
```

A lot of text will go zooming by as the code is compiled and the case is built. 
To see if it has  worked you can change directory to the case and watch the
.elm.h0. files appear.

