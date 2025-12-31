cmake_minimum_required(VERSION 3.13)
cmake_policy(SET CMP0048 NEW)
cmake_policy(SET CMP0007 NEW)
cmake_policy(SET CMP0071 NEW)
cmake_policy(SET CMP0084 NEW)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${SYSTEMC_HOME})
# Choose whether to make documention as well
  option (BUILD_DOCS "Prepare also HTML/CHM documentation" ON)
# Choose whether you need debugging support for development or making a release
  option (DEBUG_MODE "Include debug support for the package"  ON)
# Choose whether you need to build GTest executables
  option(BUILD_GTESTS "Build Gtest tests" ON)
include(Colorize)
include(BuildType)

find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
    message(STATUS "CCache enabled")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

project(NeuronScQt
        LANGUAGES CXX
        DESCRIPTION "A SystemC-based time-aware neuron simulator with Qt GUI"
        HOMEPAGE_URL "https://github.com/jvegh/ScQtNeuron"
)

if(NOT ${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
    include(GNUInstallDirs) #Must exist after declaring a project
endif()

# Error flags on everything but MSVC
if(NOT MSVC)
    set(CMAKE_CXX_FLAGS "-Wextra -Wall \
        -Wno-dev \
        -Werror=switch -Werror=return-type \
        -Werror=unreachable-code")
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

	find_package(SystemCLanguage)
set(SystemC_INCLUDE_DIRS "/opt/systemc/include")
set(SystemC_LIBRARY_DIRS "/opt/systemc/lib")

include_directories(${SystemC_INCLUDE_DIRS})

######################################################################
## GUI setup
######################################################################

# Instruct CMake to run moc automatically when needed.
set(CMAKE_AUTOMOC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)

set(CMAKE_PREFIX_PATH "~/Qt6/6.6.1/gcc_64")
# Find required Qt packages
find_package(Qt6 COMPONENTS Core Widgets Svg Charts PrintSupport REQUIRED)

# Finding Qt includes
include_directories(${Qt6Widgets_INCLUDE_DIRS})
include_directories(${Qt6Core_INCLUDE_DIRS})
include_directories(${Qt6Charts_INCLUDE_DIRS})

######################################################################
## Resources setup
######################################################################
qt6_add_resources(ICONS_SRC ${CMAKE_SOURCE_DIR}/resources/icons/icons.qrc)
##??qt6_add_resources(EXAMPLES_SRC ${CMAKE_SOURCE_DIR}/examples/examples.qrc)
##??qt6_add_resources(LAYOUTS_SRC ${CMAKE_SOURCE_DIR}/src/processors/layouts.qrc)
qt6_add_resources(FONTS_SRC ${CMAKE_SOURCE_DIR}/resources/fonts/fonts.qrc)

######################################################################
## Library setup
######################################################################

# Header-only libraries used in the project must be included from top CMakeLists.txt file
include_directories(neuron)
include_directories(neuron/neurons)
include_directories(src/version)

# Platform specific pre-target setup(s)
if(${APPLE})
    set(SYSTEM_FLAGS MACOSX_BUNDLE)
elseif(${WIN32})
    set(SYSTEM_FLAGS WIN32)
endif()

# Fix the name of the libraries.
set(NEURON_LIB neuron_lib)
#?set(VERSION_LIB version_lib)
add_subdirectory(src)
#add_subdirectory(neuron)
add_subdirectory(3rdParty/qcustomplot)

add_subdirectory(demos)

#if(BUILD_GTESTS)
    add_subdirectory(gtest)
#endif()

if(BUILD_DOCS)
  file(COPY ${CMAKE_SOURCE_DIR}/doc/dynsections.js DESTINATION ${CMAKE_BINARY_DIR}/doc)
  find_package(Doxygen)
##  add_subdirectory(doc) # The doxygen documentation
endif(BUILD_DOCS)

set(APP_NAME NeuronScQt)
qt_add_executable(${APP_NAME} ${SYSTEM_FLAGS} ${ICONS_SRC} #${EXAMPLES_SRC} ${LAYOUTS_SRC}
${FONTS_SRC} main.cpp)

# Link Qt libraries
#target_link_libraries(${APP_NAME} PUBLIC Qt6::Core Qt6::Widgets)
# Link Ripes library
target_link_libraries(${APP_NAME} PUBLIC #${RIPES_LIB}
                                  #${NEURON_LIB}
                                  #version_lib
                                  SystemC::systemc
                                  Qt6::Core Qt6::Widgets
                                  QCustomPlot  Qt6::PrintSupport
)

if(${CMAKE_SYSTEM_NAME} STREQUAL "Emscripten")
    # https://doc.qt.io/qt-6/wasm.html#asyncify
##    target_link_options(${RIPES_LIB} PUBLIC -sASYNCIFY -Os)
    target_link_options(${APP_NAME} PUBLIC -sASYNCIFY -Os)
endif()

link_directories(
  ${NEURON_LIB}
  ../3rdParty/qcustomplot/
    ${SystemC_LIBRARY_DIRS}
    /opt/systemc/lib
)

include(Summary)

if(UNIX AND NOT APPLE) #Define the LINUX variable before testing it
    set(LINUX TRUE)
endif()

if(${LINUX})
    install(TARGETS ${APP_NAME}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
elseif(${APPLE})
    install(TARGETS ${APP_NAME}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        BUNDLE  DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/appdir/usr/
    DESTINATION ${CMAKE_INSTALL_PREFIX}
)
