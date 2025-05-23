cmake_minimum_required(VERSION 3.3)

project(cgreen)

if (NOT(CMAKE_MAJOR_VERSION LESS 3) AND APPLE)
  if(POLICY CMP0042)
    cmake_policy(SET CMP0042 NEW) # CMake 3.0 to use @rpath on MacOSX libraries
  endif()
endif()

find_package (Threads)

set(CGREEN_WITH_XML ON CACHE BOOL
    "Add a simple XML report generator without external dependencies")
set(CGREEN_WITH_LIBXML2 ON CACHE BOOL
    "Add an XML report generator which uses libxml2 for output formatting")

if (CGREEN_WITH_LIBXML2)
  find_package (LibXml2)
  if (NOT LibXml2_FOUND)
    set(CGREEN_WITH_LIBXML2 OFF)
  endif (NOT LibXml2_FOUND)
endif (CGREEN_WITH_LIBXML2)

enable_testing()

# global needed variables
set(APPLICATION_NAME ${PROJECT_NAME})

# VERSION:
# NOTE: If you change version here, also change in
# include/cgreen/cgreen.h unless you write some code that
# automatically updates that...
set(APPLICATION_VERSION_MAJOR "1")
set(APPLICATION_VERSION_MINOR "6")
set(APPLICATION_VERSION_PATCH "4")

set(APPLICATION_VERSION ${APPLICATION_VERSION_MAJOR}.${APPLICATION_VERSION_MINOR}.${APPLICATION_VERSION_PATCH}${APPLICATION_VERSION_STATUS})
add_definitions(-DVERSION="${APPLICATION_VERSION}")

set(LIBRARY_VERSION ${APPLICATION_VERSION_MAJOR}.${APPLICATION_VERSION_MINOR}.${APPLICATION_VERSION_PATCH})
set(LIBRARY_SOVERSION ${APPLICATION_VERSION_MAJOR})


# INSTALL:
include(GNUInstallDirs)

if(MSVC)
  # these have sensible defaults on other platforms
  set(CMAKE_INSTALL_BINDIR ".")
  set(CMAKE_INSTALL_LIBDIR ".")
endif(MSVC)

# If OSX and using Homebrew use its install prefix
find_program(BREW brew)
if (BREW)
  execute_process(COMMAND brew --prefix OUTPUT_VARIABLE CMAKE_INSTALL_PREFIX OUTPUT_STRIP_TRAILING_WHITESPACE)
endif()

# COMPLETION:
set(BASHCOMPLETION_STATUS "Unavailable")
find_package(bash-completion QUIET)
if(BASH_COMPLETION_FOUND OR UNIX)
  set(BASHCOMPLETION_STATUS "Available")
  install(FILES tools/cgreen_completion.bash DESTINATION "${CMAKE_INSTALL_DATADIR}/bash-completion/completions" RENAME "cgreen-runner")
  install(FILES tools/cgreen_completion.bash DESTINATION "${CMAKE_INSTALL_DATADIR}/bash-completion/completions" RENAME  "cgreen-debug")
endif()

# CMAKE MODULES:
# where to look first for cmake modules, before ${CMAKE_ROOT}/Modules/ is checked
set(CMAKE_MODULE_PATH
  ${PROJECT_SOURCE_DIR}/cmake/Modules
)

# add definitions
include(DefineCMakeDefaults)
include(DefineCompilerFlags)
include(DefineOptions.cmake)
include(DefineCPackConfig)

# add macros
include(MacroAddPlugin)
include(MacroCopyFile)

# GIT REVISION:
# Generate gitrevision.h if Git is available and the .git directory is found.
find_program(GIT_EXECUTABLE git DOC "Git version control")
mark_as_advanced(GIT_EXECUTABLE)

# Find path to .git/logs/HEAD so that the gitrevision.h generation will only
# happen on new commits. This is done from $PROJECT_SOURCE_DIR because git rev-parse
# doesn't always return an absolute path so need to use get_filename_component()
# to get a cross-platform `readlink -f $(git rev-parse --git-path logs/HEAD)`
execute_process(
  COMMAND ${GIT_EXECUTABLE} rev-parse --git-path logs/HEAD
  WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
  OUTPUT_VARIABLE GITDIR
  OUTPUT_STRIP_TRAILING_WHITESPACE
  )
if (GITDIR)
  get_filename_component(GITDIR "${GITDIR}" ABSOLUTE)
endif()

# config.h checks
include(ConfigureChecks.cmake)
configure_file(config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)

# check subdirectories
add_subdirectory(src)
add_subdirectory(doc)
add_subdirectory(include)

# Dependency on our own library so we can use CGREEN_LIBRARY in all subdirectories
if (CGREEN_WITH_STATIC_LIBRARY)
  set(CGREEN_LIBRARY ${CGREEN_STATIC_LIBRARY})
else ()
  set(CGREEN_LIBRARY ${CGREEN_SHARED_LIBRARY})
endif()

if (CGREEN_WITH_UNIT_TESTS)
  include(MacroAddUnitTest)
  include(MacroAddTest)
  include(MacroAddValgrindTest)
  add_subdirectory(tests)
  if (UNIX OR MSYS)
    # reflective runner only supported on UNIX/binutils platforms
    add_subdirectory(tools)
  endif(UNIX OR MSYS)
endif (CGREEN_WITH_UNIT_TESTS)

# add custom 'check' target to run tests with output-on-failure
if (CMAKE_CONFIGURATION_TYPES)
    add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND}
        --force-new-ctest-process --output-on-failure
        --build-config "$<CONFIGURATION>")
else()
    add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND}
        --force-new-ctest-process --output-on-failure)
endif()

IF(CGREEN_INTERNAL_WITH_GCOV)
  IF(CMAKE_C_COMPILER_ID STREQUAL GNU)
    include(CodeCoverage)
    add_custom_target(coverage
        COMMAND ${LCOV_PATH} --directory . --capture --output-file coverage.info
        COMMAND ${LCOV_PATH} --remove coverage.info '/usr/*' -o coverage.info
        COMMAND ${GENHTML_PATH} -o coverage coverage.info
    )
  ENDIF()
ELSE()
  add_custom_target(coverage
      COMMAND echo "WARNING: Configure CGREEN_INTERNAL_WITH_GCOV to get coverage")
ENDIF()


#### Begin cgreen package configuration steps. ####

# After install other CMake projects can
# use find_package( cgreen )

set( CONFIG_INSTALL_DIR
        "${CMAKE_INSTALL_LIBDIR}/cmake/${APPLICATION_NAME}" )

set( PROJECT_CONFIG_IN
        "${CMAKE_CURRENT_SOURCE_DIR}/cgreen-config.cmake.in" )

set( VERSION_CONFIG_IN
        "${CMAKE_CURRENT_SOURCE_DIR}/cgreen-config-version.cmake.in" )

set( PROJECT_CONFIG
        "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/cgreen-config.cmake" )

set( VERSION_CONFIG
        "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/cgreen-config-version.cmake" )

configure_file( ${PROJECT_CONFIG_IN}
        "${PROJECT_CONFIG}" @ONLY )

configure_file( ${VERSION_CONFIG_IN}
        "${VERSION_CONFIG}" @ONLY )

install(FILES
        "${PROJECT_CONFIG}" "${VERSION_CONFIG}"
        DESTINATION
        "${CONFIG_INSTALL_DIR}" )

#### End cgreen package configuration steps. ####
