# Python binding for the GSM engine. Built from the repository root via
# scikit-build-core (root pyproject.toml defines the wheel); the engine
# libraries are provided by the parent CMakeLists.

find_package(Python3 COMPONENTS Interpreter Development.Module REQUIRED)
find_package(pybind11 CONFIG REQUIRED)

pybind11_add_module(pydatagramdb python.cpp)

target_include_directories(pydatagramdb PRIVATE ${Python3_INCLUDE_DIRS})

# The NDP/secondary-memory bindings (RandomAccessBulkReader & co.) need the
# parser/bulk subsystem plus capnproto/rocksdb; enable with -DDATAGRAMDB_NDP
# once those dependencies build on this platform.
if (DATAGRAMDB_NDP)
    target_compile_definitions(pydatagramdb PRIVATE DATAGRAMDB_NDP)
endif()

target_link_libraries(pydatagramdb PRIVATE pybind11::headers pybind11::module antlr4_static yaucl_structures roaring yaucl_hashing yaucl_graph yaucl_data gsm2 gsm_script)

install(TARGETS pydatagramdb LIBRARY DESTINATION .)
