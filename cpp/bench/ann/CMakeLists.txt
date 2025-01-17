# =============================================================================
# Copyright (c) 2023, NVIDIA CORPORATION.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except
# in compliance with the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software distributed under the License
# is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
# or implied. See the License for the specific language governing permissions and limitations under
# the License.
# =============================================================================

# ##################################################################################################
# * compiler function -----------------------------------------------------------------------------

option(RAFT_ANN_BENCH_USE_FAISS_BFKNN "Include faiss' brute-force knn algorithm in benchmark" ON)
option(RAFT_ANN_BENCH_USE_FAISS_IVF_FLAT "Include faiss' ivf flat algorithm in benchmark" ON)
option(RAFT_ANN_BENCH_USE_FAISS_IVF_PQ "Include faiss' ivf pq algorithm in benchmark" ON)
option(RAFT_ANN_BENCH_USE_RAFT_BFKNN "Include raft's brute-force knn algorithm in benchmark" ON)
option(RAFT_ANN_BENCH_USE_RAFT_IVF_FLAT "Include raft's ivf flat algorithm in benchmark" ON)
option(RAFT_ANN_BENCH_USE_RAFT_IVF_PQ "Include raft's ivf pq algorithm in benchmark" ON)
option(RAFT_ANN_BENCH_USE_HNSWLIB "Include hnsw algorithm in benchmark" ON)
option(RAFT_ANN_BENCH_USE_GGNN "Include ggnn algorithm in benchmark" ON)

find_package(Threads REQUIRED)

set(RAFT_ANN_BENCH_USE_FAISS OFF)
if(RAFT_ANN_BENCH_USE_FAISS_BFKNN
   OR RAFT_ANN_BENCH_USE_FAISS_IVFPQ
   OR RAFT_ANN_BENCH_USE_FAISS_IFFLAT
)
  set(RAFT_ANN_BENCH_USE_FAISS ON)
endif()

set(RAFT_ANN_BENCH_USE_RAFT OFF)
if(RAFT_ANN_BENCH_USE_RAFT_BFKNN
   OR RAFT_ANN_BENCH_USE_RAFT_IVFPQ
   OR RAFT_ANN_BENCH_USE_RAFT_IVFFLAT
)
  set(RAFT_ANN_BENCH_USE_RAFT ON)
endif()

if(RAFT_ANN_BENCH_USE_HNSWLIB)
  include(cmake/thirdparty/get_hnswlib.cmake)
endif()

option(RAFT_ANN_BENCH_USE_MULTIGPU "Use multi-gpus (where possible) in benchmarks" OFF)

include(cmake/thirdparty/get_nlohmann_json.cmake)

if(RAFT_ANN_BENCH_USE_GGNN)
  include(cmake/thirdparty/get_ggnn.cmake)
endif()

if(RAFT_ANN_BENCH_USE_FAISS)
  include(cmake/thirdparty/get_faiss.cmake)
endif()

function(ConfigureAnnBench)

  set(oneValueArgs NAME)
  set(multiValueArgs PATH LINKS CXXFLAGS INCLUDES)

  cmake_parse_arguments(
    ConfigureAnnBench "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN}
  )

  set(BENCH_NAME ${ConfigureAnnBench_NAME}_ANN_BENCH)

  add_executable(
    ${BENCH_NAME} ${ConfigureAnnBench_PATH} bench/ann/src/common/conf.cpp
                  bench/ann/src/common/util.cpp
  )
  target_link_libraries(
    ${BENCH_NAME}
    PRIVATE raft::raft
            nlohmann_json::nlohmann_json
            $<$<BOOL:${RAFT_ANN_BENCH_USE_MULTIGPU}>:NCCL::NCCL>
            ${ConfigureAnnBench_LINKS}
            Threads::Threads
            ${RAFT_CTK_MATH_DEPENDENCIES}
            $<TARGET_NAME_IF_EXISTS:OpenMP::OpenMP_CXX>
            $<TARGET_NAME_IF_EXISTS:conda_env>
  )

  set_target_properties(
    ${BENCH_NAME}
    PROPERTIES # set target compile options
               INSTALL_RPATH "\$ORIGIN/../../../lib"
               CXX_STANDARD 17
               CXX_STANDARD_REQUIRED ON
               CUDA_STANDARD 17
               CUDA_STANDARD_REQUIRED ON
               POSITION_INDEPENDENT_CODE ON
               INTERFACE_POSITION_INDEPENDENT_CODE ON
  )

  set(${ConfigureAnnBench_CXXFLAGS} ${RAFT_CXX_FLAGS} ${ConfigureAnnBench_CXXFLAGS})

  target_compile_options(
    ${BENCH_NAME} PRIVATE "$<$<COMPILE_LANGUAGE:CXX>:${ConfigureAnnBench_CXXFLAGS}>"
                          "$<$<COMPILE_LANGUAGE:CUDA>:${RAFT_CUDA_FLAGS}>"
  )

  if(RAFT_ANN_BENCH_USE_${ConfigureAnnBench_NAME})
    target_compile_definitions(
      ${BENCH_NAME}
      PUBLIC
        RAFT_ANN_BENCH_USE_${ConfigureAnnBench_NAME}=RAFT_ANN_BENCH_USE_${ConfigureAnnBench_NAME}
    )
  endif()

  target_include_directories(
    ${BENCH_NAME}
    PUBLIC "$<BUILD_INTERFACE:${RAFT_SOURCE_DIR}/include>"
    PRIVATE ${ConfigureAnnBench_INCLUDES}
  )

  install(
    TARGETS ${BENCH_NAME}
    COMPONENT ann_bench
    DESTINATION bin/ann
    EXCLUDE_FROM_ALL
  )
endfunction()

if(RAFT_ANN_BENCH_USE_HNSWLIB)
  ConfigureAnnBench(
    NAME HNSWLIB PATH bench/ann/src/hnswlib/hnswlib_benchmark.cpp INCLUDES
    ${CMAKE_CURRENT_BINARY_DIR}/_deps/hnswlib-src/hnswlib CXXFLAGS "${HNSW_CXX_FLAGS}"
  )
endif()

if(RAFT_ANN_BENCH_USE_RAFT)
  ConfigureAnnBench(
    NAME
    RAFT_IVF_PQ
    PATH
    bench/ann/src/raft/raft_benchmark.cu
    $<$<BOOL:${RAFT_ANN_BENCH_USE_RAFT_IVF_PQ}>:bench/ann/src/raft/raft_ivf_pq.cu>
    $<$<BOOL:${RAFT_ANN_BENCH_USE_RAFT_IVF_FLAT}>:bench/ann/src/raft/raft_ivf_flat.cu>
    LINKS
    raft::compiled
  )
endif()

if(RAFT_ANN_BENCH_USE_FAISS)
  ConfigureAnnBench(
    NAME FAISS_IVF_FLAT PATH bench/ann/src/faiss/faiss_benchmark.cu LINKS faiss::faiss
  )
endif()

if(RAFT_ANN_BENCH_USE_GGNN)
  include(cmake/thirdparty/get_glog.cmake)
  ConfigureAnnBench(
    NAME GGNN PATH bench/ann/src/ggnn/ggnn_benchmark.cu INCLUDES
    ${CMAKE_CURRENT_BINARY_DIR}/_deps/ggnn-src/include LINKS glog::glog
  )
endif()
