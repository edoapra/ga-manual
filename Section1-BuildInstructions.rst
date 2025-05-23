

Building GlobalArrays
---------------------

autotools build
^^^^^^^^^^^^^^^

Please refer to the INSTALL file for generic build instructions. That is
a good place to start if you are new to using “configure; make; make
install” types of builds. The following will cover platform-specific
considerations as well as the various optional features of GA.
Customizations to the GA build via the configure script are discussed
next.

Configuration Options
~~~~~~~~~~~~~~~~~~~~~

There are many options available when configuring GA. Although configure
can be safely run within this distributions’ root folder, we recommend
performing an out-of-source (aka VPATH) build. This will cleanly
separate the generated Makefiles and compiled object files and libraries
from the source code. This will allow, for example, one build using MPI
two-sided versus another build using OpenIB for the communication layer
to use the same source tree e.g.:

::

   mkdir bld_mpi_ts && cd bld_mpi_ts && ../configure
   mkdir bld_mpi_openib  && cd bld_mpi_openib  && ../configure --with-openib

Regardless of your choice to perform a VPATH build, the following should
hopefully elucidate the myriad options to configure. Only the options
requiring additional details are documented here. ``./configure --help``
will certainly list more options in addition to limited documentation.

Selecting the Underlying One-Sided Runtime
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This software contains a number of communication runtime implementations
which directly use MPI instead of a native communication library, e.g.,
OpenIB verbs, Cray DMAPP. The basis of all of our MPI-based ports is the
use of the MPI two-sided primitives (MPI_Send, MPI_Recv) to implement
our ComEx/ARMCI one-sided protocols. The primary benefit of these ports
is that Global Arrays and its user applications will now run on any
platform where MPI is supported.

The recommended port is MPI-1 with progress ranks ``--with-mpi-pr``.
However, there are some caveats which must be mentioned in order to use
the new MPI ports.

**How to Use Progress Ranks**

Your application code must not rely on MPI_COMM_WORLD directly. Instead,
you must duplicate the MPI communicator that the GA library returns to
you in place of any world communicator. Example code follows:

Fortran77:

.. code:: fortranfixed

         program main
         implicit none
   #include “mpi.fh"
   #include "global.fh"
   #include "ga-mpi.fh"
         integer comm
         integer ierr
         call mpi_init(ierr)
         call ga_initialize()
         call ga_mpi_comm(comm)
   ! use the returned comm as ususal
         call ga_terminate()
         call mpi_finalize(ierr)
         end

C/C++:

.. code:: c

   #include <mpi.h>
   #include "ga.h"
   #include "ga-mpi.h"
   int main(int argc, char **argv) {
       MPI_Comm comm;
       MPI_Init(&argc,&argv);
       GA_Initialize();
       comm = GA_MPI_Comm();
       GA_Terminate();
       MPI_Finalize();
       return 0;
   }

**How to Use Progress Threads**

This port uses ``MPI_Init_thread()`` internally with a threading level of
``MPI_THREAD_MULTIPLE``. It will create one progress thread per compute
node. It is advised to undersubscribe your compute nodes by one core.
Your application code can remain unchanged unless you call MPI_Init() in
your application code, in which case GA will detect the lower MPI
threading level and abort with an error.

**How to Use the New Default Two Sided Port**

The MPI two-sided port is fully compatible with the MPI-1 standard.
However, your application code will require additional ``GA_Sync()`` calls
prior to and after any MPI function calls. This effectively splits user
application code into blocks/epochs/phases of MPI code and GA code. Not
doing so will likely cause your application to hang since our two sided
port can only make communication progress inside of a GA function call.

Any application code which only makes GA function calls can remain
unchanged.

Full List of Runtimes
~~~~~~~~~~~~~~~~~~~~~                     

