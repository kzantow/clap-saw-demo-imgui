cmake_minimum_required(VERSION 3.15)
cmake_policy(SET CMP0091 NEW) # use CMAKE_MSVC_RUNTIME_LIBRARY

set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
if (APPLE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET 10.11 CACHE STRING "Build for 10.1")
endif()
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "CMAKE_BUILD_TYPE Type Unspecified; picking Release")
    # set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "" FORCE)
endif()

project(clap-saw-demo-imgui VERSION 0.9.0 LANGUAGES C CXX)

set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD 17)




#if (NOT ELEMENTS_ROOT)
#   message(FATAL_ERROR "ELEMENTS_ROOT is not set")
#endif()

# Make sure ELEMENTS_ROOT is an absolute path to add to the CMake module path
#get_filename_component(ELEMENTS_ROOT "${ELEMENTS_ROOT}" ABSOLUTE)
#set (CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${ELEMENTS_ROOT}/cmake")

# If we are building outside the project, you need to set ELEMENTS_ROOT:
set(ELEMENTS_ROOT ${CMAKE_SOURCE_DIR}/libs/elements)
set(ELEMENTS_BUILD_EXAMPLES OFF)
add_subdirectory(${ELEMENTS_ROOT} elements)

#set(ELEMENTS_APP_PROJECT "SpriteSlidersAndKnobs")
#set(ELEMENTS_APP_TITLE "Sprite Sliders And Knobs")
#set(ELEMENTS_APP_COPYRIGHT "Copyright (c) z.prime")
#set(ELEMENTS_APP_ID "com.zeek.sprite-sliders-and-knobs")
#set(ELEMENTS_APP_VERSION "1.0")

#set(ELEMENTS_APP_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/main.cpp)
#set(ELEMENTS_APP_RESOURCES
#   ${CMAKE_CURRENT_SOURCE_DIR}/resources/knob_sprites_white_128x128.png
#   ${CMAKE_CURRENT_SOURCE_DIR}/resources/slider-white.png
#)

#include(ElementsConfigApp)





if (APPLE)
    option(USE_SANITIZER "Build and link with ASAN" FALSE)
endif()

# Copy on mac (could expand to other platforms)
option(COPY_AFTER_BUILD "Copy the clap to ~/Library on MACOS, ~/.clap on linux" FALSE)

add_subdirectory(libs/clap EXCLUDE_FROM_ALL)
add_subdirectory(libs/clap-helpers EXCLUDE_FROM_ALL)
add_subdirectory(libs/readerwriterqueue EXCLUDE_FROM_ALL)

#set(IMGUI_SOURCE_DIR ${CMAKE_SOURCE_DIR}/libs/imgui)
#add_subdirectory(libs/clap-imgui-support EXCLUDE_FROM_ALL)
#set(ELEMENTS_ROOT ${CMAKE_SOURCE_DIR}/libs/elements)
#add_subdirectory(${ELEMENTS_ROOT}/lib EXCLUDE_FROM_ALL)

#set(HEADER_FILES src/clap-saw-demo-editor.h src/clap-saw-demo.h)

file(GLOB_RECURSE SOURCE_FILES 
	src/*.cpp
)

file(GLOB_RECURSE HEADER_FILES
	src/*.h
)

#source_group("Source Files\\plugin"
source_group(
   TREE src
   PREFIX "Source Files\\plugin"
   FILES ${SOURCE_FILES}
)

source_group("Header Files\\plugin"
   FILES ${HEADER_FILES}
)

source_group("Header Files\\lib"
   REGULAR_EXPRESSION .*hpp
)

add_library(${PROJECT_NAME} MODULE
	src/clap-saw-demo.cpp
	src/clap-saw-demo-editor.cpp
	src/saw-voice.cpp
        src/clap-saw-demo-pluginentry.cpp 
	${HEADER_FILES}
)

target_include_directories(${PROJECT_NAME} PRIVATE src)

target_link_libraries(${PROJECT_NAME} clap-core clap-helpers readerwriterqueue elements)

if(APPLE)
    set_target_properties(${PROJECT_NAME} PROPERTIES
            BUNDLE True
            BUNDLE_EXTENSION clap
            MACOSX_BUNDLE_GUI_IDENTIFIER org.surge-synth-team.${PROJECT_NAME}
            MACOSX_BUNDLE_BUNDLE_NAME ${PROJECT_NAME}
            MACOSX_BUNDLE_BUNDLE_VERSION "0.1"
            MACOSX_BUNDLE_SHORT_VERSION_STRING "0.1"
            MACOSX_BUNDLE_INFO_PLIST ${CMAKE_SOURCE_DIR}/cmake/${PROJECT_NAME}.plist.in
            )
    target_link_libraries(${PROJECT_NAME} "-framework CoreFoundation" "-framework AppKit" "-framework CoreGraphics")

    target_compile_definitions(${PROJECT_NAME} PRIVATE IS_MAC=1)
    target_compile_options(${PROJECT_NAME} PRIVATE
            -Werror
            $<$<BOOL:${USE_SANITIZER}>:-fsanitize=address>
            $<$<BOOL:${USE_SANITIZER}>:-fsanitize=undefined>
            )
    target_link_options(${PROJECT_NAME} PRIVATE
            $<$<BOOL:${USE_SANITIZER}>:-fsanitize=address>
            $<$<BOOL:${USE_SANITIZER}>:-fsanitize=undefined>
            )

    if (${COPY_AFTER_BUILD})
        message(STATUS "Will copy plugin after every build" )
        set(products_folder ${CMAKE_BINARY_DIR})
        add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E echo "Installing ${products_folder}/${PROJECT_NAME}.clap to ~/Library/Audio/Plug-Ins/CLAP/"
                COMMAND ${CMAKE_COMMAND} -E make_directory "~/Library/Audio/Plug-Ins/CLAP"
                COMMAND ${CMAKE_COMMAND} -E copy_directory "${products_folder}/${PROJECT_NAME}.clap" "~/Library/Audio/Plug-Ins/CLAP/${PROJECT_NAME}.clap"
                )
    endif()
elseif(UNIX)
    target_compile_definitions(${PROJECT_NAME} PRIVATE IS_LINUX=1)
    target_sources(${PROJECT_NAME} PRIVATE src/linux-vstgui-adapter.cpp)
    set_target_properties(${PROJECT_NAME} PROPERTIES SUFFIX ".clap" PREFIX "")
    if (${COPY_AFTER_BUILD})
        message(STATUS "Will copy plugin after every build" )
        set(products_folder ${CMAKE_BINARY_DIR})
        add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E echo "Installing ${products_folder}/${PROJECT_NAME}.clap to ~/.clap"
                COMMAND ${CMAKE_COMMAND} -E make_directory "~/.clap"
                COMMAND ${CMAKE_COMMAND} -E copy "${products_folder}/${PROJECT_NAME}.clap" "~/.clap"
                )
    endif()

else()
    target_link_libraries(${PROJECT_NAME} d3d12.lib dxgi.lib d3dcompiler.lib dxguid.lib)
    target_compile_definitions(${PROJECT_NAME} PRIVATE IS_WIN=1)
    set_target_properties(${PROJECT_NAME} PROPERTIES SUFFIX ".clap" PREFIX "")
endif()
