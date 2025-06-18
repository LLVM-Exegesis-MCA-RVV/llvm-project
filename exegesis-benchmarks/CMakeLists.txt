##############################################################################
# Extra logic for RISCV llvm-exegesis benchmarks
##############################################################################

# --- 1.  Make sure llvm-exegesis itself is built ----------------------------
set(LLVM_TOOL_LLVM_EXEGESIS_BUILD ON CACHE BOOL
    "Always build the llvm-exegesis tool" FORCE)

# --- 2.  Cache variables the user can override ------------------------------
set(OPCODE_NAME "" CACHE STRING
    "Instruction / pseudo-opcode for the benchmark (required!)")

set(MTRIPLE          "riscv64"         CACHE STRING "mtriple for exegesis")
set(MCPU             "sifive-p670"     CACHE STRING "mcpu   for exegesis")
set(EXEGE_MODE       "latency"         CACHE STRING "llvm-exegesis --mode")
set(EXEGE_MATTR      "+d"              CACHE STRING "llvm-exegesis -mattr")
set(RISCV_GCC        "riscv64-unknown-elf-gcc" CACHE STRING "RISCV GCC")
set(ARCH_INCLUDE_DIR "${CMAKE_SOURCE_DIR}" CACHE PATH   "Dir with arch.h etc.")
set(HTIF_SPECS_FILE  "${CMAKE_SOURCE_DIR}/htif.specs"   CACHE FILEPATH "htif.specs")
set(HTIF_LD_SCRIPT   "${CMAKE_SOURCE_DIR}/htif.ld"      CACHE FILEPATH "htif.ld")
set(GLOSS_HTIF_LIB_DIR "/opt/htif/lib"        CACHE PATH     "libgloss-htif dir")

# Abort if the user forgot to pass -DOPCODE_NAME=…
if(NOT OPCODE_NAME)
  message(FATAL_ERROR "You must configure with -DOPCODE_NAME=<YourInsn>")
endif()

# --- 3.  File names derived from OPCODE_NAME --------------------------------
set(BENCHMARK_OBJ  "${CMAKE_BINARY_DIR}/${OPCODE_NAME}.o")
set(WRAPPER_SRC    "${CMAKE_SOURCE_DIR}/wrapper-PseudoVMERGE_VVM_M1.c")
set(COMMON_SRC     "${CMAKE_SOURCE_DIR}/common1.c")
set(BENCHMARK_ELF  "${CMAKE_BINARY_DIR}/benchmark-${OPCODE_NAME}.elf")

# Sanity checks
foreach(path ${WRAPPER_SRC} ${COMMON_SRC} ${ARCH_INCLUDE_DIR}
             ${HTIF_SPECS_FILE} ${HTIF_LD_SCRIPT})
  if(NOT EXISTS ${path})
    message(FATAL_ERROR "Required file not found: ${path}")
  endif()
endforeach()

# --- 4.  Generate <OPCODE>.o with the just-built llvm-exegesis ---------------
add_custom_command(
  OUTPUT       ${BENCHMARK_OBJ}
  COMMAND      $<TARGET_FILE:llvm-exegesis>
               "--mode=${EXEGE_MODE}"
               "--mtriple=${MTRIPLE}"
               "--opcode-name=${OPCODE_NAME}"
               "--mcpu=${MCPU}"
               --benchmark-phase=assemble-measured-code
               "-mattr=${EXEGE_MATTR}"
               "--dump-object-to-disk=${BENCHMARK_OBJ}"
  DEPENDS      llvm-exegesis       # ensure the tool itself is built first
  COMMENT      "llvm-exegesis → ${OPCODE_NAME}.o"
  VERBATIM
)

add_custom_target(exegesis-benchmark
  DEPENDS ${BENCHMARK_OBJ}
)

# --- 5.  Link the final RISCV ELF with riscv64-unknown-elf-gcc --------------
add_custom_command(
  OUTPUT  ${BENCHMARK_ELF}
  DEPENDS exegesis-benchmark ${WRAPPER_SRC} ${COMMON_SRC}
  COMMAND ${RISCV_GCC}
          -I${ARCH_INCLUDE_DIR}
          --target=${MTRIPLE}-unknown-elf
          -march=rv64gcv
          -mabi=lp64
          -static -fno-common -fno-builtin-printf
          -specs=${HTIF_SPECS_FILE}
          -T ${HTIF_LD_SCRIPT}
          ${COMMON_SRC}
          ${WRAPPER_SRC}
          ${BENCHMARK_OBJ}
          -o ${BENCHMARK_ELF}
          -L${GLOSS_HTIF_LIB_DIR} -lgloss-htif
  COMMENT "Linking ${BENCHMARK_ELF}"
  VERBATIM
)

add_custom_target(build-benchmark ALL
  DEPENDS ${BENCHMARK_ELF}
  COMMENT "Full llvm-exegesis benchmark build pipeline"
)
##############################################################################
# End of extra benchmark logic
##############################################################################
