project(mig)

cmake_minimum_required(VERSION 3.13)

find_package(BISON)
find_package(FLEX)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99")
# These -D__int64_t=int64_t etc. force-rename Linux glibc's __int64_t macro
# to match BSD's int64_t typedef. On macOS host, __int64_t is already an
# internal Apple macro that expands to a real type — overriding it makes
# Apple's <sys/_types.h> recurse and fail to parse.
if (NOT DARLING_HOST_MACOS)
	add_definitions(-Du_int=unsigned -D__int64_t=int64_t -D__int32_t=int32_t -D__darwin_natural_t=unsigned -D__uint32_t=uint32_t -D__uint16_t=uint16_t -D__uint64_t=uint64_t -D_BSD_I386__TYPES_H_)
	if(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
		add_definitions(-D_BSD_ARM__TYPES_H_ -D__arm64__)
	endif()
endif()
add_definitions('-DMIG_VERSION="1.0"')
# darling/include/sys/_types.h and arm/_types.h shadow Apple's real headers.
# On macOS host let Apple's win — keep the mach/ subdirectory for the
# bundled mach headers (those don't conflict with anything system-side).
if (DARLING_HOST_MACOS)
	include_directories(${CMAKE_CURRENT_SOURCE_DIR}/darling/include/mach)
else()
	include_directories(${CMAKE_CURRENT_SOURCE_DIR}/darling/include)
	include_directories(${CMAKE_CURRENT_SOURCE_DIR}/darling/include/mach)
endif()
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/migcom.tproj)

BISON_TARGET(MyParser migcom.tproj/parser.y ${CMAKE_CURRENT_BINARY_DIR}/parser.c)
FLEX_TARGET(MyScanner migcom.tproj/lexxer.l ${CMAKE_CURRENT_BINARY_DIR}/lexer.c)
ADD_FLEX_BISON_DEPENDENCY(MyScanner MyParser)

# Hide some unimportant warnings
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-dangling-else -Wno-implicit-function-declaration")

set(mig_sources
	migcom.tproj/error.c
	migcom.tproj/global.c
	#migcom.tproj/handler.c
	migcom.tproj/header.c
	migcom.tproj/mig.c
	migcom.tproj/routine.c
	migcom.tproj/server.c
	migcom.tproj/statement.c
	migcom.tproj/string.c
	migcom.tproj/type.c
	migcom.tproj/user.c
	migcom.tproj/utils.c
	darling/src/xtracemig.c
	${BISON_MyParser_OUTPUTS}
    ${FLEX_MyScanner_OUTPUTS}
)

add_executable(migcom ${mig_sources})

add_custom_command(
	OUTPUT "${CMAKE_BINARY_DIR}/build-mig"
	DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/migcom.tproj/mig.sh"
	# COMMAND args don't pass through a shell, so '>' and '&&' were parsed
	# literally as awk arguments and chmod never ran. Use CMake -E redirect
	# (3.18+) and chmod via CMake's own -E chmod (3.19+).
	#
	# IMPORTANT: do not write the COMMAND token as a bare "chmod" — the
	# Darling tree builds a Darwin-targeted `chmod` Mach-O executable
	# (src/external/file_cmds/chmod). CMake resolves a bare command name
	# against existing targets, attaches an implicit STRONG dependency,
	# and that pulls every libsystem_* the Darwin chmod links against into
	# the migexe build graph — closing the cycle migexe → chmod → libc →
	# libsyscall → libsyscall_generate_mig_headers → migexe. Always use
	# CMake's host-side -E chmod, or `/bin/chmod` with absolute path, so
	# no implicit target lookup happens.
	COMMAND ${CMAKE_COMMAND} -E env
		awk
		-v "migcc=${CMAKE_C_COMPILER}"
		-v "migcom=${CMAKE_CURRENT_BINARY_DIR}/migcom"
		-f "${CMAKE_CURRENT_SOURCE_DIR}/darling/src/mig.awk"
		"${CMAKE_CURRENT_SOURCE_DIR}/migcom.tproj/mig.sh"
		> "${CMAKE_BINARY_DIR}/build-mig"
	# cmake -E chmod requires cmake ≥3.21 (container has 3.18); invoke
	# /bin/chmod via -E env to avoid CMake's bare-name resolution binding
	# to the Darling-built Mach-O `chmod` target (which would create a
	# circular libsyscall→migexe dependency).
	COMMAND ${CMAKE_COMMAND} -E env /bin/chmod 0755 "${CMAKE_BINARY_DIR}/build-mig"
	)
add_custom_target(migexe DEPENDS "${CMAKE_BINARY_DIR}/build-mig")

add_dependencies(migexe migcom)