::

   --with-armci[=ARG]      select armci network as external; path to external
                             ARMCI library
   --with-cray-shmem[=ARG] select armci network as Cray XT shmem
   --with-dmapp[=ARG]      select armci network as (Comex) Cray DMAPP
   --with-gemini[=ARG]     select armci network as Cray XE Gemini using
                             libonesided
   --with-lapi[=ARG]       select armci network as IBM LAPI
   --with-mpi-mt[=ARG]     select armci network as (Comex) MPI-2
                             multi-threading
   --with-mpi-pt[=ARG]     select armci network as (Comex) MPI-2
                             multi-threading with progress thread
   --with-mpi-pr[=ARG]     select armci network as (Comex) MPI-1 two-sided with
                             progress rank
   --with-mpi-spawn[=ARG]  select armci network as MPI-2 dynamic process mgmt
   --with-mpi-ts[=ARG]     select armci network as (Comex) MPI-1 two-sided
   --with-mpi3[=ARG]       select armci network as (Comex) MPI-3 one-sided
   --with-ofa[=ARG]        select armci network as (Comex) Infiniband OpenIB
   --with-ofi[=ARG]        select armci network as (Comex) OFI
   --with-openib[=ARG]     select armci network as Infiniband OpenIB
   --with-portals4[=ARG]   select armci network as (Comex) Portals4
   --with-portals[=ARG]    select armci network as Cray XT portals
   --with-sockets[=ARG]    select armci network as Ethernet TCP/IP

Other Options
~~~~~~~~~~~~~

::

   --disable-f77           Disable Fortran code. This used to be the old
                           GA_C_CORE or NOFORT environment variables which
                           enabled the C++ bindings. However, it is severely
                           broken. There are certain cases where Fortran code is
                           required but this will not inhibit the building of the
                           C++ bindings.  In the future we may be able to
                           eliminate the need for the Fortran compiler/linker.
                           Use at your own risk (of missing symbols at link-time.)
   --enable-cxx            Build C++ interface. This will require the C++ linker
                           to locate the Fortran libraries (handled
                           automatically) but user C++ code will require the same
                           considerations (C++ linker, Fortran libraries.)
   --disable-opt           Don't use hard-coded optimization flags. GA is a
                           highly-optimized piece of software. There are certain
                           optimization levels or flags that are known to break
                           the software. If you experience mysterious faults,
                           consider rebuilding without optimization by using this
                           option.
   --enable-sysv           Enable System V Shared Memory.
   --enable-peigs          Enable Parallel Eigensystem Solver interface. This
                           will build the stubs required to call into the peigs
                           library (external).
   --enable-checkpoint     Enable checkpointing.  Untested.  For use with old
                           X-based visualization tool.
   --enable-profile        Enable profiling. Not sure what this does, sorry.
   --enable-trace          Enable tracing. Not sure what this does, sorry.
   --enable-underscoring   Force single underscore for all external Fortran
                           symbols. Usually, configure is able to detect the name
                           mangling scheme of the detected Fortran compiler and
                           will default to using what is detected. This includes
                           any variation of zero, one, or two underscores or
                           whether UPPERCASE or lowercase symbols are used. If
                           you want to force a single underscore which was the
                           default of older GA builds, use this option.
                           Otherwise, you can use the FFLAGS environment variable
                           to override the Fortran compiler's or platform's
                           defaults e.g. configure FFLAGS=-fno-underscoring.
   --enable-i4             Use 4 bytes for Fortran INTEGER size. Otherwise, the
                           default INTEGER size is set to the results of the C
                           sizeof(void*) operator.
   --enable-i8             Use 8 bytes for Fortran INTEGER size. Otherwise, the
                           default INTEGER size is set to the results of the C
                           sizeof(void*) operator.
   --enable-shared         Build shared libraries [default=no]. Useful, for
                           example, if you plan on wrapping GA with an
                           interpreted language such as Python. Otherwise, some
                           systems only support static libraries (or vice versa)
                           but static libraries are the default.

For most of the external software packages an optional argument is
allowed (represented as ARG below.) **ARG can be omitted** or can be one
or more whitespace-separated directories, linker or preprocessor
directives. For example:

::
   
   --with-mpi="/path/to/mpi -lmylib -I/mydir" --with-mpi=/path/to/mpi/base --with-mpi=-lmpich

The messaging libraries supported include MPI, TCGMSG, and TCGMSG over
MPI. If you omit their respective ``--with-`` option, MPI is the
default. GA can be built to work with MPI or TCGMSG. Since the TCGMSG
package is small (comparing to portable MPI implementations) and
compiles fast, it is still bundled with the GA package.

