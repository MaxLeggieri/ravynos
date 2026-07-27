# Mach-O prefixes every symbol with an underscore, so the ELF-style export
# list resolves to nothing on a Darwin host (every entry comes back as an
# <initial-undefines> link error). Use the underscored list there.
if(APPLE)
  set(LLVM_EXPORTED_SYMBOL_FILE ${CMAKE_CURRENT_SOURCE_DIR}/libtapi.exports.darwin)
else()
  set(LLVM_EXPORTED_SYMBOL_FILE ${CMAKE_CURRENT_SOURCE_DIR}/libtapi.exports)
endif()
set(LLVM_LINK_COMPONENTS
  Support
  )

add_tapi_library(libtapi
  SHARED
  APIVersion.cpp
  LinkerInterfaceFile.cpp
  Version.cpp
  ${TAPI_CXX_API_HEADERS}

  LINK_LIBS
  tapiCore
  tapiNoInits

  NO_INSTALL_RPATH
  )

set_target_properties(libtapi
  PROPERTIES
  OUTPUT_NAME "tapi"
  )

set(DYLIB_VERSION "${LLVM_VERSION_MAJOR}.${LLVM_VERSION_MINOR}.${LLVM_VERSION_PATCH}")
set_property(TARGET libtapi APPEND_STRING
  PROPERTY
  LINK_FLAGS " -current_version ${DYLIB_VERSION} -compatibility_version 1 -allowable_client ld"
  )

if(LINKER_SUPPORTS_NO_INITS)
  set_property(TARGET libtapi APPEND_STRING
    PROPERTY
    LINK_FLAGS " -Wl,-no_inits"
    )
endif()
