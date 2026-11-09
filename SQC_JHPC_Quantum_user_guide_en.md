# JHPC-quantum Quantum and Supercomputing Collaboration Platform SQC User Guide

# 1. Introduction
This document describes how to use SQC to run quantum applications on the JHPC-quantum collaboration platform from "Fugaku". <br>
If you want to view this procedure in a browser, please refer to the following: <br>
[jhpc-quantum GitHub](https://github.com/jhpc-quantum)<br>
For update information, please refer to the [release notes](https://github.com/jhpc-quantum/documents/blob/main/releasenotes.md).

# 2. What is SQC?
SQC is a library that provides a C/C++ interface (C-API) for the following purposes.
* Creating and loading quantum circuits
* Transpilation optimized for quantum computers directly connected to Fugaku (IBM quantum computers only)
* Execution of quantum circuits on quantum computers directly connected to Fugaku

For a list of available C-APIs, please refer to the official documentation ([SQC.md](https://github.com/jhpc-quantum/SQC/blob/main/docs/SQC.md)).

# 3. How to Use
## 3.1. Environment Setup

※If the environment has already been set up, you do not need to build it again.<br>
※The scripts described below are common to the front-end server and compute nodes (Arm) of "Fugaku".

### 3.1.1. Creating a Working Directory
Create an SQC directory in any working directory of your choice ($WORK).  
```
$ mkdir -p ${WORK}/SQC
$ cd ${WORK}/SQC
```

### 3.1.2. Building a Python Virtual Environment and Installing Packages
This step is required only when the target is an IBM quantum computer.<br>
Use the environment setup script (venv_setup.sh) to build a Python virtual environment and install the required packages.<br>
The packages to be installed are qiskit, qiskit-ibm-runtime, qiskit-qasm3-import, and related packages.<br>
Installation takes a few minutes.

Copy the environment setup script (venv_setup.sh) and environment variable settings script (config.sh) from the shared directory (/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK_latest/<***SQC_library_x.x.x***>) to your working directory.  
※Specify a64fx or x86 according to your environment in place of <***ARCH***>.<br>
※Specify the version used in place of x.x.x in <***SQC_library_x.x.x***>.
```
$ cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<SQC_library_x.x.x>/venv_setup.sh .
$ cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<SQC_library_x.x.x>/config.sh .
```

Execute the environment setup script (venv_setup.sh).  
Example job script (when running on the front-end server)  
```
#!/bin/bash
#SBATCH -p mem1            # Specify queue
#SBATCH -n 1               # Specify number of CPUs
#SBATCH --mem 27G          # Specify memory size
#SBATCH --time=00:10:00    # Maximum execution time

./venv_setup.sh
```
Example job script (when running on a compute node (Arm))  
```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:10:00"
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -g groupname

./venv_setup.sh
```

Environment setup script (venv_setup.sh)
```
#!/bin/bash

#Setting up a virtual environment and installing packages
# 1. Set up environment variables
source ./config.sh

# 2. Load related packages with Spack
source /vol0004/apps/oss/spack/share/spack/setup-env.sh
spack load ${SPACK_PKG}

# 3. Set up a Python virtual environment in each user's working directory
mkdir -p ${TARGET_NAME}
cd ${TARGET_NAME}
python -m venv ${VENV_NAME}

# 4. Install packages such as qiskit into the virtual environment
source ./${VENV_NAME}/bin/activate
python -m pip install -r ${SQC_DIR}/requirements.txt
deactivate
```

Environment variable setup script (config.sh)  
※When building the SQC environment and running SQC programs on Fugaku, required packages are loaded using Spack.  
The hash value of the Spack package used in the following environment variable settings script is current as of May 2026.<br>
※<**x.x.x**> indicates the version of the SQC library being used.
```
#!/bin/bash

#SQC Library Version
SQC_VERSION=<x.x.x>
#Architecture
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
  #Package Name：python, gcc, boost
  SPACK_PKG="python@3.13.5/xwl6x7i gcc@15.1.0/pmdm4gu boost@1.88.0/zxt2ddd"
  #Target Name
  TARGET_NAME=x86
elif [ "$ARCH" = "aarch64" ]; then
  #Package Name：python, gcc, boost, numpy
  SPACK_PKG="python@3.13.5/qhm66vh gcc@15.1.0/c3wm4pb boost@1.88.0/cx2lqoq py-numpy@2.2.6/ce7iuxj"
  #Target Name
  TARGET_NAME=a64fx
else
  echo "error ${ARCH} is unknown"
  exit 1
fi
#Python Virtual Environment Name
VENV_NAME=venv_SQC_${SQC_VERSION}
#SQC Library Directory
SQC_DIR=/vol0300/share/ra010014/jhpcq_modules/${TARGET_NAME}/SDK/SQC_library_${SQC_VERSION}
```
Once the environment setup script has been executed successfully, a Python virtual environment and packages are installed under the SQC directory in the following structure:
```
`-- ${WORK}
        |-- SQC
        |    |-- <ARCH>
        |    |      `-- ${VENV_NAME} (Python virtual environment)
        |    |-- config.sh
        |    `-- venv_setup.sh
```

## 3.2. Execution
### 3.2.1. Setting Up the Environment
The scripts and sample programs used in the following steps are stored in the shared directory (/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK_latest/<***SQC_library_x.x.x***>/samples).  
※Specify a64fx or x86 according to your environment in place of <***ARCH***>.<br>
※Specify the version used in place of x.x in <***SQC_library_x.x.x***>.<br>
<br>
<br>
Environment setup script (backend_setup.sh)  
The environment setup script loads the various settings and dependencies required to run SQC programs.  
※Set /path/to according to your environment.<br>
※As an argument to backend_setup.sh, specify one of the following target quantum computers.
* reimei
* reimei-simulator
* ibm-kobe-dacc
```
#!/bin/bash

#Setting up Remote Procedure Call (RPC)
# 1. Set up IP address of quantum computer to SQC_RPC_SERVER according to a argument
if [ "$#" -eq 1 ]; then
  QPU="$1"
  if [ $QPU = "reimei" ]; then
     # The server port number will be changed for different target
     export SQC_RPC_SERVER=ip:port
  elif [ $QPU = "reimei-simulator" ]; then
     export SQC_RPC_SERVER=ip:port
  elif [ $QPU = "ibm-kobe-dacc" ]; then
     export SQC_RPC_SERVER=ip:port
  else
     echo "invalid qpu $QPU"
  fi
else
 echo "usage: /path/to/backend_setup.sh <target-qpu-name>"
fi

# 2. Set up environment variables
source /path/to/config.sh

# 3. Load related packages with Spack
source /vol0004/apps/oss/spack/share/spack/setup-env.sh
spack load ${SPACK_PKG}

# 4. Add SQC library paths to various environment variables
export LD_LIBRARY_PATH=${SQC_DIR}/lib:${SQC_DIR}/lib64:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=${SQC_DIR}/lib64/pkgconfig
export PYTHONPATH=${SQC_DIR}/python:${PYTHONPATH}

# 5. Set SQC_COMPILE_OPTIONS variable to compile program using SQC C-API.
SQC_LIBS="-lsqc_api -lsqc_rpc -lsqc_dbmgr\
 -lsqc_util -lsqc_rpccommon -luuid -lsqlite3 -lprotobuf -labsl_leak_check -labsl_die_if_null\
 -labsl_log_initialize -lutf8_validity -lutf8_range -lgrpc++ -lgrpc -laddress_sorting -lupb_textformat_lib\
 -lupb_json_lib -lupb_wire_lib -lupb_message_lib -lutf8_range_lib -lupb_mini_descriptor_lib -lupb_mem_lib\
 -lupb_base_lib -labsl_statusor -lgpr -labsl_log_internal_check_op -labsl_flags_internal\
 -labsl_flags_reflection -labsl_flags_private_handle_accessor -labsl_flags_commandlineflag\
 -labsl_flags_commandlineflag_internal -labsl_flags_config -labsl_flags_program_name -labsl_raw_hash_set\
 -labsl_hashtablez_sampler -labsl_flags_marshalling -labsl_log_internal_conditions\
 -labsl_log_internal_message -labsl_examine_stack -labsl_log_internal_format -labsl_log_internal_proto\
 -labsl_log_internal_nullguard -labsl_log_internal_log_sink_set -labsl_log_internal_globals -labsl_log_sink\
 -labsl_log_entry -labsl_log_globals -labsl_hash -labsl_city -labsl_low_level_hash\
 -labsl_vlog_config_internal -labsl_log_internal_fnmatch -labsl_random_distributions\
 -labsl_random_seed_sequences -labsl_random_internal_pool_urbg -labsl_random_internal_randen\
 -labsl_random_internal_randen_hwaes -labsl_random_internal_randen_hwaes_impl\
 -labsl_random_internal_randen_slow -labsl_random_internal_platform -labsl_random_internal_seed_material\
 -labsl_random_seed_gen_exception -labsl_status -labsl_cord -labsl_cordz_info -labsl_cord_internal\
 -labsl_cordz_functions -labsl_exponential_biased -labsl_cordz_handle -labsl_crc_cord_state -labsl_crc32c\
 -labsl_crc_internal -labsl_crc_cpu_detect -labsl_bad_optional_access -labsl_strerror\
 -labsl_str_format_internal -labsl_synchronization -labsl_graphcycles_internal\
 -labsl_kernel_timeout_internal -labsl_stacktrace -labsl_symbolize -labsl_debugging_internal\
 -labsl_demangle_internal -labsl_malloc_internal -labsl_time -labsl_civil_time -labsl_strings\
 -labsl_strings_internal -labsl_string_view -labsl_base -labsl_spinlock_wait -labsl_int128\
 -labsl_throw_delegate -labsl_time_zone -labsl_bad_variant_access -labsl_raw_logging_internal\
 -labsl_log_severity -lcares -lssl -lre2 -lz -lmunge -lcrypto -lsqc_rpccommon -lprotobuf-c -lrt\
 -lpthread -ldl -lnuma -pthread"
SQC_INCS="-I${SQC_DIR}/include"
PY_PATH=$(readlink -f $(which python3.13) | sed 's@/bin/python3.13@@g')
PYLIB="-L${PY_PATH}/lib -lpython3.13"

SQC_COMPILE_OPTIONS="-D_GNU_SOURCE ${SQC_INCS} -L${SQC_DIR}/lib -L${SQC_DIR}/lib64 ${SQC_LIBS} ${PYLIB}"
```

### 3.2.2. Running the Program
This section explains how to run a sample program that executes a quantum circuit on the JHPC-quantum collaboration platform.

Run the script used to translate the sample program (test_comp.sh).

Example job script (when running on the front-end server)

```
#!/bin/bash
#SBATCH -p mem1            # Specify queue
#SBATCH -n 1               # Specify number of CPUs
#SBATCH --mem 27G          # Specify memory size
#SBATCH --time=00:10:00    # Maximum execution time

./test_comp.sh
```

Example job script (when running on a compute node (Arm))

```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:10:00"
#PJM -g groupname
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -S

./test_comp.sh
```

Script to translate the sample program (test_comp.sh)

```
#!/bin/bash

# 1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

# 2. Compile program
gcc sample_api_sync.c -o sample.out ${SQC_COMPILE_OPTIONS} 
```

<br>
Run the script used to execute the sample program (test_exec.sh).
<br>

Example job script (when running on the front-end server)  
```
#!/bin/bash
#SBATCH -p mem1            # Specify queue
#SBATCH -n 1               # Specify number of CPUs
#SBATCH --mem 27G          # Specify memory size
#SBATCH --time=00:10:00    # Maximum execution time

./test_exec.sh
```
Example job script (when running on a compute node (Arm))   
```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:10:00"
#PJM -g groupname
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -S

./test_exec.sh
```
Script to execute the sample program (test_exec.sh)<br>
※Specify the path to the Python virtual environment created in Section 3.1.2 in place of "/path/to/".
```
#!/bin/bash

# 1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

# 2. Activate the python virtual environment when the connection destination is IBM quantum computer
source /path/to/${VENV_NAME}/bin/activate

# 3. Execute program
./sample.out
```

### 3.2.3. Sample Programs
There are three sample programs.
* Synchronous execution
   * Executing a circuit created with the C-API (sample_api_sync.c)
   * Executing an OpenQASM file (sample_qasm_sync.c)
* Asynchronous execution
   * Executing a circuit created with the C-API (sample_api_async.c)

#### 3.2.3.1. Synchronous Execution: Executing a Circuit Created with the C-API (sample_api_sync.c)
```
#include "sqc_api.h"
#include "sqc_ecode.h"
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[])
{
  // Specify backend
  sqcBackend backend = SQC_RPC_SCHED_QC_TYPE_QTM_SIM_GRPC;

  // Initialize C-API
  sqcInitOptions* init_options = sqcMallocInitOptions();
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC){
    init_options->use_qiskit = 1;
  } else {
    init_options->use_qiskit = 0;
  }
  sqcInitialize(init_options);

  // Construct circuit
  const int qubits = 2;
  sqcQC* qcir = sqcQuantumCircuit(qubits);
  sqcHGate(qcir, 0);
  sqcCXGate(qcir, 0, 1);
  sqcMeasure(qcir, 0, 0, NULL);
  sqcMeasure(qcir, 1, 1, NULL);

  // Set run option
  sqcRunOptions* run_options = (sqcRunOptions*)malloc(sizeof(sqcRunOptions));
  sqcInitializeRunOpt(run_options);
  run_options->nshots = 10;
  run_options->qubits = qubits;
  run_options->outFormat = SQC_OUT_RAW;

  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC) {
    // Convert quantum circuit to OpenQASM string
    qcir->qasm = (char*)malloc(500);
    sqcConvQASMtoMemory(qcir, backend, qcir->qasm, 500);
    
    // Transpile quantum circuit
    sqcTranspile(qcir, backend, NULL);
    printf("QASM after transpile: %s\n", qcir->qasm);
  }

  // Run quantum circuit
  sqcOut* result_out = (sqcOut *)malloc(sizeof(sqcOut));
  int error_code = sqcQCRun(qcir, backend, *run_options, result_out);

  // Show error_code
  printf("error_code:%d\n", error_code);

  // Write result to file
  if ( error_code == SQC_RESULT_OK ) {
    FILE *file;
    file = fopen("result_api_sync.txt", "w");
    if (file == NULL) {
      printf("Error opening file.\n");
    } else {
      sqcPrintQCResult(file, result_out, run_options->outFormat);
      fclose(file);
    }
  }

  // End processing of C-API
  sqcFreeOut(result_out, run_options->outFormat);
  free(result_out);
  free(run_options);
  sqcDestroyQuantumCircuit(qcir);
  sqcFinalize(init_options);
  sqcFreeInitOptions(init_options);

  return 0;
}
```

#### 3.2.3.2. Synchronous Execution: Executing an OpenQASM File (sample_qasm_sync.c)
```
#include "sqc_api.h"
#include "sqc_ecode.h"
#include <stdlib.h>
#include <string.h>

#define MAX_QASM_LEN (1024*1024)

int main(int argc, char *argv[])
{
  // Specify backend and 
  sqcBackend backend = SQC_RPC_SCHED_QC_TYPE_QTM_SIM_GRPC;

  // Initialize C-API
  sqcInitOptions* init_options = sqcMallocInitOptions();
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC){
    init_options->use_qiskit = 1;
  } else {
    init_options->use_qiskit = 0;
  }
  sqcInitialize(init_options); 

  // Read OpenQASM file
  const int qubits = 2;
  sqcQC* qcir = sqcQuantumCircuit(qubits);
  qcir->qasm = (char *)calloc(sizeof(char), MAX_QASM_LEN);
  
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC){
    sqcReadQasmFile(&(qcir->qasm), "./sample.qasm3", MAX_QASM_LEN);
  } else {
    sqcReadQasmFile(&(qcir->qasm), "./sample.qasm2", MAX_QASM_LEN);
  }

  // Set run option
  sqcRunOptions* run_options = (sqcRunOptions*)malloc(sizeof(sqcRunOptions));
  sqcInitializeRunOpt(run_options);
  run_options->nshots = 10;
  run_options->qubits = qubits;
  run_options->outFormat = SQC_OUT_RAW;

  // Transpile quantum circuit
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC) {
    sqcTranspile(qcir, backend, NULL);
    printf("QASM after transpile: %s\n", qcir->qasm);
  }

  // Run quantum circuit
  sqcOut* result_out = (sqcOut *)malloc(sizeof(sqcOut));
  int error_code = sqcQCRun(qcir, backend, *run_options, result_out);
  
  // Show error_code
  printf("error_code:%d\n", error_code);

  // Write result to file
  if ( error_code == SQC_RESULT_OK ) {
    FILE *file;
    file = fopen("result_qasm.txt", "w");
    if (file == NULL) {
      printf("Error opening file.\n");
    } else {
      sqcPrintQCResult(file, result_out, run_options->outFormat);
      fclose(file);
    }
  }

  // End processing of C-API
  sqcFreeOut(result_out, run_options->outFormat);
  free(result_out);
  free(run_options);
  sqcDestroyQuantumCircuit(qcir);
  sqcFinalize(init_options);
  sqcFreeInitOptions(init_options);
  
  return 0;
}
```
Example OpenQASM file used above<br>
When using reimei or reimei-simulator (sample.qasm2)
```
OPENQASM 2.0;
include "qelib1.inc";
qreg q[2];
creg c[2];
h q[0];
cx q[0], q[1];
measure q[0] -> c[0];
measure q[1] -> c[1];
```
When using ibm-kobe-dacc (sample.qasm3)
```
OPENQASM 3.0;
include "stdgates.inc";
bit[2] c;
qubit[2] q;
h q[0];
cx q[0], q[1];
c[0] = measure q[0];
c[1] = measure q[1];
```

#### 3.2.3.3. Asynchronous Execution: Executing a Circuit Created with the C-API (sample_api_async.c)
```
// Copyright (C) 2026 RIKEN, Japan.

