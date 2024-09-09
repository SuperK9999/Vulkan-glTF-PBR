cmake_minimum_required(VERSION 3.4.1 FATAL_ERROR)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

set(NAME Vulkan-glTF-PBR)

project(${NAME})

include_directories(external)
include_directories(external/glm)
include_directories(external/gli)
include_directories(external/imgui)
include_directories(external/tinygltf)
include_directories(external/basisu/transcoder)
include_directories(external/basisu/zstd)
include_directories(external/optick/src)
include_directories(base)

# -------------------------------------------------------------------------------------------
# Optick Core
# -------------------------------------------------------------------------------------------
file(GLOB OPTICK_SRC "external/optick/src/*.*")
source_group("OptickCore" FILES ${OPTICK_SRC})

# OptickCore
add_library(OptickCore SHARED ${OPTICK_SRC})
target_include_directories(OptickCore
	PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/src>
)
set_target_properties(OptickCore 
	PROPERTIES 
		PUBLIC_HEADER "${CMAKE_CURRENT_LIST_DIR}/src/optick.h;${CMAKE_CURRENT_LIST_DIR}/src/optick.config.h"
		DEBUG_POSTFIX d # So that we can install debug and release side by side
)
target_compile_definitions(OptickCore PRIVATE OPTICK_EXPORTS=1)
if(OPTICK_USE_VULKAN)
	message(STATUS "Optick uses Vulkan")
	find_package(Vulkan REQUIRED)
	target_link_libraries(OptickCore PRIVATE Vulkan::Vulkan)
else()
	target_compile_definitions(OptickCore PRIVATE OPTICK_ENABLE_GPU_VULKAN=0)
endif()
if(OPTICK_USE_D3D12)
	message(STATUS "Optick uses DirectX 12")
	target_link_libraries(OptickCore PRIVATE "d3d12.lib" "dxgi.lib")
else()
	target_compile_definitions(OptickCore PRIVATE OPTICK_ENABLE_GPU_D3D12=0)
endif()
if(OPTICK_USE_D3D12 OR OPTICK_USE_VULKAN)
	target_compile_definitions(OptickCore PRIVATE OPTICK_ENABLE_GPU=1)
else()
	target_compile_definitions(OptickCore PRIVATE OPTICK_ENABLE_GPU=0)
endif()

if(MSVC)
	# temporary solution to unblock C++17 users
	target_compile_definitions(OptickCore PRIVATE _SILENCE_ALL_CXX17_DEPRECATION_WARNINGS)
endif()

set(EXTRA_LIBS ${EXTRA_LIBS} OptickCore)
if(NOT MSVC)
	set(EXTRA_LIBS ${EXTRA_LIBS} pthread)
endif()


# Gui App
if(MSVC AND OPTICK_BUILD_GUI_APP)
	add_subdirectory(gui)
endif()


# Console App
if(OPTICK_BUILD_CONSOLE_SAMPLE)
	file(GLOB TEST_ENGINE_SRC "samples/Common/TestEngine/*.*")
	source_group("TestEngine" FILES ${TEST_ENGINE_SRC})
	add_executable(ConsoleApp "samples/ConsoleApp/main.cpp" ${TEST_ENGINE_SRC})
	target_include_directories(ConsoleApp PRIVATE "samples/Common/TestEngine")
	target_link_libraries(ConsoleApp ${EXTRA_LIBS})
	set_target_properties(ConsoleApp PROPERTIES FOLDER Samples)
endif()


# -------------------------------------------------------------------------------------------
# Vulkan glTF-PBR
# -------------------------------------------------------------------------------------------

OPTION(USE_D2D_WSI "Build the project using Direct to Display swapchain" OFF)
OPTION(USE_D2D_WSI "Build the project using Direct to Display swapchain" OFF)
OPTION(USE_WAYLAND_WSI "Build the project using Wayland swapchain" OFF)

set(RESOURCE_INSTALL_DIR "" CACHE PATH "Path to install resources to (leave empty for running uninstalled)")

