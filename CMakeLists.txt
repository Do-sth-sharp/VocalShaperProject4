cmake_minimum_required (VERSION 3.15)
project (vsp4 VERSION 1.0.0 LANGUAGES CXX)

if (NOT DEFINED CMAKE_BUILD_TYPE)
	set (CMAKE_BUILD_TYPE "Release"  CACHE STRING "Choose the type of build." FORCE)
endif (NOT DEFINED CMAKE_BUILD_TYPE)

set(CMAKE_CXX_STANDARD 20)# Using C++20
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 17)# Using C17
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_C_STANDARD_REQUIRED ON)

set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")# Using /MD and /MDd on MSVC

find_package (Protobuf CONFIG REQUIRED)

include (GNUInstallDirs)
include (CMakePackageConfigHelpers)

set (VSP4_INSTALL_NAME ${PROJECT_NAME})

# Directories
set (VSP4_PROTO_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/proto" CACHE STRING INTERNAL)
set (VSP4_PROTO_CXX_DIR "${CMAKE_CURRENT_SOURCE_DIR}/cxx" CACHE STRING INTERNAL)

# Get All Proto Files
file (GLOB_RECURSE VSP4_PROTO_SRC CONFIGURE_DEPENDS "${VSP4_PROTO_SRC_DIR}/*.proto")

# Proto Build Commands
set (VSP4_PROTO_CXX_SRC "" CACHE STRING INTERNAL)
set (VSP4_PROTO_CXX_INC "" CACHE STRING INTERNAL)
set (VSP4_HEADER_INC "")
foreach (PROTO_FILE ${VSP4_PROTO_SRC})
	get_filename_component (PROTO_FILE_NAME "${PROTO_FILE}" NAME_WE)

	set (PROTO_CXX_SRC_FILE "${VSP4_PROTO_CXX_DIR}/${PROTO_FILE_NAME}.pb.cc")
	set (PROTO_CXX_INC_FILE "${VSP4_PROTO_CXX_DIR}/${PROTO_FILE_NAME}.pb.h")
	list (APPEND VSP4_PROTO_CXX_SRC "${PROTO_CXX_SRC_FILE}")
	list (APPEND VSP4_PROTO_CXX_INC "${PROTO_CXX_INC_FILE}")
	set (VSP4_HEADER_INC "${VSP4_HEADER_INC}#include \"${PROTO_FILE_NAME}.pb.h\"\n")

	add_custom_command (
		OUTPUT "${PROTO_CXX_SRC_FILE}" "${PROTO_CXX_INC_FILE}"
		COMMAND $<TARGET_FILE:protobuf::protoc>
		ARGS --cpp_out=dllexport_decl=VSP4_API:${VSP4_PROTO_CXX_DIR}
			--proto_path=${VSP4_PROTO_SRC_DIR}
			${PROTO_FILE}
		DEPENDS ${PROTO_FILE}
		COMMENT "Running C++ protocol buffer compiler on ${PROTO_FILE}"
		VERBATIM)
endforeach (PROTO_FILE ${VSP4_PROTO_SRC})

# Proto Header Commands
configure_file (
	"${VSP4_PROTO_CXX_DIR}/VSP4.h.in"
	"${VSP4_PROTO_CXX_DIR}/VSP4.h"
	@ONLY)

# Set File Properties
set_source_files_properties (
	${VSP4_PROTO_CXX_SRC} ${VSP4_PROTO_CXX_INC} "${VSP4_PROTO_CXX_DIR}/VSP4.h"
	PROPERTIES GENERATED TRUE)

# Add Target
add_custom_target (vsp4_proto_generate
	DEPENDS ${VSP4_PROTO_CXX_SRC} ${VSP4_PROTO_CXX_INC} "${VSP4_PROTO_CXX_DIR}/VSP4.h"
	COMMENT "Generate messages target"
	VERBATIM)
add_dependencies (
	vsp4_proto_generate
	protobuf::protoc)

# Add Dll Target
add_library (libvsp4 SHARED ${VSP4_PROTO_CXX_INC} ${VSP4_PROTO_CXX_SRC})
add_library (libvsp4-static STATIC ${VSP4_PROTO_CXX_INC} ${VSP4_PROTO_CXX_SRC})
target_link_libraries (libvsp4 PUBLIC protobuf::libprotobuf)
target_link_libraries (libvsp4-static PUBLIC protobuf::libprotobuf)
add_dependencies (libvsp4 vsp4_proto_generate)
add_dependencies (libvsp4-static vsp4_proto_generate)

# Dll Target Definitions
file (RELATIVE_PATH CXX_RELATIVE_DIR ${CMAKE_CURRENT_SOURCE_DIR} ${VSP4_PROTO_CXX_DIR})
target_include_directories (libvsp4 PUBLIC
	"$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${CXX_RELATIVE_DIR}>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${VSP4_INSTALL_NAME}/${CXX_RELATIVE_DIR}>"
)
target_include_directories (libvsp4-static PUBLIC
	"$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${CXX_RELATIVE_DIR}>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${VSP4_INSTALL_NAME}/${CXX_RELATIVE_DIR}>"
)

target_compile_definitions (libvsp4 PRIVATE "VSP4_DLL_BUILD=1")
target_compile_definitions (libvsp4 PUBLIC "VSP4_DLL=1")

if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
target_compile_definitions (libvsp4 PRIVATE "VSP4_API=__declspec(dllexport)")
else ()
target_compile_definitions (libvsp4 PRIVATE "VSP4_API=__attribute__((visibility(\"default\")))")
endif ()
target_compile_definitions (libvsp4-static PRIVATE "VSP4_API= ")

# Install
set (VSP4_CMAKE_CONFIG_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/${VSP4_INSTALL_NAME})

install (TARGETS libvsp4
	EXPORT ${VSP4_INSTALL_NAME}Targets
	RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" OPTIONAL
	LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" OPTIONAL
	ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" OPTIONAL
)
install (TARGETS libvsp4-static
	EXPORT ${VSP4_INSTALL_NAME}Targets
	RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" OPTIONAL
	LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" OPTIONAL
	ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" OPTIONAL
)

install (FILES $<TARGET_PDB_FILE:libvsp4> DESTINATION "${CMAKE_INSTALL_BINDIR}" OPTIONAL)

file (GLOB_RECURSE VSP4_HDR_FILES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "${VSP4_PROTO_CXX_DIR}/*.h")
foreach (HDR_FILE ${VSP4_PROTO_CXX_INC})
	file (RELATIVE_PATH HDR_RELATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR} "${HDR_FILE}")
	list (APPEND VSP4_HDR_FILES "${HDR_RELATIVE_PATH}")
