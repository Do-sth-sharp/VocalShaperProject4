cmake_minimum_required (VERSION 3.15)
project (VocalShaperProject4 VERSION 1.0.0 LANGUAGES CXX)

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

# Directories
set (VSP4_PROTO_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/proto" CACHE STRING INTERNAL)
set (VSP4_PROTO_CXX_DIR "${CMAKE_CURRENT_SOURCE_DIR}/cxx" CACHE STRING INTERNAL)
set (PROTOBUF_DIR "${CMAKE_CURRENT_SOURCE_DIR}/protobuf" CACHE STRING INTERNAL)

# Get Protobuf
set (protobuf_BUILD_SHARED_LIBS ON)
set (protobuf_BUILD_TESTS OFF)
set (protobuf_WITH_ZLIB OFF)
#set (CMAKE_POLICY_DEFAULT_CMP0077 NEW)
set (ABSL_PROPAGATE_CXX_STD ON)
add_subdirectory (${PROTOBUF_DIR})

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
add_custom_target (vsp4_proto_generate ALL
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

# Dll Target Definitions
target_include_directories (libvsp4 PUBLIC "${PROTOBUF_DIR}/src")
target_include_directories (libvsp4-static PUBLIC "${PROTOBUF_DIR}/src")

if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
target_compile_definitions (libvsp4 PRIVATE "VSP4_API=__declspec(dllexport)")
else ()
target_compile_definitions (libvsp4 PRIVATE "VSP4_API=__attribute__((visibility(\"default\")))")
endif ()
target_compile_definitions (libvsp4-static PRIVATE "VSP4_API= ")

# Output Path
if (DEFINED VSP4_OUTPUT)
	set_target_properties(libvsp4 libvsp4-static libprotobuf
		PROPERTIES
		ARCHIVE_OUTPUT_DIRECTORY "${VSP4_OUTPUT}"
		LIBRARY_OUTPUT_DIRECTORY "${VSP4_OUTPUT}"
		RUNTIME_OUTPUT_DIRECTORY "${VSP4_OUTPUT}"
	)
endif (DEFINED VSP4_OUTPUT)
