include_directories(pch ${OPENSSL_INCLUDE_DIR})

set(SOURCES_COMMON
  http/client/http_client_msg.cpp
  http/client/x509_cert_utilities.cpp
  http/common/http_helpers.cpp
  http/common/http_msg.cpp
  http/listener/http_listener.cpp
  http/listener/http_listener_msg.cpp
  http/listener/http_server_api.cpp
  http/oauth/oauth1.cpp
  http/oauth/oauth2.cpp
  json/json.cpp
  json/json_parsing.cpp
  json/json_serialization.cpp
  pplx/pplx.cpp
  uri/uri.cpp
  uri/uri_builder.cpp
  uri/uri_parser.cpp
  utilities/asyncrt_utils.cpp
  utilities/base64.cpp
  utilities/web_utilities.cpp
  websockets/client/ws_msg.cpp
  websockets/client/ws_client.cpp
  websockets/client/ws_client_wspp.cpp
)

# THE ORDER OF FILES IS VERY /VERY/ IMPORTANT
if(UNIX)
  set(SOURCES
    ${SOURCES_COMMON}
    streams/fileio_posix.cpp
    pplx/threadpool.cpp
    http/client/http_client_asio.cpp
    http/listener/http_server_asio.cpp
  )
  if(APPLE)
    list(APPEND SOURCES
      pplx/pplxapple.cpp
      http/client/x509_cert_utilities_apple.cpp
    )
    find_library(COREFOUNDATION CoreFoundation "/")
    find_library(SECURITY Security "/")
    set(EXTRALINKS ${COREFOUNDATION} ${SECURITY})
  elseif(ANDROID)
    list(APPEND SOURCES
      http/client/x509_cert_utilities_android.cpp
      pplx/pplxlinux.cpp
    )
  else()
    list(APPEND SOURCES pplx/pplxlinux.cpp)
  endif()
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${WARNINGS} -Wno-error -pedantic")
elseif(WIN32)
  set(SOURCES
    ${SOURCES_COMMON}
    http/client/http_client_winhttp.cpp
    http/client/x509_cert_utilities_win32.cpp
    http/listener/http_server_httpsys.cpp
    pplx/pplxwin.cpp
    streams/fileio_win32.cpp
    pch/stdafx.cpp
  )
  set(EXTRALINKS
    bcrypt.lib
    crypt32.lib
    httpapi.lib
    Winhttp.lib
  )
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${WARNINGS} /Yustdafx.h /Zm200")
  set_source_files_properties(pch/stdafx.cpp PROPERTIES COMPILE_FLAGS "/Ycstdafx.h")
  if (BUILD_SHARED_LIBS)
    add_definitions(-D_ASYNCRT_EXPORT -D_PPLX_EXPORT -D_USRDLL)
  endif()
endif()

add_library(${Casablanca_LIBRARY} ${SOURCES})

target_link_libraries(${Casablanca_LIBRARY}
  ${CMAKE_THREAD_LIBS_INIT}
  ${Boost_SYSTEM_LIBRARY}
  ${Boost_THREAD_LIBRARY}
  ${Boost_ATOMIC_LIBRARY}
  ${Boost_CHRONO_LIBRARY}
  ${Boost_RANDOM_LIBRARY}
  ${EXTRALINKS}
  ${Boost_FRAMEWORK}
  ${OPENSSL_LIBRARIES}
  ${COREFOUNDATION}
  ${ANDROID_STL_FLAGS}
  )

# Portions specific to cpprest binary versioning.
set (CPPREST_VERSION_MAJOR 2)
set (CPPREST_VERSION_MINOR 7)
set (CPPREST_VERSION_REVISION 0)

if(WIN32)
  set_target_properties(${Casablanca_LIBRARY} PROPERTIES
    OUTPUT_NAME "${Casablanca_LIBRARY}_${CPPREST_VERSION_MAJOR}_${CPPREST_VERSION_MINOR}")
elseif(ANDROID)
  # Do not use SOVERSION on android. It is completely unsupported (and causes problems).
  # Perhaps revisit in the future? (NDK r9d, 8/7/14)
else()
  set_target_properties(${Casablanca_LIBRARY} PROPERTIES
    SOVERSION ${CPPREST_VERSION_MAJOR}.${CPPREST_VERSION_MINOR})

  install(
    TARGETS ${Casablanca_LIBRARY}
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    )
endif()

###############################################################################
# Installation settings
###############################################################################

set (casablanca_include_dir "include/cpprestsdk")
set (config_install_dir "lib/cmake/${PROJECT_NAME}")
set (targets_export_name "${PROJECT_NAME}Targets")


set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")

install(
    TARGETS ${Casablanca_LIBRARY}
 EXPORT "${targets_export_name}"    
 LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"    
    INCLUDES DESTINATION "${casablanca_include_dir}")
 
install(
 DIRECTORY "${PROJECT_SOURCE_DIR}/include/"
    DESTINATION "${casablanca_include_dir}"
 FILES_MATCHING PATTERN "*.*")

set(namespace "${PROJECT_NAME}::")
install(
    EXPORT "${targets_export_name}"
 NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}")
