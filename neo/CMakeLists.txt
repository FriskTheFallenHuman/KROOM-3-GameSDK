cmake_minimum_required(VERSION 2.6...3.22 FATAL_ERROR)

set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER  "")
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/CMakeModules")

project(Kroom3BFG-GameSDK
		VERSION "1.3.0"
		DESCRIPTION "Fork of RBDOOM3-BFG v1.3.0 source port aims to modernize DOOM 3 BFG"
		HOMEPAGE_URL "https://github.com/FriskTheFallenHuman/KROOM-3"
		LANGUAGES C CXX)

option(FORCE_COLOR_OUTPUT "Always produce ANSI-colored output (GNU/Clang only)." OFF)
option(COMPILE_COMMANDS "Generate compile_commands.json" OFF)
option(USE_PRECOMPILED_HEADERS "Use precompiled headers during build" ON)
option(USE_VULKAN "Use Vulkan instead of OpenGL" OFF)
option(ONATIVE "Optimize for the host CPU" OFF)
option(WINDOWS10 "Build for Windows 10" ON)
option(REPRODUCIBLE_BUILD "Replace __DATE__ and __TIME__ by hardcoded values for reproducible builds." OFF)
set(MOD_FOLDER "mod_example" CACHE STRING "Name of our mod")

include(CPUDefine)

if(UNIX)
	# SRS - Disable precompiled headers when cross-compiling on OSX, e.g. when creating universal binaries
	#     - Also disable precompiled headers when using the Ninja Multi-Config generator on linux or macOS
	if((CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "x86_64" AND CMAKE_OSX_ARCHITECTURES MATCHES "arm64" ) OR
	   (CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "arm64"  AND CMAKE_OSX_ARCHITECTURES MATCHES "x86_64") OR
	   (CMAKE_GENERATOR MATCHES "Ninja Multi-Config"))
		set(USE_PRECOMPILED_HEADERS OFF)
	endif()

	# SRS - For single-config dev builds, define NO_MULTI_CONFIG to search for basepath up 1 dir level vs 2
	if((CMAKE_GENERATOR MATCHES "Unix Makefiles") OR
	   (CMAKE_GENERATOR MATCHES "Ninja" AND NOT CMAKE_GENERATOR MATCHES "Multi-Config"))
		add_definitions(-DNO_MULTI_CONFIG)
	endif()
endif()

if(REPRODUCIBLE_BUILD)
	# don't use __DATE__ and __TIME__ macros so builds are reproducible
	add_definitions(-DID_REPRODUCIBLE_BUILD)
endif()

if(COMPILE_COMMANDS)
	set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
endif()

if (FORCE_COLOR_OUTPUT)
	if (CMAKE_COMPILER_IS_GNUCC)
		add_compile_options(-fdiagnostics-color=always)
	elseif(CMAKE_C_COMPILER_ID MATCHES "Clang")
		add_compile_options(-fcolor-diagnostics)
	endif ()
endif ()

macro(SET_FORCED_OPTION option value)
	set(${option} ${value} CACHE INTERNAL "" FORCE)
endmacro()

include(FetchContent)
macro(find_or_fetch PACKAGE_NAME)
	if(USE_SYSTEM_LIBS)
		find_package(${PACKAGE_NAME} REQUIRED)
		include_directories(${${PACKAGE_NAME}_INCLUDE_DIRS})
	else()
		find_package(${PACKAGE_NAME} QUIET)
		if(NOT ${PACKAGE_NAME}_FOUND)
			message(STATUS "Fetching ${PACKAGE_NAME}")
			FetchContent_Declare(
				${PACKAGE_NAME}
				${ARGN}
			)
			FetchContent_MakeAvailable(${PACKAGE_NAME})
			FetchContent_GetProperties(${PACKAGE_NAME})

			include_directories(${${PACKAGE_NAME}_SOURCE_DIR} ${${PACKAGE_NAME}_SOURCE_DIR}/include ${${PACKAGE_NAME}_BINARY_DIR}/include)
		endif()
	endif()
endmacro()

configure_file("${CMAKE_SOURCE_DIR}/config.h.in" "${CMAKE_BINARY_DIR}/config.h")

set(cpu_arch)
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(cpu_arch "x86_64")
elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
	set(cpu_arch "x86")
endif()
if(NOT MSVC)
	set(cpu_arch "${cpu}")
endif()

if(NOT MSVC)
	message(STATUS "Building ${CMAKE_BUILD_TYPE} for ${os}-${cpu}")
