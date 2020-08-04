CMAKE_MINIMUM_REQUIRED(VERSION 3.0)

add_definitions(-DWITH_NUOPC)
# MOM6 only uses Fortran, so make sure the chosen compiler is set in $ENV
foreach(env_var IN ITEMS CMAKE_Fortran_COMPILER)
  if ( NOT DEFINED ENV{${env_var}})
    message(FATAL_ERROR "${env_var} is not defined")
  endif()
endforeach()
set(CMAKE_Fortran_COMPILER $ENV{CMAKE_Fortran_COMPILER})

# GRAB SPECIAL MODULES
LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")

PROJECT(mom6
        LANGUAGES Fortran)

# NEED TO KNOW WHICH COMPILER TO USE
if(NOT DEFINED CMAKE_Fortran_COMPILER_ID)
    message(STATUS "CMAKE_Fortran_COMPILER_ID is not set. CMake will use default options")
else()
    MESSAGE(STATUS "Loading ${CMAKE_Fortran_COMPILER_ID}.cmake")
    include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/${CMAKE_Fortran_COMPILER_ID}.cmake)
endif()


#ESMF, MPI and NetCDF required
FIND_PACKAGE(ESMF MODULE REQUIRED)
FIND_PACKAGE(MPI REQUIRED)
FIND_PACKAGE(NetCDF REQUIRED C Fortran)
INCLUDE_DIRECTORIES(${NETCDF_INCLUDE_DIRS})

# FMS REQUIRED
# FMS INSTALL DIRECTORY NEEDS TO BE PROVIDED BY THE USER IN THE COMMAND LINE
IF(NOT FMS_INSTALL_DIR)
    MESSAGE(FATAL_ERROR "Location of FMS install is required. Please set with -DFMS_INSTALL_DIR=<path>")
ELSE()
    FIND_PATH(FMS_LIB "libfms.a"
        PATHS
            ${FMS_INSTALL_DIR}
    )
    IF(NOT FMS_LIB)
        MESSAGE(FATAL_ERROR "Can't find libfms.a")
    ELSE()
        MESSAGE(STATUS "Found FMS")
        INCLUDE_DIRECTORIES(${FMS_INSTALL_DIR})
    ENDIF()
ENDIF()

# LETS GET THE HEADERS NEEDED FOR THE LIBRARY
INCLUDE_DIRECTORIES(src/MOM6/src/framework)
INCLUDE_DIRECTORIES(src/MOM6/config_src/dynamic)

file(GLOB_RECURSE lib_ocean_list RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "src/MOM6/src/*.*90")
# HARD CODE THE SYMLINKS AS GLOB_RECURSE DOES NOT FOLLOW SYMLINKS
list(APPEND lib_ocean_list

 src/MOM6/pkg/CVMix-src/src/shared/cvmix_background.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_convection.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_ddiff.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_kinds_and_types.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_kpp.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_math.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_put_get.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_shear.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_tidal.F90
 src/MOM6/pkg/CVMix-src/src/shared/cvmix_utils.F90
 src/MOM6/pkg/geoKdTree/kdtree.f90
 src/MOM6/pkg/MOM6_DA_hooks/src/core/ocean_da_core.F90
 src/MOM6/pkg/MOM6_DA_hooks/src/core/ocean_da_types.F90
 src/MOM6/pkg/MOM6_DA_hooks/src/core/write_ocean_obs.F90

)

MESSAGE(STATUS "FILES: ${lib_ocean_list}")
file(GLOB_RECURSE nuopc_driver_list RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "src/MOM6/config_src/nuopc_driver/*.F90")

IF(WITH_NUOPC)
    ADD_LIBRARY(
        mom

        "${lib_ocean_list};${nuopc_driver_list}"
    )
    SET_TARGET_PROPERTIES(mom PROPERTIES ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    SET_TARGET_PROPERTIES(mom PROPERTIES Fortran_MODULE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/mod)
    TARGET_LINK_LIBRARIES(mom PRIVATE esmf)
    INSTALL(TARGETS mom
        LIBRARY DESTINATION lib
        PUBLIC_HEADER DESTINATION include)
ELSE()
    ADD_LIBRARY(
        _ocean

        "${lib_ocean_list}"
    )
    SET_TARGET_PROPERTIES(_ocean PROPERTIES ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    SET_TARGET_PROPERTIES(_ocean PROPERTIES Fortran_MODULE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/mod)
    INSTALL(TARGETS _ocean
        LIBRARY DESTINATION lib
        PUBLIC_HEADER DESTINATION include)
ENDIF()
