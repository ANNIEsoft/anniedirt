#
# cd /annie/app/users/rhatcher/tank_flux/
# mkdir G4ANNIE-cmakebuild
# cd G4ANNIE-cmakebuild
#
#
# WHICH_G4=`ls -d ${GEANT4_FQ_DIR}/lib64/Geant4-*`
# WHICH_CLHEP=`ls -d ${CLHEP_LIB_DIR}/CLHEP-*`
# 
# cmake -DGeant4_DIR=${WHICH_G4} -DCLHEP_DIR=${WHICH_CLHEP}  
#    [ -DWITH_G4NU=ON ] ../G4ANNIE
#
#
# Required environment variables:
#    ROOTSYS    e.g. /nusoft/app/externals/root/v5_34_01/Linux64bit+2.6-2.5-nu-e2-debug
#    XERCESROOT e.g. /nusoft/app/externals/xerces_c/v3_1_1/Linux64bit+2.6-2.5-e2-debug
#
# if WITH_G4NU G4NuPhysicsLists, then ${G4NU} must exist
#
#----------------------------------------------------------------------------
# Setup the project
cmake_minimum_required(VERSION 2.6 FATAL_ERROR)
project(G4ANNIE)

option(WITH_G4NU "build against G4NuPhysicsLists" OFF)

#----------------------------------------------------------------------------
# Find Geant4 package, activating all available UI and Vis drivers by default
# You can set WITH_GEANT4_UIVIS to OFF via the command line or ccmake/cmake-gui
# to build a batch mode only executable

if(DEFINED ENV{GEANT4_PATH})
  set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} $ENV{GEANT4_PATH})
  message(STATUS "Looking for GEANT4 in = ${CMAKE_PREFIX_PATH}")
endif()
#
option(WITH_GEANT4_UIVIS "Build example with Geant4 UI and Vis drivers" ON)
if(WITH_GEANT4_UIVIS)
  find_package(Geant4 REQUIRED ui_all vis_all)
else()
  find_package(Geant4 REQUIRED)
endif()

find_package(CLHEP REQUIRED)

#----------------------------------------------------------------------------
# Setup Geant4 include directories and compile definitions
# Setup include directory for this project
#
message(STATUS "using Geant4: ${Geant4_USE_FILE}")
include(${Geant4_USE_FILE})

include_directories(${PROJECT_SOURCE_DIR}/include)
include_directories($ENV{XERCESCROOT}/include)
include_directories($ENV{ROOTSYS}/include)
include_directories($ENV{GEANT4_FQ_DIR}/include)

include_directories(${PROJECT_SOURCE_DIR}/include)

include_directories(${G4NUPL}/include)

## ============== replace ROOT version for using later version of genie

if(DEFINED ENV{ROOT_PATH})
  list(APPEND CMAKE_PREFIX_PATH $ENV{ROOTSYS})
  #message(STATUS "looking for root in ${CMAKE_MODULE_PATH}")
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} $ENV{ROOT_PATH})
  message(STATUS "looking for root in ${CMAKE_MODULE_PATH}")
endif()
find_package(ROOT CONFIG REQUIRED Geom EGPythia6)
#find_package(ROOT MODULE REQUIRED)
if(NOT ROOT_FOUND)
  message(STATUS "ROOT package not found.")  
  return()
else()
  message(STATUS "found ROOT at: ${ROOT_USE_FILE}")
endif()

## Load macros: need to compile ROOT through cmake first and execute bin/thisroot.sh
include(${ROOT_USE_FILE})

#----------------------------------------------------------------------------
# GENIE stuff
#

include_directories($ENV{GENIE}/src)
link_directories($ENV{GENIE}/lib)

set(_genie_internal_libraries
    GAlgorithm GBaryonResonance GBase GBodekYang GCharm GCoh GDfrc GDIS GCrossSections GDecay GElas GElFF GHEP GEVGCore GEVGModules GEVGDrivers GGiBUU GHadronTransp GFragmentation GInteraction GLlewellynSmith GMEC GMessenger GNuGamma GNuE GNtuple GNuclear GNumerical GQPM GPDG GPDF GQEL GRES GRegistry GUtils GGeo GFluxDrivers GVLE GMuELoss GReWeight GNucleonDecay
)

include_directories($ENV{LOG4CPP_FQ_DIR}/include)
link_directories($ENV{LOG4CPP_FQ_DIR}/lib)

set(_log4cpp_internal_libraries log4cpp)

link_directories($ENV{XERCESCROOT}/lib)
link_directories($ENV{LIBXML2ROOT}/lib)

set(_xerces_internal_libraries xerces-c xml2)

set(Genie_LIBRARIES ${_genie_internal_libraries} ${_log4cpp_internal_libraries} ${_xerces_internal_libraries} )

#----------------------------------------------------------------------------
# Locate sources and headers for this project
# NB: headers are included so they will show up in IDEs
#
file(GLOB sources ${PROJECT_SOURCE_DIR}/src/*.cc)
file(GLOB headers ${PROJECT_SOURCE_DIR}/include/*.hh)

if(WITH_G4NU)
   set(G4NU $ENV{G4NU})
   include_directories(${G4NU}/include)
   link_directories(${G4NU}/lib)
   set(G4NU_LIBRARIES g4nupl)
   list(REMOVE_ITEM sources ${PROJECT_SOURCE_DIR}/src/AltG4PhysListFactory.cc)
endif()


#----------------------------------------------------------------------------
# Add the executable, and link it to the Geant4 libraries
#
add_executable(g4annie_dirt_flux g4annie_dirt_flux.cc ${sources} ${headers})
target_link_libraries(g4annie_dirt_flux ${Geant4_LIBRARIES} ${Genie_LIBRARIES} ${ROOT_LIBRARIES} ${Root_LIBRARIES} ${G4NU_LIBRARIES})

if(WITH_G4NU)
  SET_TARGET_PROPERTIES(g4annie_dirt_flux PROPERTIES COMPILE_FLAGS -DWITH_G4NU )
endif()

#----------------------------------------------------------------------------
# Copy all scripts to the build directory, i.e. the directory in which we
# build B1. This is so that we can run the executable directly because it
# relies on these scripts being in the current working directory.

set(G4DIRTSCRIPTS
  src/annie_v04.gdml
  )

foreach(_script ${G4DIRTSCRIPTS})
  configure_file(
    ${PROJECT_SOURCE_DIR}/${_script}
    ${PROJECT_BINARY_DIR}/${_script}
    COPYONLY
    )
endforeach()

#----------------------------------------------------------------------------
# For internal Geant4 use - but has no effect if you build this
# example standalone
#
add_custom_target(G4SA DEPENDS g4annie_dirt_flux)

#----------------------------------------------------------------------------
# Install the executable to 'bin' directory under CMAKE_INSTALL_PREFIX
#
install(TARGETS g4annie_dirt_flux DESTINATION bin)


