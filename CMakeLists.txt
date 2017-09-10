cmake_minimum_required(VERSION 3.1.0 FATAL_ERROR)
PROJECT(sioclient)

option(BUILD_SHARED_LIBS "Build the shared library" OFF)
option(BUILD_UNIT_TESTS  "Builds unit tests target" OFF)

set(MAJOR 1)
set(MINOR 6)
set(PATCH 0)

# On platforms where single-configuration generators are supported (e.g. Make) -
# Log what configuration we are building with. Users can change this either with
# regeneration or with 'cmake --build . --config <type>'
if(CMAKE_BUILD_TYPE)
MESSAGE(STATUS
    "Building in ${CMAKE_BUILD_TYPE} mode.\n"
    "To change this, reconfigure with -DCMAKE_BUILD_TYPE."
)
endif(CMAKE_BUILD_TYPE)

# Globally turning C++11 on
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

aux_source_directory(${CMAKE_CURRENT_LIST_DIR}/src ALL_SRC)
aux_source_directory(${CMAKE_CURRENT_LIST_DIR}/src/internal ALL_SRC)
file(GLOB ALL_HEADERS ${CMAKE_CURRENT_LIST_DIR}/src/*.h )
set(SIO_INCLUDEDIR ${CMAKE_CURRENT_LIST_DIR})

add_definitions(
    # These will force ASIO to compile without Boost
    -DBOOST_DATE_TIME_NO_LIB
    -DBOOST_REGEX_NO_LIB
    -DASIO_STANDALONE
    # These will force WebsocketPP to compile with C++11
    -D_WEBSOCKETPP_CPP11_STL_
    -D_WEBSOCKETPP_CPP11_FUNCTIONAL_
)

if(BUILD_SHARED_LIBS)
set(TARGET_TYPE SHARED)
else()
set(TARGET_TYPE STATIC)
endif(BUILD_SHARED_LIBS)

add_library(sioclient ${TARGET_TYPE} ${ALL_SRC})
target_include_directories(sioclient PRIVATE 
    ${CMAKE_CURRENT_LIST_DIR}/src 
    ${CMAKE_CURRENT_LIST_DIR}/lib/websocketpp 
    ${CMAKE_CURRENT_LIST_DIR}/lib/rapidjson/include
    ${CMAKE_CURRENT_LIST_DIR}/lib/asio/asio/include
)

if(BUILD_SHARED_LIBS)
set_target_properties(sioclient
	PROPERTIES
		SOVERSION ${MAJOR}
		VERSION ${MAJOR}.${MINOR}.${PATCH}
	)
endif(BUILD_SHARED_LIBS)
list(APPEND TARGET_LIBRARIES sioclient)

find_package(OpenSSL)
if(OPENSSL_FOUND)
add_library(sioclient_tls ${TARGET_TYPE} ${ALL_SRC})
target_include_directories(sioclient_tls PRIVATE
    ${CMAKE_CURRENT_LIST_DIR}/src 
    ${CMAKE_CURRENT_LIST_DIR}/lib/websocketpp 
    ${CMAKE_CURRENT_LIST_DIR}/lib/rapidjson/include
    ${CMAKE_CURRENT_LIST_DIR}/lib/asio/asio/include
    ${OPENSSL_INCLUDE_DIR}
)

target_link_libraries(sioclient_tls PRIVATE ${OPENSSL_LIBRARIES} )
target_compile_definitions(sioclient_tls PRIVATE -DSIO_TLS)
if(BUILD_SHARED_LIBS)
set_target_properties(sioclient_tls
	PROPERTIES
		SOVERSION ${MAJOR}
		VERSION ${MAJOR}.${MINOR}.${PATCH}
	)
endif(BUILD_SHARED_LIBS)
list(APPEND TARGET_LIBRARIES sioclient_tls)
endif(OPENSSL_FOUND)

set(EXPORT_NAME ${CMAKE_PROJECT_NAME})
install(FILES ${ALL_HEADERS} DESTINATION "include")
install(TARGETS ${TARGET_LIBRARIES} EXPORT ${EXPORT_NAME} DESTINATION "lib")
set(CONFIG_FILE "${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_NAME}Config.cmake")
configure_file("src/Config.cmake.in" "${CONFIG_FILE}" @ONLY)
set(VERSION_FILE "${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_NAME}ConfigVersion.cmake")
configure_file("src/ConfigVersion.cmake.in" "${VERSION_FILE}" @ONLY)
install(FILES ${CONFIG_FILE} ${VERSION_FILE} DESTINATION "cmake")
install(EXPORT ${EXPORT_NAME} DESTINATION "cmake")

if(BUILD_UNIT_TESTS)
message(STATUS "Building with unit test support.")
enable_testing()
add_subdirectory(test)
endif(BUILD_UNIT_TESTS)

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${CMAKE_PROJECT_NAME}")
set(CPACK_PACKAGE_VENDOR "${CMAKE_PROJECT_NAME}")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
set(CPACK_PACKAGE_VERSION_MAJOR "${MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${PATCH}")
if(WIN32)
set(CPACK_GENERATOR "ZIP")
else(WIN32)
set(CPACK_GENERATOR "TGZ")
endif(WIN32)
include(CPack)