::

   --with-mpi=ARG          Select MPI as the messaging library (default). If you
                           omit ARG, we attempt to locate the MPI compiler
                           wrappers. If you supply anything for ARG, we will
                           parse ARG as indicated above.
   --with-tcgmsg           Select TCGMSG as the messaging library; if
                           --with-mpi is also specified then TCGMSG over MPI is
                           used.
   --with-blas=ARG         Use external BLAS library; if not found, an
                           internal BLAS is built
   --with-blas4=ARG        Use external BLAS library compiled with
                           sizeof(INTEGER)==4
   --with-blas8=ARG        Use external BLAS library compiled with
                           sizeof(INTEGER)==8
   --with-lapack=ARG       Use external LAPACK library. If not found, an internal
                           one is built.
   --with-scalapack=ARG    Use external ScaLAPACK library compiled with
                           sizeof(INTEGER)==4
   --with-scalapack8=ARG   Use external ScaLAPACK library compiled with
                           sizeof(INTEGER)==8

There are some influential environment variables as documented in
``configure --help``, however there are a few that are special to GA.

::

   - F77_INT_FLAG
     Fortran compiler flag to set the default INTEGER size. We know about certain
     Fortran flags that set the default INTEGER size, but there will certainly be
     some new (or old) ones that we don't know about. If the configure test to
     determine the correct flag fails, please try setting this variable and
     rerunning configure.

   - F2C_HIDDEN_STRING_LENGTH_AFTER_ARGS
     If cross compiling, set to either "yes" (default) or "no" (after string).
     For compatibility between Fortran and C, a Fortran subroutine written in C
     that takes a character string must take an additional argument (one per
     character string) indicating the length of the string. This 'hidden'
     argument appears either immediately after the string in the argument list
     or after all other arguments to the function. This is compiler dependent. We
     attempt to detect this behavior automatically, but in the case of
     cross-compiled systems it may be necessary to specify the less usual after
     string convention the gaf2c/testarg program crashes.

Special Notes for BLAS
~~~~~~~~~~~~~~~~~~~~~~

BLAS, being a Fortran library, can be compiled with a default INTEGER
size of 4 or a promoted INTEGER size of 8. Experience has shown us that
most of the time the default size of INTEGER used is 4. In some cases,
however, you may have an external BLAS library which is using 8-byte
INTEGERs. In order to correctly interface with an external BLAS library,
GA must know the size of INTEGER used by the BLAS library.

configure has the following BLAS-related options: ``--with-blas``,
``--with-blas4``, and ``--with-blas8``. The latter two will force the
INTEGER size to 4- or 8-bytes, respectively. The first option,
``--with-blas``, defaults to 4-byte INTEGERS. 
As documented in the ACML manual, if the path to
the library has ``_int64`` then 8-byte INTEGERs are used. As documented
in the MKL manual, if the library is ``ilp64``, then 8-byte INTEGERs are
used.

You may always override ``--with-blas`` by specifying the INTEGER size
using one of the two more specific options.

Cross-Compilation Issues
~~~~~~~~~~~~~~~~~~~~~~~~

Certain platforms cross-compile from a login node for a compute node, or
one might choose to cross-compile for other reasons. Cross-compiling
requires the use of the ``--host`` option to configure which indicates
to configure that certain run-time tests should not be executed. See
INSTALL for details on use of the ``--host`` option.

Two of our target platforms are known to require cross-compilation, Cray
XT and IBM Blue Gene.

Cray XT
~~~~~~~

It has been noted that configure still succeeds without the use of the
-host flag. If you experience problems without -host, we recommend

::

   configure --host=x86_64-unknown-linux-gnu

And if that doesn’t work (cross-compilation is not detected) you must
then *force* cross-compilation using both ``--host`` and ``--build``
together:

::

   configure --host=x86_64-unknown-linux-gnu --build=x86_64-unknown-linux-gnu

Alternatively, you can just tell configure directly.

::

   configure cross_compiling=yes

Compiler Selection
~~~~~~~~~~~~~~~~~~

Unless otherwise noted you can try to overwrite the default compiler
names detected by configure by defining F77, CC, and CXX for Fortran
(77), C, and C++ compilers, respectively. Or when using the MPI
compilers MPIF77, MPICC, and MPICXX for MPI Fortran (77), C, and C++
compilers, respectively:

