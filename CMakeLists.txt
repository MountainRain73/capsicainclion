cmake_minimum_required(VERSION 3.20)
project(capsicain
    VERSION 99
    DESCRIPTION "Keyboard configuration tool that re-maps keys and modifier-key-combos"
    LANGUAGES CXX RC
)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

file(GLOB SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.rc
)

add_executable(capsicain ${SOURCES})

target_include_directories(capsicain PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)

target_precompile_headers(capsicain PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src/pch.h
)

target_compile_definitions(capsicain PRIVATE
    _CONSOLE
    WIN32
)

if(MSVC)
    target_compile_options(capsicain PRIVATE
        /MP
        /utf-8
    )
    target_link_options(capsicain PRIVATE
        /SUBSYSTEM:CONSOLE
    )
else()
    target_compile_options(capsicain PRIVATE
        -mconsole
    )
endif()

target_link_libraries(capsicain PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src/interception.lib
    kernel32
    user32
    gdi32
    winspool
    comdlg32
    advapi32
    shell32
    ole32
    oleaut32
    uuid
    odbc32
    odbccp32
)

add_custom_command(TARGET capsicain POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_CURRENT_SOURCE_DIR}/src/interception.dll
        $<TARGET_FILE_DIR:capsicain>/interception.dll
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_CURRENT_SOURCE_DIR}/src/capsicain.ini
        $<TARGET_FILE_DIR:capsicain>/capsicain.ini
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_CURRENT_SOURCE_DIR}/src/capsicain.example.ini
        $<TARGET_FILE_DIR:capsicain>/capsicain.example.ini
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_CURRENT_SOURCE_DIR}/src/capsicain.kingcon.ini
        $<TARGET_FILE_DIR:capsicain>/capsicain.kingcon.ini
    COMMENT "Copying runtime dependencies..."
)
