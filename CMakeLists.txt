cmake_minimum_required(VERSION 3.22.1)
set(CMAKE_POLICY_VERSION_MINIMUM 3.5)
project(gsm2)
if(APPLE)
    set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "Build for both architectures" FORCE)
endif()
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(BUILD_TESTING OFF)
set(ANTLR_BUILD_CPP_TESTS OFF)
set(CMAKE_WARN_DEPRECATED OFF CACHE BOOL "" FORCE)

find_package(Python3 3.8 COMPONENTS Interpreter Development.Module)
find_package(Threads)

include(FetchContent)

# Easylogging++: only the source file and headers are needed (compiled into gsm2)
if (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/submodules/easylogging/src/easylogging++.cc")
    FetchContent_Declare(
            easylogging
            GIT_REPOSITORY https://github.com/foxr1/easyloggingpp.git
            GIT_TAG fix-macos-build
    )
    FetchContent_MakeAvailable(easylogging)
    include_directories(${easylogging_SOURCE_DIR}/src)
    set(ELPP_BASE_DIR "${easylogging_SOURCE_DIR}")
else()
    include_directories(submodules/easylogging/src)
    set(ELPP_BASE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/submodules/easylogging")
endif()

# YAUCL: sources are compiled directly into the static libraries below
set(YAUCL_LOCAL_PATH "${CMAKE_CURRENT_SOURCE_DIR}/submodules/yaucl")
if (NOT EXISTS "${YAUCL_LOCAL_PATH}/CMakeLists.txt")
    message(FATAL_ERROR "The yaucl submodule is required: git submodule update --init --recursive")
else()
    set(YAUCL_BASE_DIR "${YAUCL_LOCAL_PATH}")
endif()

# CRoaring
if (NOT TARGET roaring)
    if (NOT EXISTS "${YAUCL_BASE_DIR}/submodules/CRoaring/CMakeLists.txt")
        FetchContent_Declare(
                roaring
                GIT_REPOSITORY https://github.com/RoaringBitmap/CRoaring.git
                GIT_TAG v3.0.0
        )
        FetchContent_MakeAvailable(roaring)
    else()
        add_subdirectory(${YAUCL_BASE_DIR}/submodules/CRoaring binbuild/croaring)
    endif()
endif()

# ANTLR4 runtime
if (NOT TARGET antlr4_static)
    if (NOT EXISTS "${YAUCL_BASE_DIR}/submodules/antlr4/runtime/Cpp/CMakeLists.txt")
        FetchContent_Declare(
                antlr4_runtime
                GIT_REPOSITORY https://github.com/antlr/antlr4.git
                GIT_TAG 4.13.1
                SOURCE_SUBDIR runtime/Cpp
        )
        FetchContent_MakeAvailable(antlr4_runtime)
    else()
        add_subdirectory(${YAUCL_BASE_DIR}/submodules/antlr4/runtime/Cpp binbuild/antlrcpp)
    endif()
endif()

include(CTest)

IF(CMAKE_BUILD_TYPE MATCHES Debug)
    message(WARNING "Compiling in debug mode")
    add_compile_definitions(DEBUG)
    set(CMAKE_CXX_FLAGS_DEBUG "-g")
else()
    message(WARNING "Compiling in optimized mode. Level=3")
    set(CMAKE_CXX_FLAGS_RELEASE "-O3")
ENDIF(CMAKE_BUILD_TYPE MATCHES Debug)

include_directories(submodules/args)
include_directories(include)
include_directories(antlr4)
include_directories(${YAUCL_BASE_DIR}/submodules/magic_enum/include)
include_directories(${YAUCL_BASE_DIR}/include)
include_directories(${YAUCL_BASE_DIR}/submodules/json/include)
include_directories(${YAUCL_BASE_DIR}/submodules/CRoaring/cpp)
include_directories(${YAUCL_BASE_DIR}/submodules/CRoaring/include)

include_directories(${YAUCL_BASE_DIR}/submodules/antlr4/runtime/Cpp/runtime/src)
include_directories(${YAUCL_BASE_DIR}/antlr4/cpp/antlr4)
include_directories(submodules/tabulate/include)


add_library(yaucl_hashing STATIC
        ${YAUCL_BASE_DIR}/include/yaucl/hashing/evaluateHashes.h
        ${YAUCL_BASE_DIR}/src/yaucl/hashing/evaluateHashes.c
        ${YAUCL_BASE_DIR}/src/yaucl/hashing/hash_combine.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/hashing/hash_combine.h
        ${YAUCL_BASE_DIR}/src/yaucl/hashing/hashing.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/hashing/hashing.h
        ${YAUCL_BASE_DIR}/include/yaucl/hashing/pair_hash.h
        ${YAUCL_BASE_DIR}/include/yaucl/hashing/vector_hash.h
        ${YAUCL_BASE_DIR}/include/yaucl/hashing/uset_hash.h)
target_include_directories(yaucl_hashing PUBLIC ${YAUCL_BASE_DIR}/include/yaucl/hashing/ ${CMAKE_CURRENT_BINARY_DIR})

# Data layer required by the v4 engine core (mmap, serializers, fuzzy matching).
# RocksDBMap and xml.cpp are intentionally excluded: they belong to the optional
# NDP/secondary-memory subsystem (DATAGRAMDB_NDP), not the query engine.
add_library(yaucl_data STATIC
        ${YAUCL_BASE_DIR}/src/yaucl/data/csv/csv.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/data/new_iovec.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/data/MemoryMappingFile.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/data/mmapFile.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/data/VariadicSizeArrayElements.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/data/VarSizeNDPSorter.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/data/NDPFuzzyStringMatching.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/data/SimpleStringSerializer.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/data/json.h
        ${YAUCL_BASE_DIR}/src/yaucl/data/malloc_freelist.c
        ${YAUCL_BASE_DIR}/src/yaucl/data/SecondaryMemoryMalloc.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/data/SecondaryMemoryMalloc.h
        ${YAUCL_BASE_DIR}/include/yaucl/data/smart_malloc.h
        ${YAUCL_BASE_DIR}/src/yaucl/data/smart_malloc.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/data/smart_index_pointer.h
        ${YAUCL_BASE_DIR}/src/yaucl/data/smart_index_pointer.cpp)
target_link_libraries(yaucl_data yaucl_hashing)

add_library(yaucl_structures STATIC
        ${YAUCL_BASE_DIR}/include/yaucl/structures/setoids/set_comparators.h
        ${YAUCL_BASE_DIR}/include/yaucl/structures/setoids/set_partitioning.h
        ${YAUCL_BASE_DIR}/include/yaucl/structures/setoids/basics.h
        ${YAUCL_BASE_DIR}/include/yaucl/structures/indexed/BlockOfRecordsIndexedTable.h
        ${YAUCL_BASE_DIR}/src/yaucl/structures/RoaringBitmapWrapper.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/structures/NAryTree.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/structures/NAryTree.h
        ${YAUCL_BASE_DIR}/include/yaucl/structures/RelationalTables.h)
target_link_libraries(yaucl_structures roaring)


add_library(yaucl_graph STATIC
        ${YAUCL_BASE_DIR}/src/yaucl/graphs/adjacency_graph.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/adjacency_graph.h
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/FlexibleGraph.h
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/NodeLabelBijectionGraph.h
        ${YAUCL_BASE_DIR}/src/yaucl/graphs/graph_join_pm_algorithms.cpp
        ${YAUCL_BASE_DIR}/src/yaucl/graphs/graph_join_pm_conversion.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/NodeLabelBijectionFA.h
        ${YAUCL_BASE_DIR}/src/yaucl/graphs/adjacency_entry.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/adjacency_entry.h
        ${YAUCL_BASE_DIR}/src/yaucl/graphs/graph_join_pm.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/graph_join_pm.h
        ${YAUCL_BASE_DIR}/src/yaucl/graphs/minimizeDFA.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/algorithms/minimizeDFA.h
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/algorithms/all_cycles.h
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/algorithms/AllDirectedPaths.h
        ${YAUCL_BASE_DIR}/src/yaucl/graphs/algorithms/prim.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/algorithms/prim.h
        ${YAUCL_BASE_DIR}/src/yaucl/graphs/algorithms/connected_components.cpp
        ${YAUCL_BASE_DIR}/include/yaucl/graphs/algorithms/connected_components.h
        )
target_link_libraries(yaucl_graph  yaucl_hashing roaring)

add_library(gsm_script STATIC ./antlr4/scriptv2/scriptBaseListener.cpp
        ./antlr4/scriptv2/scriptBaseVisitor.cpp
        ./antlr4/scriptv2/scriptLexer.cpp
        ./antlr4/scriptv2/scriptParser.cpp
        ./antlr4/scriptv2/scriptVisitor.cpp
        ./antlr4/scriptv2/scriptListener.cpp
        ./src/scriptv2/ScriptAST.cpp ./include/scriptv2/ScriptAST.h ./src/scriptv2/Funzione.cpp ./include/scriptv2/Funzione.h ./src/scriptv2/Javification.cpp ./include/scriptv2/Javification.h ./src/scriptv2/ScriptASTVisitor.cpp ./include/scriptv2/ScriptASTVisitor.h)
target_link_libraries(gsm_script antlr4_static)

add_library(gsm2 STATIC antlr4/graph_grammar/simple_graph_grammarLexer.cpp antlr4/graph_grammar/simple_graph_grammarParser.cpp antlr4/graph_grammar/simple_graph_grammarListener.cpp  antlr4/graph_grammar/simple_graph_grammarVisitor.cpp src/database/PhiTable.cpp include/database/PhiTable.h src/database/AttributeTable.cpp include/database/AttributeTable.h src/database/SimplifiedFuzzyStringMatching.cpp include/database/SimplifiedFuzzyStringMatching.h src/database/LinearGSM.cpp include/database/LinearGSM.h src/database/ActivityTable.cpp include/database/ActivityTable.h src/result.cpp include/result.h src/database/gsm_indices.cpp include/database/gsm_indices.h src/queries/DataQuery.cpp include/queries/DataQuery.h src/database/gsm_object_xi_content.cpp include/database/gsm_object_xi_content.h include/simple_pair_hash.h src/queries/DataPredicate.cpp include/queries/DataPredicate.h
        ${ELPP_BASE_DIR}/src/easylogging++.cc ${ELPP_BASE_DIR}/src/easylogging++.h
        src/database/GSMPatternVisitor.cpp include/database/GSMPatternVisitor.h src/database/GraphEdgeMatchTable.cpp include/database/GraphEdgeMatchTable.h src/database/SimpleTable.cpp include/database/SimpleTable.h include/database/gsm_object.h src/database/gsm_object.cpp include/database/gsm_inmemory_db.h src/database/gsm_inmemory_db.cpp include/database/utility.h src/queries/preserve_results.cpp include/queries/preserve_results.h src/queries/delta_updates.cpp include/queries/delta_updates.h src/queries/closure.cpp include/queries/closure.h src/database/GSMIso.cpp include/database/GSMIso.h include/parsing.h include/configuration/base.h src/configuration/Serialisation.cpp include/configuration/Serialisation.h src/configuration/Configuration.cpp include/configuration/Configuration.h src/configuration/Environment.cpp include/configuration/Environment.h
        src/queries/NestedResultTable.cpp include/queries/NestedResultTable.h
        src/database/SchemaIndexer.cpp include/database/SchemaIndexer.h)
target_link_libraries(gsm2 antlr4_static yaucl_structures roaring  yaucl_hashing yaucl_graph yaucl_data)

add_executable(gsm2_server main.cpp)
target_link_libraries(gsm2_server  antlr4_static yaucl_structures roaring  yaucl_hashing yaucl_graph  gsm2 gsm_script)

if (SKBUILD)
    add_subdirectory(DatagramDB)
endif()
