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

install(FILES ${ALL_HEADERS} DESTINATION "include")
install(TARGETS ${TARGET_LIBRARIES} DESTINATION "lib")

if(BUILD_UNIT_TESTS)
message(STATUS "Building with unit test support.")
enable_testing()
add_subdirectory(test)
endif(BUILD_UNIT_TESTS)