# Enable Draco mesh compression if source and library are present
IF(EXISTS ${CMAKE_SOURCE_DIR}/external/draco/draco_features.h)
  message(STATUS "Draco source and library found at " ${CMAKE_SOURCE_DIR}/external/draco)
  add_definitions(-DTINYGLTF_ENABLE_DRACO)
  include_directories(external/draco)
  find_library(DRACO_DECODER_LIBRARY NAMES dracodec PATHS ${CMAKE_SOURCE_DIR}/libs/draco REQUIRED)
  IF (EXISTS ${DRACO_DECODER_LIBRARY})
	message(STATUS "Draco decoder library found at " ${DRACO_DECODER_LIBRARY})
	message(STATUS "Draco mesh compression enabled")
  ELSE()
    message(FATAL_ERROR "Draco decoder library not found!")
  ENDIF()
ENDIF()

# Use FindVulkan module added with CMAKE 3.7
if (NOT CMAKE_VERSION VERSION_LESS 3.7.0)
	message(STATUS "Using module to find Vulkan")
	find_package(Vulkan)
endif()

IF(UNIX AND NOT APPLE)
    set(LINUX TRUE)
ENDIF()

IF(WIN32)
	IF (NOT Vulkan_FOUND)
		find_library(Vulkan_LIBRARY NAMES vulkan-1 vulkan PATHS ${CMAKE_SOURCE_DIR}/libs/vulkan)
		IF (Vulkan_LIBRARY)
			set(Vulkan_FOUND ON)
			MESSAGE("Using bundled Vulkan library version")
		ENDIF()
	ENDIF()
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DVK_USE_PLATFORM_WIN32_KHR")
ELSEIF(LINUX)
	IF (NOT Vulkan_FOUND)
		find_library(Vulkan_LIBRARY NAMES vulkan HINTS "$ENV{VULKAN_SDK}/lib" "${CMAKE_SOURCE_DIR}/libs/vulkan" REQUIRED)
		IF (Vulkan_LIBRARY)
			set(Vulkan_FOUND ON)
			MESSAGE("Using bundled Vulkan library version")
		ENDIF()
	ENDIF()
	find_package(Threads REQUIRED)
	IF(USE_D2D_WSI)
		MESSAGE("Using direct to display extension...")
		add_definitions(-D_DIRECT2DISPLAY)
	ELSEIF(USE_WAYLAND_WSI)
		message("Using wayland extension...")
		find_package(Wayland REQUIRED)
		if (NOT WAYLAND_FOUND)
			message(FATAL_ERROR "Wayland development package not found")
		endif ()
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DVK_USE_PLATFORM_WAYLAND_KHR")
		include_directories(${WAYLAND_INCLUDE_DIR})
	ELSE(USE_D2D_WSI)
		message("Using XCB extension...")
		find_package(XCB REQUIRED)
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DVK_USE_PLATFORM_XCB_KHR")
	ENDIF(USE_D2D_WSI)
ELSEIF(APPLE)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DVK_USE_PLATFORM_MACOS_MVK -DVK_EXAMPLE_XCODE_GENERATED")
	# Todo : android?
ENDIF(WIN32)

IF (NOT Vulkan_FOUND)
	message(FATAL_ERROR "Could not find Vulkan library!")
ELSE()
	message(STATUS ${Vulkan_LIBRARY})
ENDIF()

# Set preprocessor defines
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNOMINMAX -D_USE_MATH_DEFINES")

# Clang specific stuff
if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-switch-enum")
endif()

add_definitions(-D_CRT_SECURE_NO_WARNINGS)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

file(GLOB SOURCE *.cpp )

if(RESOURCE_INSTALL_DIR)
	add_definitions(-DVK_EXAMPLE_DATA_DIR=\"${RESOURCE_INSTALL_DIR}/\")
	install(DIRECTORY data/ DESTINATION ${RESOURCE_INSTALL_DIR}/)
else()
	add_definitions(-DVK_EXAMPLE_DATA_DIR=\"${CMAKE_SOURCE_DIR}/data/\")
endif()

# Compiler specific stuff
IF(MSVC)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc")
ELSEIF(APPLE)
	if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
		SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fobjc-arc -ObjC++")
	ELSE()
		SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fobjc-arc -xobjective-c++")
	ENDIF()
ENDIF(MSVC)

IF(WIN32)
	# Nothing here (yet)
ELSEIF(APPLE)
	link_libraries(${Vulkan_LIBRARY} "-framework AppKit" "-framework QuartzCore")
ELSE(WIN32)
	link_libraries(${XCB_LIBRARIES} ${Vulkan_LIBRARY} ${Vulkan_LIBRARY} ${WAYLAND_CLIENT_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
ENDIF(WIN32)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}")

add_subdirectory(base)
add_subdirectory(src)