::

   configure F77=f90 CC=gcc
   configure MPIF77=mpif90 MPICC=mpicc

Although you can change the compiler at make-time it will likely fail.
Many platform-specific compiler flags are detected at configure-time
based on the compiler selection. If changing compilers, we recommend
rerunning configure as above.

After Configuration
~~~~~~~~~~~~~~~~~~~

By this point we assume you have successfully run configure either from
the base distribution directory or from a separate build directory (aka
VPATH build.) You are now ready to run ‘make’. You can optionally run
parallel make using the “-j” option which significantly speeds up the
build. If using the MPI compiler wrappers, occasionally using “-j” will
cause build failures because the MPI compiler wrapper creates a
temporary symlink to the mpif.h header. In that case, you won’t be able
to use the “-j” option. Further, the influential environment variables
used at configure-time can be overridden at make-time in case problems
are encountered. For example:

::

   ./configure CFLAGS=-Wimplicit
   ...
   make CFLAGS="-Wimplicit -g -O0"

One particularly influential make variable is “V” which controls the
verbosity of the make output. This variable corresponds to the
``--disable-silent-rules/--enable-silent-riles`` configure-time option,
but we recommend the make-time variable:

::

   make V=0 (configure --enable-silent-rules)
   make V=1 (configure --disable-silent-rules)

Test Programs
~~~~~~~~~~~~~

Running “make checkprogs” will build most test and example programs.
Note that not all tests are built - some tests depend on certain
features being detected or enabled during configure. These programs are
not intented to be examples of good GA coding practices because they
often include private headers. However, they help us debug or time our
GA library.

Test Suite
~~~~~~~~~~

Running “make check” will build most test and example programs (See
“make checkprogs” notes above) in addition to running the test suite.
The test suite runs both the serial and parallel tests. The test suite
must know how to launch the parallel tests via the MPIEXEC variable.
Please read your MPI flavor’s documentation on how to launch, or if
using TCGMSG you will use the “parallel” tool. For example, the
following is the command to launch the test suite when compiled with
OpenMPI:

::

   make check MPIEXEC="mpiexec -np 4"

All tests have a per-test log file containing the output of the test. So
if the test is global/testing/test.x, the log file would be
global/testing/test.log. The output of failed tests is collected in the
top-level log summary test-suite.log.

The test suite will recurse into the ComEx directory and run the ComEx
test suite first. If the ComEx test suite fails, the GA test suite will
not run (the assumption here is that you should fix bugs in the
dependent library first.) To run only the GA test suite, type “make
check-ga” with the appropriate MPIEXEC variable.

Performance Tuning
~~~~~~~~~~~~~~~~~~

Setting an environment variable MA_USE_ARMCI_MEM forces MA library to
use ARMCI memory, communication via which can be faster on networks like
GM, VIA and InfiniBand.

CMake Build
^^^^^^^^^^^

The CMake build only supports the MPI-based runtimes so GA can only be
built using MPI two-sided, MPI progress ranks, MPI thread multiple, MPI
progress threads and MPI-3 (MPI RMA) runtimes. We recommend using MPI
two-sided/MPI progress ranks based approach.

Dependencies
~~~~~~~~~~~~

-  CMake (v3.18+)
-  MPI
-  BLAS / LAPACK (Optional)

The following options are supported:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  ``ENABLE_CXX`` [Default:ON]
-  ``ENABLE_FORTRAN`` [Default:ON]
-  ``ENABLE_TESTS`` Build GA testsuite. [Default:ON]
-  ``GA_RUNTIME`` [Default: MPI_2SIDED] Options are

   -  MPI_2SIDED (Default) use simple MPI-2 sided runtime
   -  MPI_PROGRESS_RANK Use progress ranks runtime
   -  MPI_MULTITHREADED Use thread multiple runtime
   -  MPI_PROGRESS_THREAD Use progress thread runtime
   -  MPI_RMA Use MPI RMA based runtime.

-  ``ENABLE_SYSV`` Enable System V Shared Memory
-  ``ENABLE_PROFILING`` Build GA operation profiler. Does not work when
   using Clang compilers. [Default:OFF]
-  ``GA_EXTRA_LIBS`` Specify additional libraries or linker options when
   building GA.