endforeach (HDR_FILE ${VSP4_PROTO_CXX_INC})
foreach (HDR_FILE ${VSP4_HDR_FILES})
	get_filename_component (HDR_DIR ${HDR_FILE} DIRECTORY)
	install (FILES "${CMAKE_CURRENT_SOURCE_DIR}/${HDR_FILE}" DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/${VSP4_INSTALL_NAME}/${HDR_DIR}" OPTIONAL)
endforeach (HDR_FILE ${VSP4_HDR_FILES})

write_basic_package_version_file (
	"${CMAKE_CURRENT_BINARY_DIR}/${VSP4_INSTALL_NAME}-config-version.cmake"
	VERSION ${PROJECT_VERSION}
	COMPATIBILITY AnyNewerVersion
)

configure_package_config_file (
	"${CMAKE_CURRENT_LIST_DIR}/${VSP4_INSTALL_NAME}-config.cmake.in"
	"${CMAKE_CURRENT_BINARY_DIR}/${VSP4_INSTALL_NAME}-config.cmake"
	INSTALL_DESTINATION ${VSP4_CMAKE_CONFIG_INSTALL_DIR}
	NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

install (FILES
	"${CMAKE_CURRENT_BINARY_DIR}/${VSP4_INSTALL_NAME}-config.cmake"
	"${CMAKE_CURRENT_BINARY_DIR}/${VSP4_INSTALL_NAME}-config-version.cmake"
	DESTINATION ${VSP4_CMAKE_CONFIG_INSTALL_DIR}
)

install (EXPORT ${VSP4_INSTALL_NAME}Targets
	FILE "${VSP4_INSTALL_NAME}-targets.cmake"
	NAMESPACE ${VSP4_INSTALL_NAME}::
	DESTINATION ${VSP4_CMAKE_CONFIG_INSTALL_DIR}
)