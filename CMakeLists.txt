cmake_minimum_required(VERSION 3.16)
# Version of the Go bindings; keep in sync with Version in robotraconteur/version.go
project(RobotRaconteurGo VERSION 0.1.1 LANGUAGES C CXX)

# Native part of the Robot Raconteur Go bindings (see README.md, "Building"):
#   - libRobotRaconteurGoNative.a: the committed SWIG C++ wrapper
#     (native/gen, regenerated with scripts/regen_swig.sh) plus the core's
#     SWIG/RobotRaconteurWrapped.cpp. Static; linked into Go programs by cgo.
#   - RobotRaconteurGenGo: generates Go packages from .robdef files.
#   - lib/pkgconfig/robotraconteur-go.pc: the link line for cgo
#     (#cgo pkg-config: robotraconteur-go in robotraconteur/internal/swig).
# Needs an installed static RobotRaconteurCore (find_package) and the core
# source tree (RR_SOURCE_DIR) of the same version. SWIG and Python are not used.

set(RR_SOURCE_DIR "${CMAKE_SOURCE_DIR}/../robotraconteur" CACHE PATH "Robot Raconteur core source directory")
get_filename_component(RR_SOURCE_DIR "${RR_SOURCE_DIR}" ABSOLUTE)
if(NOT EXISTS "${RR_SOURCE_DIR}/SWIG/RobotRaconteurWrapped.cpp")
    message(FATAL_ERROR "RR_SOURCE_DIR (${RR_SOURCE_DIR}) is not a Robot Raconteur core source tree")
endif()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
include(GNUInstallDirs)

find_package(RobotRaconteur REQUIRED)
include(cmake/GoLinkFlags.cmake)

# The C++ wrapper depends on the data model: int64_t is long on LP64
# (Linux, macOS) and long long on LLP64 (Windows).
include(CheckTypeSize)
check_type_size("long" RRGO_SIZEOF_LONG)
if(RRGO_SIZEOF_LONG EQUAL 8)
    set(RRGO_DATA_MODEL lp64)
else()
    set(RRGO_DATA_MODEL llp64)
endif()
set(RRGO_WRAP_DIR "${CMAKE_SOURCE_DIR}/native/gen/${RRGO_DATA_MODEL}")
set(RRGO_WRAP_CXX "${RRGO_WRAP_DIR}/RobotRaconteurGo_wrap.cxx")

add_library(RobotRaconteurGoNative STATIC "${RRGO_WRAP_CXX}" "${CMAKE_SOURCE_DIR}/native/gen/rrgo_upcasts.cxx"
                                          "${RR_SOURCE_DIR}/SWIG/RobotRaconteurWrapped.cpp")
target_include_directories(RobotRaconteurGoNative PRIVATE "${RRGO_WRAP_DIR}" "${RR_SOURCE_DIR}/SWIG" "${CMAKE_SOURCE_DIR}/swig")
target_compile_definitions(RobotRaconteurGoNative PRIVATE SWIG_TYPE_TABLE=robotraconteur_go)
target_link_libraries(RobotRaconteurGoNative PUBLIC RobotRaconteurCore)
# The SWIG wrapper is a very large translation unit; keep memory use bounded.
set_source_files_properties("${RRGO_WRAP_CXX}" PROPERTIES COMPILE_OPTIONS "-g0")
install(TARGETS RobotRaconteurGoNative ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

# RobotRaconteurGenGo: Go package generator (uses the core robdef parser)
find_package(Boost REQUIRED COMPONENTS program_options filesystem)
add_executable(RobotRaconteurGenGo gen/main.cpp gen/GoServiceLangGen.cpp gen/GoServiceLangGen.h)
target_link_libraries(RobotRaconteurGenGo RobotRaconteurCore Boost::program_options Boost::filesystem)
target_compile_definitions(RobotRaconteurGenGo PRIVATE RRGO_VERSION_TEXT="${PROJECT_VERSION}")
install(TARGETS RobotRaconteurGenGo RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

# robotraconteur-go.pc: written at install time with the installed library
# paths and a fingerprint of the archives (cmake/write_pkgconfig.cmake).
set(RRGO_LINK_GROUP ON)
if(WIN32)
    # libstdc++, libgcc and winpthread linked statically: no MinGW DLLs needed at run time
    set(RRGO_SYSTEM_LIBS "-static -lstdc++ -lwinpthread")
elseif(APPLE)
    set(RRGO_SYSTEM_LIBS "-lc++ -lm -lpthread")
    set(RRGO_LINK_GROUP OFF)
else()
    set(RRGO_SYSTEM_LIBS "-lstdc++ -lm -lpthread")
endif()
rrgo_collect_link_flags(RRGO_LINK_FLAGS RobotRaconteurGoNative)
list(REMOVE_ITEM RRGO_LINK_FLAGS "$<TARGET_LINKER_FILE:RobotRaconteurGoNative>")
string(JOIN "\n" RRGO_LINK_LINES ${RRGO_LINK_FLAGS})
file(GENERATE OUTPUT "${CMAKE_BINARY_DIR}/rrgo_link_libs.txt" CONTENT "${RRGO_LINK_LINES}\n")
install(
    CODE "set(RRGO_PREFIX \"\${CMAKE_INSTALL_PREFIX}\")
          set(RRGO_LIBDIR \"${CMAKE_INSTALL_LIBDIR}\")
          set(RRGO_LINK_LIBS_FILE \"${CMAKE_BINARY_DIR}/rrgo_link_libs.txt\")
          set(RRGO_SYSTEM_LIBS \"${RRGO_SYSTEM_LIBS}\")
          set(RRGO_VERSION \"${PROJECT_VERSION}\")
          set(RRGO_CORE_VERSION \"${RobotRaconteur_VERSION}\")
          set(RRGO_LINK_GROUP ${RRGO_LINK_GROUP})
          include(\"${CMAKE_SOURCE_DIR}/cmake/write_pkgconfig.cmake\")")

# Generator golden test (ctest -R test_gen_go). Update with scripts/update_golden.sh.
enable_testing()
# The golden test runs the generator, which a cross build cannot do.
if(NOT CMAKE_CROSSCOMPILING)
    add_test(NAME test_gen_go
             COMMAND ${CMAKE_COMMAND} -DGEN=$<TARGET_FILE:RobotRaconteurGenGo> -DROBDEF_DIR=${RR_SOURCE_DIR}/test/robdef
                     -DOUT=${CMAKE_BINARY_DIR}/gen_golden -DGOLDEN=${CMAKE_SOURCE_DIR}/test/golden -P
                     ${CMAKE_SOURCE_DIR}/cmake/check_golden.cmake)
endif()