else()
	message(STATUS "Building ${CMAKE_BUILD_TYPE} for ${os}-${cpu_arch} with Visual Studio ${CMAKE_VS_VERSION}")
endif()

if(CMAKE_COMPILER_IS_GNUCC OR (CMAKE_C_COMPILER_ID MATCHES "Clang" AND NOT CMAKE_CXX_SIMULATE_ID STREQUAL "MSVC"))

	add_compile_options(-pipe)
	#add_compile_options(-Wall)
	add_compile_options(-Wno-error=format-security)
	add_compile_options(-Wno-error=format)
	add_compile_options(-Wno-format-zero-length)
	add_compile_options(-Wno-nonnull)

	# Compiler check (needs -std=c++17 flag)
	include(CheckCXXCompilerFlag)
	CHECK_CXX_COMPILER_FLAG("-std=c++17" COMPILER_SUPPORTS_CXX17)
	if(COMPILER_SUPPORTS_CXX17)
		set(CMAKE_CXX_STANDARD 17)
		set(CMAKE_CXX_STANDARD_REQUIRED ON)
		set(CMAKE_CXX_EXTENSIONS OFF)
	else()
		message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++17 support. Please use a different C++ compiler.")
	endif()

	if(NOT WIN32)
		CHECK_CXX_COMPILER_FLAG("-fvisibility=hidden" CXX_HAS_FVISIBILITY)
		if(NOT CXX_HAS_FVISIBILITY)
			message(FATAL_ERROR "Compiler does not support -fvisibility")
		endif()
		add_compile_options(-fvisibility=hidden)
	endif()

	if (CPU_OPTIMIZATION)
		SEPARATE_ARGUMENTS(CPU_OPTIMIZATION)
		add_compile_options(${CPU_OPTIMIZATION})
	endif()

	if(WIN32)
		# require msvcr70.dll or newer for _aligned_malloc etc
		# I think it is from Visual C++ .NET 2002, so it should be available on any remotely modern system.
		add_definitions(-D__MSVCRT_VERSION__=0x0700)

		if(MINGW)
			include_directories(extern/mingw-hacks)
		endif()
	endif()

	add_compile_options(-Wno-pragmas -Wno-unused-variable -Wno-switch -Wno-unused-value -Winvalid-pch -Wno-multichar)
	if(CMAKE_C_COMPILER_ID MATCHES "Clang")
		# add clang-specific settings for warnings (the second one make sure clang doesn't complain
		# about unknown -W flags, like -Wno-unused-but-set-variable)
		# SRS - Add -Wno-expansion-to-defined, Wno-nullability-completeness and -Wno-shorten-64-to-32 to list of warning settings
		add_compile_options(-Wno-local-type-template-args -Wno-unknown-warning-option -Wno-inline-new-delete -Wno-switch-enum -Wno-expansion-to-defined -Wno-nullability-completeness -Wno-shorten-64-to-32 -Wno-nontrivial-memcall)
	endif()

	if(NOT CMAKE_CROSSCOMPILING AND ONATIVE)
		if(CMAKE_SYSTEM_PROCESSOR MATCHES "((powerpc|ppc)64le)|(mips64)")
			add_compile_options(-mcpu=native)
		else()
			add_compile_options(-march=native)
		endif()
	endif()

	add_compile_options(-fno-strict-aliasing)

	# SRS - Make sure OSX can find system headers and add support for minimum OSX runtime version
	if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
		# SRS - Also add -fasm-blocks otherwise Xcode complains and -Qunused-arguments to silence MMX/SSE unused arg warnings when compiling for Apple arm64
		add_compile_options(-fasm-blocks -Qunused-arguments)
		if(CMAKE_OSX_SYSROOT)
			add_compile_options(-isysroot ${CMAKE_OSX_SYSROOT})
			message(STATUS "Using macOS sysroot: " ${CMAKE_OSX_SYSROOT})
		endif()
		if(CMAKE_OSX_DEPLOYMENT_TARGET)
			add_compile_options(-mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET})
		endif()
	endif()

	set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -ggdb -D_DEBUG -O0")
	#set(CMAKE_C_FLAGS_DEBUGALL "${CMAKE_C_FLAGS_DEBUGALL} -g -ggdb -D_DEBUG")
	#set(CMAKE_C_FLAGS_PROFILE "${CMAKE_C_FLAGS_PROFILE} -g -ggdb -D_DEBUG -O1 -fno-omit-frame-pointer")
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3 -ffp-contract=off -fomit-frame-pointer -fno-math-errno -fno-trapping-math")
	set(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} -g -O3 -ffp-contract=off -fomit-frame-pointer -fno-math-errno -fno-trapping-math")
	set(CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL} -Os -ffp-contract=off -fomit-frame-pointer -fno-math-errno -fno-trapping-math")

	set(CMAKE_CXX_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG})
	#set(CMAKE_CXX_FLAGS_DEBUGALL ${CMAKE_C_FLAGS_DEBUGALL})
	#set(CMAKE_CXX_FLAGS_PROFILE ${CMAKE_C_FLAGS_PROFILE})
	set(CMAKE_CXX_FLAGS_RELEASE ${CMAKE_C_FLAGS_RELEASE})
	set(CMAKE_CXX_FLAGS_RELWITHDEBINFO ${CMAKE_C_FLAGS_RELWITHDEBINFO})
	set(CMAKE_CXX_FLAGS_MINSIZEREL ${CMAKE_C_FLAGS_MINSIZEREL})

	if(NOT WIN32)
		set(sys_libs pthread ${CMAKE_DL_LIBS})

		if(NOT "${CMAKE_SYSTEM}" MATCHES "Darwin")
			list(APPEND sys_libs rt)
		endif()
	endif()

