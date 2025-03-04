add_library(wolf_runner)
add_library(wolf::runner ALIAS wolf_runner)

##############################
# External deps via GIT
####

# Formatting library
FetchContent_Declare(
        fmtlib
        GIT_REPOSITORY https://github.com/fmtlib/fmt.git
        GIT_TAG 9.1.0)
FetchContent_MakeAvailable(fmtlib) # Download and build
target_link_libraries_system(wolf_runner PUBLIC fmt::fmt-header-only)

FetchContent_Declare(
        eventbus
        GIT_REPOSITORY https://github.com/DeveloperPaul123/eventbus
        GIT_TAG 0.10.1
)
set(EVENTBUS_BUILD_TESTS OFF)
FetchContent_MakeAvailable(eventbus)
target_link_libraries_system(wolf_runner PUBLIC dp::eventbus)

# HTTP server library
FetchContent_Declare(
        simplewebserver
        GIT_REPOSITORY https://gitlab.com/eidheim/Simple-Web-Server.git
        GIT_TAG bdb1057)
FetchContent_MakeAvailable(simplewebserver)
target_link_libraries_system(wolf_runner PUBLIC simple-web-server)

# TOML parser (config file)
FetchContent_Declare(
        toml
        GIT_REPOSITORY https://github.com/ToruNiina/toml11.git
        GIT_TAG v3.7.1)
set(TOML11_COLORIZE_ERROR_MESSAGE, ON)
set(TOML11_PRESERVE_COMMENTS_BY_DEFAULT, ON)
FetchContent_MakeAvailable(toml)
target_link_libraries_system(wolf_runner PUBLIC toml11::toml11)

# Enet
set(CMAKE_PROJECT_INCLUDE_BEFORE "${PROJECT_SOURCE_DIR}/cmake/EnableCMP0048.cmake")
FetchContent_Declare(
        enet
        GIT_REPOSITORY https://github.com/cgutman/enet
        GIT_TAG 47e42dbf422396ce308a03b5a95ec056f0f0180c)
FetchContent_MakeAvailable(enet)
target_link_libraries_system(wolf_runner PUBLIC enet)
unset(CMAKE_PROJECT_INCLUDE_BEFORE)

##############################

find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED)

find_package(Boost
        REQUIRED COMPONENTS
        log
        stacktrace_basic)
include_directories(${Boost_INCLUDE_DIRS})

### Boost stacktrace exception handling
# adapted from https://github.com/jschueller/boost-stacktrace-example/blob/master/CMakeLists.txt
# also see: https://www.boost.org/doc/libs/develop/doc/html/stacktrace/configuration_and_build.html
find_library(BOOST_STACKTRACE_BACKTRACE_LIBRARY NAMES boost_stacktrace_backtrace)
if (UNIX)
    message(STATUS "Using Boost::stacktrace/addr2line")
    add_definitions(-D_GNU_SOURCE=1)
    target_compile_definitions(wolf_runner PRIVATE BOOST_STACKTRACE_USE_ADDR2LINE)
    target_link_libraries(wolf_runner PRIVATE dl)
elseif (MINGW AND BOOST_STACKTRACE_BACKTRACE_LIBRARY)
    message(STATUS "Using Boost::stacktrace/backtrace")
    target_compile_definitions(wolf_runner PRIVATE BOOST_STACKTRACE_USE_BACKTRACE)
    target_link_libraries(wolf_runner PRIVATE boost_stacktrace_backtrace backtrace)
else ()
    message(STATUS "Using Boost::stacktrace/basic")
endif ()

target_link_libraries(
        wolf_runner PUBLIC
        wolf::moonlight
        wolf::streaming
        wolf::input
        wolf::helpers
        wolf::audio
        wolf::docker
        ${Boost_LIBRARIES}
        ${CMAKE_DL_LIBS}
        ${CMAKE_THREAD_LIBS_INIT})

target_include_directories(wolf_runner PUBLIC .)

file(GLOB HEADER_LIST CONFIGURE_DEPENDS */*.hpp)
file(GLOB SRC_LIST SRCS control/*.cpp rest/*.cpp rtp/*.cpp rtsp/*.cpp runners/*.cpp state/*.cpp)

if (UNIX AND NOT APPLE)
    pkg_check_modules(LIBDRM REQUIRED IMPORTED_TARGET libdrm)
    pkg_check_modules(LIBPCI REQUIRED IMPORTED_TARGET libpci)

    list(APPEND SRC_LIST platforms/hw_linux.cpp)
    target_link_libraries(
            wolf_runner
            PUBLIC
            PkgConfig::LIBDRM
            PkgConfig::LIBPCI
    )
else ()
    list(APPEND SRC_LIST platforms/hw_unknown.cpp)
endif ()

target_sources(wolf_runner
        PRIVATE
        ${SRC_LIST}

        PUBLIC
        ${HEADER_LIST})

## Make files includable
function(make_includable input_file output_file)
    file(READ ${input_file} content)
    set(delim "for_c++_include")
    set(content "R\"${delim}(\n${content})${delim}\"")
    file(WRITE ${output_file} "${content}")
    message(STATUS "Generating file: ${output_file}")
endfunction(make_includable)

make_includable(rest/html/pin.html rest/html/pin.include.html)
make_includable(state/default/config.v2.toml state/default/config.include.toml)

# All users of this library will need at least C++17
target_compile_features(wolf_runner PUBLIC cxx_std_17)

####################
# Executable
add_executable(wolf wolf.cpp)
target_link_libraries(wolf wolf::runner
        #-static # enable to statically link all libraries
        )
target_compile_features(wolf PRIVATE cxx_std_17)