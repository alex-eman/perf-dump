Performance Dump Module (perf_dump)
============================================

`libperfdump` is a tool for collecting counter data from all processes in a
cluster and dumping counter values out to an HDF5 file with parallel I/O.

Usage
-----------------

Here's example usage of perf_dump from really simple MPI program:

    int main(int argc, char **argv) {
       MPI_Init(&argc, &argv);
       pdump_init();     // init perf_dump library

       int rank;
       MPI_Comm_rank(MPI_COMM_WORLD, &rank);

       vector<double> a(VEC_SIZE);
       vector<double> b(VEC_SIZE);

       srand(23578 * rank);
       for (size_t step=0; step < MAX_STEP; step++) {
          pdump_start_step();   // start a time step, starting counters

          init_vector(a);
          init_vector(b);
          double d = dot(a, b);

          double prod;
          MPI_Reduce(&d, &prod, 1, MPI_DOUBLE, MPI_SUM, 0, MPI_COMM_WORLD);
          pdump_end_step();     // end a time step and stop counters

          if (rank == 0) {
             cout << step << ":\t" << prod << endl;
          }
       }

       // finish writing out performance data to disk
       pdump_finalize();
       MPI_Finalize();
    }

This example will record counter values for each time step and write them
to a dataset in an hdf5 file called `perf-dump.h5`.

If you run the above program (which you can find in `test-perf-dump.C`),
the output should look like this:

    $ env PDUMP_EVENTS=PAPI_L1_TCM srun -n 4 -ppdebug ./test-perf-dump
    ============== perf_dump module started ============
      Initialized PAPI with 1 events:
          PAPI_L1_TCM
    ====================================================
    0:      4.60027e+21
    1:      4.67344e+21
    2:      4.6825e+21
    3:      4.61943e+21
    4:      4.60479e+21
    5:      4.54754e+21
    6:      4.56227e+21
    7:      4.52928e+21
    8:      4.51068e+21
    9:      4.60068e+21

This will produce a file with a data set for PAPI_L1_TCM counts.  The
dataset is stored in HDF5 format and might look kind of like this:

    $ h5dump ./perf-dump.h5
    HDF5 "./perf-dump.h5" {
    GROUP "/" {
       DATASET "PAPI_L1_TCM" {
          DATATYPE  H5T_STD_I64LE
          DATASPACE  SIMPLE { ( 4, 11 ) / ( 4, H5S_UNLIMITED ) }
          DATA {
          (0,0): 1032, 46912530456296, 32, 65, 3925982895, 12884901892,
          (0,6): 17179869188, 4, 9498448, 0, 134,
          (1,0): 305, 3926048431, 12884901892, 9498448, 25769803782, 0, 1,
          (1,7): 9498640, 9498640, 9499008, 4294967431,
          (2,0): 0, 9471688, 515396075520, 0, 0, 0, 0, 46912501089472, 9499008,
          (2,9): 0, 0,
          (3,0): 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
          }
       }
    }
    }

For more on HDF5, see the HDF Group's website: http://www.hdfgroup.org/HDF5/.

You can load this dataset into many visualization and data analysis packages to
examine the data you have collected.

Configuration
------------------------

In `perf_dump.h`, there are a number of environment variables you can set to change
the behavior of perf_dump:

    // Location to put dumps.  Best if this is in a parallel file system.
    // Default location is in the working directory.
    #define PDUMP_DUMP_DIR "PDUMP_DUMP_DIR"

    // List of PAPI event names to monitor, separated by ':'
    #define PDUMP_EVENTS "PDUMP_EVENTS"

    // Comma-separated list of timestep numbers to dump data for.
    // First timestep is 0.
    #define PDUMP_DUMP_STEPS "PDUMP_DUMP_STEPS"

    // Default value for PDUMP_EVENTS.
    #define PDUMP_DEFAULT_EVENTS  "PAPI_L1_TCM"

    // Chunk size for timestep dimension for stored files
    #define PDUMP_TIME_CHUNK "PDUMP_TIME_CHUNK"

If you want good performance, you probably want to point `PDUMP_DUMP_DIR` a the
parallel filesystem like `/p/lscratcha`, e.g.:

    export PDUMP_DUMP_DIR=/p/lscratcha/gamblin2/dumps

If you want to measure multiple events, you can customize PDUMP_EVENTS, e.g.:

    export PDUMP_EVENTS="PAPI_L1_TCM,PAPI_BR_UCN"


Types of builds
------------------------

In this project, there are builds three versions of libperfdump:

1. `libperfdump.so`
2. `libpmpi_perfdump.so`
2. `pnmpi_perfdump.so`

The first is a library with C linkage that lets the user call functions.  This is
the simplest one to use.  The second is a PMPI library, which can be linked
with an MPI program so that perf_dump is inited and finalized automatically.
The third is for PnMPI, and can be loaded as a PnMPI module.


