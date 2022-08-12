# Introduction
This is accompany code and data associated with the paper submission 'SPEL: Software Tool for Porting ELM with OpenACC in a Function Unit Test Framework'.

This software tool builds off of previous work done by Dali Wang and Yao Cindy to
create a robust method for developing the E3SM Land Model (ELM).

# File Structure
./SourceFiles/: folder contains the original ELM source files and the GPU-ready test Modules 
./scripts/: folder contains SPEL Python scripts

Currently, these SPEL Python scripts are used to:
* extract and prepare ELM files to run and compile without MPI and netcdf.
* modify ELM routines to remove modules that cannot or are undesired to run on the GPU.
* Perform automatic OpenACC acceleration using either the __routine__ directive or __parallel loop__ directives.
* Understand code by generating simple call tree and dependency graph of the modules.
* write Fortran routines to generate input/output needed to initialize variables and verify the results and a needed Makefile

__Setup__ : In scripts directory, edit __mod_config.py__ with specific file layout as needed and  __add_input_output.py__ with a list of subroutines to parse (only parent subroutine needs to be listed) and a name for the case. While running with __python3 add_input_output.py__, a directory will be created in __./unit-test/{casename}__ to contain the Functional Unit Test.

A Makefile will automatically be generated for the chosen subroutines to test.  
__elm_initializeMod.F90__ and __main.F90__ will be modified by the scripts to 
use and allocate only the variables that are needed. 

__readMod.F90__, __writeMod.F90__, and __verificationMod.F90__ are generated by the scripts
to create the appropriate I/O and validation functions. __duplicationMod.F90__ is generated to duplicate the same variables as many times as desired at run-time.

__Get Reference Data__: Compile ELM with __writeMod.F90__ and place subroutine __write_vars()__ before subroutine used for the Unit Test.

In addition to the scripts, the __main.F90__ file was created to effectively replace
the lnd_cpl_mct and elm_drv routines and is where all testing is done and configurations should be done.  
Compilation of unit-test only requires NV Fortran compiler, CUDA 10+, and potentially LAPACK.

__make__ command will create the _elmtest.exe_ which is then run with __./elmtest.exe [numSetsOfSites] [clump-pproc]__ where _numSetsOfSites_ controls the number of unique sites used for the reference output to be computed and _clump-pproc_ (optional default = 1) are the number of clumps to have per mpi task. 
Example:
>./elmtest.exe 2
Is used to perform a Unit Test for 2 sets of the 42 Ameriflux sites.

### Script Description
__edit_file.py__ :
>Contains functions that are intended to be used on entire .F90 files rather than
>on specific subroutines.  These functions were created for the purpose of preparing
>ELM files to work with the unit-test.  The user must provide a list of the modules
>and subroutines/functions that need to be removed, and the python functions can then
>comment them out (entire subroutines for some modules) with a '!#py ' comment. 
>If a module is encountered that is not present in the SourceFiles/ directory, 
>The user will be prompted if this module is necessary and add it to the omit list if not 
>and exit if it is.
>
>The file keeps track of any mods used in the file that have not been processed and
>will recursively process them.  Currently, the user must manually keep track of
>what has been processed in a separate file.
>
>There are special comments for BeTR and FATES additions to ELM to allow for easy
>search and replace to enable them.

__analyze_subroutines.py__ :
>Contains a class _Subroutine_ designed to hold all the relevant info and functions
>needed to analyze subroutines for the unit-test and openACC, such as derived types
>and components read/written to and any other subroutines called(and their variable info).
>
>The functions will add !$acc routine info to each subroutine (if not present)
>and do necessary edits to the subroutines required for GPU compilation.  Mostly,
>this means changing subroutine calls containing array bounds.  The python functions
>operate recursively on all subroutines called by the main one.
>
>The __examineLoops__ function is used for further optimization to go beyond the naive
>implementation.  There is also functionality to automatically replace bounds allocations 
>with the compact filter allocation if needed. 
>Currently, if __examineLoops__ is called with the __add_acc__ enabled, it will only 
>accelerate loops that do not have a race condition detected (reduction operation).
>Loops that have that detected are listed in Yellow for the user to examine afterwards.

__add_input_output.py__ :
>This has the __main__ python function that calls the others.
>Also contains a class for derived types that holds the structure of all the
>types used for the unit testing and is primarily needed to create the write and
>read fortran subroutines.  
__DerivedType.py__ 
>Contains DerivedType Class used for processing the ELM data types
__LoopConstructs.py__
>Contains Loop Class used for processing and modifying loops in ELM functions
__errorAnalysis.py__ 
>Functions used for analyszing output from __verficationMod.F90__
__mod_config.py__ 
>Configure location of source files and essential files.
__process_associate.py__ 
>Holds function to obtain global variables in associate list. 
__variable_analysis.py__
>Functions to find global variables that aren't derived types.
__write_routines.py__ 
>Functions that write needed .F90 files (e.g., duplicateMod.F90)

### Notes
+Uses CUDA Fortran to manage memory, so needs to be adapted to work without NVHPC.