// Sample program using SQC C-API and a asynchronous execution function.

#include "sqc_api.h"
#include "sqc_ecode.h"
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[])
{
  // Specify backend
  sqcBackend backend = SQC_RPC_SCHED_QC_TYPE_QTM_SIM_GRPC;

  // Initialize C-API
  sqcInitOptions* init_options = sqcMallocInitOptions();
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC){
    init_options->use_qiskit = 1;
  } else {
    init_options->use_qiskit = 0;
  }
  sqcInitialize(init_options);

  // Construct circuit
  const int qubits = 2;
  sqcQC* qcir = sqcQuantumCircuit(qubits);
  sqcHGate(qcir, 0);
  sqcCXGate(qcir, 0, 1);
  sqcMeasure(qcir, 0, 0, NULL);
  sqcMeasure(qcir, 1, 1, NULL);

  // Set run option
  sqcRunOptions* run_options = (sqcRunOptions*)malloc(sizeof(sqcRunOptions));
  sqcInitializeRunOpt(run_options);
  run_options->nshots = 10;
  run_options->qubits = qubits;
  run_options->outFormat = SQC_OUT_RAW;

  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC) {
    // Convert quantum circuit to OpenQASM string
    qcir->qasm = (char*)malloc(500);
    sqcConvQASMtoMemory(qcir, backend, qcir->qasm, 500);
    
    // Transpile quantum circuit
    sqcTranspile(qcir, backend, NULL);
    printf("QASM after transpile: %s\n", qcir->qasm);
  }

  // Run quantum circuit
  sqc_handle_t* sqc_handle = (sqc_handle_t*)malloc(sizeof(sqc_handle_t));
  int run_error_code = sqcQCRunAsync(qcir, backend, *run_options, sqc_handle);
  printf("sqcQCRunAsync error_code:%d\n", run_error_code);
  printf("Job ID:%s\n", sqc_handle->job_id);

  // Wait job
  sqcOut* result_out = (sqcOut *)malloc(sizeof(sqcOut));
  int wait_error_code = sqcQCWait(result_out, *sqc_handle);
  printf("sqcQCWait error_code:%d\n", wait_error_code);

  // Write result to file
  if ( wait_error_code == SQC_RESULT_OK ) {
    FILE *file;
    file = fopen("result_api_async.txt", "w");
    if (file == NULL) {
      printf("Error opening file.\n");
    } else {
      sqcPrintQCResult(file, result_out, run_options->outFormat);
      fclose(file);
    }
  }

  // End processing of C-API
  sqcFreeOut(result_out, run_options->outFormat);
  free(result_out);
  free(sqc_handle);
  free(run_options);
  sqcDestroyQuantumCircuit(qcir);
  sqcFinalize(init_options);
  sqcFreeInitOptions(init_options);

  return 0;
}
```

### 3.2.4. Example Execution Results of the Sample Programs
Below is an example of the execution result file for the sample programs.<br>
The contents of the execution result file are the same for all sample programs.<br>
For reimei or reimei-simulator
```
{
   c    : "00"
}
{
   c    : "00"
}
{
   c    : "11"
}
{
   c    : "11"
}
{
   c    : "11"
}
{
   c    : "00"
}
{
   c    : "11"
}
{
   c    : "00"
}
{
   c    : "00"
}
{
   c    : "11"
}
```
For ibm-kobe-dacc
```
{
 "metadata": {
  "execution": {
   "execution_spans": [
    [
     {
      "date": "2025-10-27T06:24:56.046955"
     },
     {
      "date": "2025-10-27T06:24:56.941759"
     },
     {
      "0": [
       [
        10
       ],
       [
        0,
        1
       ],
       [
        0,
        10
       ]
      ]
     }
    ]
   ]
  },
  "version": 2
 },
 "results": [
  {
   "data": {
    "c": {
     "num_bits": 2,
     "samples": [
      "0x0",
      "0x3",
      "0x0",
      "0x3",
      "0x0",
      "0x3",
      "0x0",
      "0x2",
      "0x0",
      "0x0"
     ]
    }
   },
   "metadata": {
    "circuit_metadata": {}
   }
  }
 ]
}
```

```