elseif(MSVC)

	set(CMAKE_CXX_STANDARD 17)
	set(CMAKE_CXX_STANDARD_REQUIRED True)

	# Suppress clang-cl warnings
	if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND CMAKE_CXX_SIMULATE_ID STREQUAL "MSVC")
		add_compile_options(-Wno-inline-new-delete)
		add_compile_options(-Wno-nontrivial-memcall)
		add_compile_options(-Wno-unused-variable)
		add_compile_options(-Wno-overloaded-virtual)
		add_compile_options(-Wno-local-type-template-args)
		add_compile_options(-Wno-unknown-warning-option)
		add_compile_options(-Wno-switch-enum)
		add_compile_options(-Wno-switch)
		add_compile_options(-Wno-unused-but-set-variable)
		add_compile_options(-Wno-expansion-to-defined)
		add_compile_options(-Wno-nullability-completeness)
		add_compile_options(-Wno-shorten-64-to-32)
		add_compile_options(-Wno-writable-strings)
		add_compile_options(-Wno-tautological-overlap-compare)
		add_compile_options(-Wno-inconsistent-missing-override)
		add_compile_options(-Wno-missing-braces)
		add_compile_options(-Wno-reorder-ctor)
		add_compile_options(-Wno-deprecated-non-prototype)
		add_compile_options(-Wno-unused-private-field) # TODO remove for cleanup
		add_compile_options(-Wno-unused-function)
		add_compile_options(-Wno-microsoft-exception-spec)
		add_compile_options(-Wno-pessimizing-move)
		add_compile_options(-Wno-undefined-bool-conversion)
		add_compile_options(-Wno-unused-local-typedef)
		add_compile_options(-fms-extensions)
		add_compile_options(-Wno-microsoft-include)
		string(REPLACE "/MP" "" CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
		string(REPLACE "/MP" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
		string(REPLACE "/MP" "" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
		string(REPLACE "/MP" "" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
		string(REPLACE "/MP" "" CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
		string(REPLACE "/MP" "" CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL}")
		string(REPLACE "/MP" "" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
		string(REPLACE "/MP" "" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
		string(REPLACE "/MP" "" CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
		string(REPLACE "/MP" "" CMAKE_CXX_FLAGS_MINSIZEREL "${CMAKE_CXX_FLAGS_MINSIZEREL}")
	else()
		set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /D _DEBUG /MP /MDd")
		set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MP /Oi /Oy /MD")
		set(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} /MP /Oi /MDd")
		set(CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL} /MP /Oi /Oy /MD")

		set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MP /MDd")
		set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MP /Oi /Oy /MD")
		set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} /MP /MDd")
		set(CMAKE_CXX_FLAGS_MINSIZEREL "${CMAKE_CXX_FLAGS_MINSIZEREL} /MP /Oi /Oy /MD")
	endif()

	# RB: without /SAFESEH:NO we can't link against ffmpeg libs and VS2013 or we have to build our own
	# libs for newer compilers
	set(CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG} /SAFESEH:NO")
	set(CMAKE_EXE_LINKER_FLAGS_MINSIZEREL "${CMAKE_EXE_LINKER_FLAGS_MINSIZEREL} /SAFESEH:NO")
	set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /SAFESEH:NO")
	set(CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO} /SAFESEH:NO")

	add_definitions(-DWIN32
					-D_WINDOWS
					-D_CRT_SECURE_NO_DEPRECATE
					-D_CRT_NONSTDC_NO_DEPRECATE
					-D_CRT_SECURE_NO_WARNINGS
					-D_MBCS)

	if( WINRT OR WINDOWS10 ) # Windows RT
		add_definitions(-DUSE_WINRT)
	endif()

	if(NOT CMAKE_CL_64)
		add_definitions(-D_USE_32BIT_TIME_T)
	endif()

	set(sys_libs dbghelp iphlpapi winmm wsock32 shlwapi ${CMAKE_DL_LIBS})

	if(MSVC)
		if(WINDOWS10)
			list(APPEND sys_libs dxguid)
		else()
			find_package(WindowsSDK REQUIRED)
			get_windowssdk_library_dirs("${WINDOWSSDK_LATEST_DIR}" WINSDK_LIB_DIRS)

			set(WINSDK_LIB_DIR "")
			foreach(_dir ${WINSDK_LIB_DIRS})
				if(EXISTS "${_dir}/dinput8.lib")
					set(WINSDK_LIB_DIR "${_dir}")
					break()
				endif()
			endforeach()

			list(APPEND sys_libs "${WINSDK_LIB_DIR}/dxguid.lib")
		endif()
	endif()
endif()

add_definitions(-DID_PC)

if (USE_INTRINSICS_SSE)
	add_definitions(-DUSE_INTRINSICS_SSE)
endif()

if(USE_VULKAN)
	add_definitions(-DUSE_VULKAN)
else()
	add_definitions(-DGLEW_STATIC)
endif()

##################### Dependencies ###########################

set(sys_thirdparty)
set(sys_thirdparty_dependencies)

if(UNIX)
	find_package(PkgConfig REQUIRED)
endif()

# CppTrace
SET_FORCED_OPTION(CPPTRACE_BUILD_TESTING OFF)
SET_FORCED_OPTION(CPPTRACE_BUILD_BENCHMARKING OFF)
SET_FORCED_OPTION(CPPTRACE_BUILD_TOOLS OFF)
SET_FORCED_OPTION(CPPTRACE_BUILD_SHARED OFF)

find_or_fetch(cpptrace
	GIT_REPOSITORY https://github.com/jeremy-rifkin/cpptrace.git
	GIT_TAG v1.0.4
	GIT_SHALLOW TRUE)

# SteamWorksSDK
if(USE_STEAMWORKS)
	add_definitions(-DUSE_STEAMWORKS)
endif()

include_directories(
	${CMAKE_BINARY_DIR}
	${CMAKE_SOURCE_DIR}
	${CMAKE_SOURCE_DIR}/idlib
	${CMAKE_SOURCE_DIR}/extern)

add_definitions(-D_D3SDK)
add_definitions(-D__DOOM__)
add_definitions(-D__DOOM_DLL__)

add_subdirectory(idlib)
add_subdirectory(d3xp)
add_subdirectory(extern)

# Group every FetchContent'd target under "third-party" in IDE solutions.
function(_collect_targets out dir)
	get_property(_targets DIRECTORY "${dir}" PROPERTY BUILDSYSTEM_TARGETS)
	get_property(_subdirs DIRECTORY "${dir}" PROPERTY SUBDIRECTORIES)
	foreach(_sub IN LISTS _subdirs)
		_collect_targets(_sub_targets "${_sub}")
		list(APPEND _targets ${_sub_targets})
	endforeach()
	set(${out} "${_targets}" PARENT_SCOPE)
endfunction()

_collect_targets(_all_targets "${CMAKE_SOURCE_DIR}")
foreach(_tgt IN LISTS _all_targets)
	get_target_property(_type ${_tgt} TYPE)
	if(_type STREQUAL "INTERFACE_LIBRARY")
		continue()
	endif()

	get_target_property(_dir ${_tgt} SOURCE_DIR)
	cmake_path(IS_PREFIX FETCHCONTENT_BASE_DIR "${_dir}" NORMALIZE _is_dep)
	if(_is_dep)
		#get_target_property(_folder ${_tgt} FOLDER)
		#if(_folder)
		#	set_target_properties(${_tgt} PROPERTIES FOLDER "third-party/${_folder}")
		#else()
		set_target_properties(${_tgt} PROPERTIES FOLDER "third-party")
		#endif()
	endif()
endforeach()