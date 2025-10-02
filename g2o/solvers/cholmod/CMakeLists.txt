if(NOT G2O_USE_CHOLMOD)
  return()
endif()

# --- define an IMPORTED target for your prebuilt CHOLMOD ----------------------
# Adjust these two paths ONCE:
if(IOS)
set(G2O_PREBUILT_CHOLMOD_LIB "${PROJECT_SOURCE_DIR}/external/ios/libcholmod.a")
elseif(APPLE)
set(G2O_PREBUILT_CHOLMOD_LIB "${PROJECT_SOURCE_DIR}/external/mac/libcholmod.a")
elseif(UNIX)
set(G2O_PREBUILT_CHOLMOD_LIB "${PROJECT_SOURCE_DIR}/external/linux/libcholmod.a")
endif()
set(G2O_PREBUILT_SS_INCLUDE "${PROJECT_SOURCE_DIR}/external/linux/include")

if(NOT TARGET SuiteSparse::CHOLMOD)
  add_library(SuiteSparse::CHOLMOD STATIC IMPORTED)
  set_target_properties(SuiteSparse::CHOLMOD PROPERTIES
    IMPORTED_LOCATION "${G2O_PREBUILT_CHOLMOD_LIB}"
    INTERFACE_INCLUDE_DIRECTORIES "${G2O_PREBUILT_SS_INCLUDE}"
  )
endif()
# -----------------------------------------------------------------------------

set(PUBLIC_HEADERS
  linear_solver_cholmod.h
  cholmod_wrapper.h
  g2o_cholmod_api.h
)

add_library(solver_cholmod ${G2O_LIB_TYPE}
  cholmod_wrapper.cpp
  solver_cholmod.cpp
)

target_include_directories(solver_cholmod PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<INSTALL_INTERFACE:include/g2o/solvers/cholmod>
)

set_target_properties(solver_cholmod PROPERTIES
  OUTPUT_NAME ${LIB_PREFIX}solver_cholmod
  PUBLIC_HEADER "${PUBLIC_HEADERS}"
  VERSION ${G2O_LIB_VERSION}
  SOVERSION ${G2O_LIB_SOVERSION}
)

if(APPLE OR IOS)
  set_target_properties(solver_cholmod PROPERTIES
    INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}"
  )
endif()

# BLAS/LAPACK (Linux) or Accelerate (Apple)
if(IOS)
  message(WARNING "Not using Accelerate on IOS")
    target_link_libraries(solver_cholmod
    PUBLIC core
    PRIVATE SuiteSparse::CHOLMOD
    )
elseif(APPLE)
  find_library(ACCELERATE_FRAMEWORK Accelerate REQUIRED)
  target_link_libraries(solver_cholmod
    PUBLIC core
    PRIVATE SuiteSparse::CHOLMOD
    ${ACCELERATE_FRAMEWORK}
  )
elseif(UNIX)
  find_package(BLAS REQUIRED)
  find_package(LAPACK REQUIRED)
  target_link_libraries(solver_cholmod
    PUBLIC core
    PRIVATE SuiteSparse::CHOLMOD
    ${BLAS_LIBRARIES}
    ${LAPACK_LIBRARIES}
  )
endif()

install(TARGETS solver_cholmod
  EXPORT ${G2O_TARGETS_EXPORT_NAME}
  RUNTIME DESTINATION ${RUNTIME_DESTINATION}
  LIBRARY DESTINATION ${LIBRARY_DESTINATION}
  ARCHIVE DESTINATION ${ARCHIVE_DESTINATION}
  PUBLIC_HEADER DESTINATION ${INCLUDES_DESTINATION}/g2o/solvers/cholmod
)

# Create alias target for easier usage when included as subdirectory
add_library(g2o::solver_cholmod ALIAS solver_cholmod)