-  ``GCCROOT`` Specify root of GCC installation. Only required when
   building with Clang compilers.
-  ``ENABLE_BLAS`` Use an external BLAS library. [Default:OFF]

   -  ``Note``: ``ENABLE_BLAS=OFF`` builds internal (netlib) ``BLAS``.
   -  Only ``IntelMKL``, ``IBMESSL``, ``BLIS``, ``OpenBLAS``,
      ``ReferenceBLAS`` (Netlib) are supported.
   -  Need to provide the following cmake options if ``ENABLE_BLAS=ON``

      -  ``LINALG_VENDOR``: Should be one of ``IntelMKL``, ``IBMESSL``,
         ``BLIS``, ``OpenBLAS``, ``ReferenceBLAS`` (Netlib)
      -  ``LINALG_PREFIX``: Specify root of the LinAlg libraries
         installation. If the various libraries are in different
         locations, one needs to set ``BLAS_PREFIX``, ``LAPACK_PREFIX``,
         ``ScaLAPACK_PREFIX`` individually. These three options are set
         to the ``LINALG_PREFIX`` provided by default unless explicitly
         set otherwise.
      -  ``LINALG_THREAD_LAYER``: Options are ``openmp`` (default),
         ``sequential`` for ``IntelMKL`` and ``smp`` (default) for
         ``IBMESSL``. Does not apply to other BLAS libraries.
      -  ``LINALG_REQUIRED_COMPONENTS``: Options are ``lp64`` or
         ``ilp64``. [Default:lp64]
      -  ``LINALG_OPTIONAL_COMPONENTS``: ``sycl`` [Default:none]
      -  ``ENABLE_SCALAPACK``: To enable ScaLAPACK discovery.

-  ``[OPTIONAL]`` CTEST options for handling different types of job
   launchers and their parameters.

   -  ``GA_JOB_LAUNCH_CMD``: ``mpirun``
   -  ``GA_JOB_LAUNCH_ARGS``: ``"-n 5"`` 

  **The following options are standard CMake parameters. More information about them can be found in the CMake documentation.**

-  ``CMAKE_INSTALL_PREFIX`` Specify the install location for GA.
-  ``CMAKE_BUILD_TYPE`` [Default:RELEASE] The options are:

   -  RELWITHDEBINFO This will be compiled in a release mode but with
      debugger information (-g) included
   -  RELEASE Compiled in release mode and no debugger information is
      included in the code
   -  DEBUG Compiled with internal debugger information

-  ``BUILD_SHARED_LIBS`` Build GA as a shared library. [Default:OFF]

**If there is a missing feature that you would like to be added to the CMake build, please submit a feature request to our** `GitHub issue tracker <https://github.com/GlobalArrays/ga/issues>`__.


-  Sample CMake invocation for Linux/MAC users

   -  A minimal invocation with defaults for all options:

   ::

      CC=gcc CXX=g++ FC=gfortran cmake -DCMAKE_INSTALL_PREFIX=$HOME/ga_install

   -  A more complete invocation that shows most options:

   ::

      CC=gcc CXX=g++ FC=gfortran cmake -DCMAKE_INSTALL_PREFIX=$HOME/ga_install \ 
      -DGA_RUNTIME=MPI_PROGRESS_RANK \
      -DENABLE_BLAS=ON -DLINALG_VENDOR=IntelMKL -DLINALG_PREFIX=/opt/intel/mkl \
      -DENABLE_TESTS=ON -DENABLE_CXX=ON -DENABLE_FORTRAN=ON -DENABLE_PROFILING=OFF  

-  Sample CMake invocation for Windows users.

   -  ``ENABLE_FORTRAN=OFF`` option is needed for Windows build.
   -  We do not recommend using the ``BLAS`` and ``PROFILING`` options
      as they are not tested for Windows builds.

   ::

      cmake ^
         -D ENABLE_FORTRAN=OFF ^
         -D GA_RUNTIME=MPI_2SIDED ^
         -D CMAKE_INSTALL_PREFIX:PATH="\my\GA\install\path" ^
         ..
      cmake --build . --config Release
      cmake --build . --config Release --target install

**Known issues:** The CMake build currently does not work with IBM XL compilers.

