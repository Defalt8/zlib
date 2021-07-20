cmake_minimum_required( VERSION 3.10 )

project( zlib VERSION 1.2.11 LANGUAGES C )

set( CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE )
set( DEBUG_SUFFIX "" )
set( TYPE_SUFFIX "" )
set( ZLIB_INSTALL_LIBS ON CACHE BOOL "" )
set( ZLIB_INSTALL_BINS ON CACHE BOOL "" )
set( ZLIB_LIB_DESTINATION "lib" CACHE PATH "" )
set( ZLIB_BIN_DESTINATION "bin" CACHE PATH "" )

option(ASM686 "Enable building i686 assembly implementation")
option(AMD64 "Enable building amd64 assembly implementation")

include(CheckTypeSize)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckCSourceCompiles)
enable_testing()

check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(stdint.h    HAVE_STDINT_H)
check_include_file(stddef.h    HAVE_STDDEF_H)

# Check to see if we have large file support
set(CMAKE_REQUIRED_DEFINITIONS -D_LARGEFILE64_SOURCE=1)
# We add these other definitions here because CheckTypeSize.cmake
# in CMake 2.4.x does not automatically do so and we want
# compatibility with CMake 2.4.x.
if(HAVE_SYS_TYPES_H)
	list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_SYS_TYPES_H)
endif()
if(HAVE_STDINT_H)
	list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDINT_H)
endif()
if(HAVE_STDDEF_H)
	list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDDEF_H)
endif()
check_type_size(off64_t OFF64_T)
if(HAVE_OFF64_T)
   add_definitions(-D_LARGEFILE64_SOURCE=1)
endif()
set(CMAKE_REQUIRED_DEFINITIONS) # clear variable

# Check for fseeko
check_function_exists(fseeko HAVE_FSEEKO)
if(NOT HAVE_FSEEKO)
	add_definitions(-DNO_FSEEKO)
endif()

# Check for unistd.h
check_include_file(unistd.h Z_HAVE_UNISTD_H)

if(MSVC)
	add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
	add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
	include_directories(${CMAKE_CURRENT_SOURCE_DIR})
endif()

if( "${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_SOURCE_DIR}" )
	set( ZLIB_STANDALONE TRUE )
else()
	set( ZLIB_STANDALONE FALSE )
endif()

set( CMAKE_BUILD_TYPE Debug CACHE STRING "Build configuration type: ${CMAKE_CONFIGURATION_TYPES}." )
set( ZLIB_TYPE STATIC CACHE STRING "Library type." )
set( ZLIB_RUNTIME_DIR "${PROJECT_BINARY_DIR}/bin" CACHE PATH "Binary directory path." )
set( ZLIB_ARCHIVE_DIR "${PROJECT_BINARY_DIR}/lib" CACHE PATH "archive library directory path." )
set( ZLIB_INSTALL_HEADERS ${ZLIB_STANDALONE} CACHE BOOL "" )
set( ZLIB_BUILD_EXAMPLES ${ZLIB_STANDALONE} CACHE BOOL "" )
set( ZLIB_INCLUDE_DIR "${PROJECT_SOURCE_DIR}/include" )

set( ZLIB_PC "${CMAKE_CURRENT_BINARY_DIR}/zlib.pc" )
configure_file( "${CMAKE_CURRENT_SOURCE_DIR}/zlib.pc.cmakein" "${ZLIB_PC}" @ONLY )
configure_file(	"${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.cmakein" "${ZLIB_INCLUDE_DIR}/zlib/zconf.h" @ONLY )

set( ZLIB_PUBLIC_HDRS
	${ZLIB_INCLUDE_DIR}/zlib/zconf.h
	${ZLIB_INCLUDE_DIR}/zlib/zlib.h
)
set( ZLIB_PRIVATE_HDRS
	crc32.h
	deflate.h
	gzguts.h
	inffast.h
	inffixed.h
	inflate.h
	inftrees.h
	trees.h
	zutil.h
)
set( ZLIB_SRCS
	adler32.c
	compress.c
	crc32.c
	deflate.c
	gzclose.c
	gzlib.c
	gzread.c
	gzwrite.c
	inflate.c
	infback.c
	inftrees.c
	inffast.c
	trees.c
	uncompr.c
	zutil.c
)

if(NOT MINGW)
	set(ZLIB_DLL_SRCS
		win32/zlib1.rc # If present will override custom build rule below.
	)
endif()

if(CMAKE_COMPILER_IS_GNUCC)
	if(ASM686)
		set(ZLIB_ASMS contrib/asm686/match.S)
	elseif (AMD64)
		set(ZLIB_ASMS contrib/amd64/amd64-match.S)
	endif ()

	if(ZLIB_ASMS)
		add_definitions(-DASMV)
		set_source_files_properties(${ZLIB_ASMS} PROPERTIES LANGUAGE C COMPILE_FLAGS -DNO_UNDERLINE)
	endif()
endif()

if(MSVC)
	if(ASM686)
		ENABLE_LANGUAGE(ASM_MASM)
		set(ZLIB_ASMS
			contrib/masmx86/inffas32.asm
			contrib/masmx86/match686.asm
		)
	elseif (AMD64)
		ENABLE_LANGUAGE(ASM_MASM)
		set(ZLIB_ASMS
			contrib/masmx64/gvmat64.asm
			contrib/masmx64/inffasx64.asm
		)
	endif()

	if(ZLIB_ASMS)
		add_definitions(-DASMV -DASMINF)
	endif()
endif()

# parse the full version number from zlib.h and include in ZLIB_FULL_VERSION
file( READ "${ZLIB_INCLUDE_DIR}/zlib/zlib.h" _zlib_h_contents )
string( REGEX REPLACE ".*#define[ \t]+ZLIB_VERSION[ \t]+\"([-0-9A-Za-z.]+)\".*" "\\1" ZLIB_FULL_VERSION ${_zlib_h_contents} )

if(MINGW)
	# This gets us DLL resource information when compiling on MinGW.
	if(NOT CMAKE_RC_COMPILER)
		set(CMAKE_RC_COMPILER windres.exe)
	endif()

	add_custom_command( OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
		COMMAND "'${CMAKE_RC_COMPILER}'"
			-D GCC_WINDRES
			-I ${CMAKE_CURRENT_SOURCE_DIR}
			-I ${CMAKE_CURRENT_BINARY_DIR}
			-o ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
			-i ${CMAKE_CURRENT_SOURCE_DIR}/win32/zlib1.rc
		)
	set( ZLIB_DLL_SRCS "${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj" )
endif(MINGW)

if( "${ZLIB_TYPE}" MATCHES "^[Ss][Hh][Aa][Rr][Ee][Dd]$" OR BUILD_SHARED_LIBS )
	set( ZLIB_TYPE SHARED )
else()
	set( ZLIB_TYPE STATIC )
	set( TYPE_SUFFIX s )
endif()

if( "${CMAKE_BUILD_TYPE}" MATCHES "[Dd][Ee][Bb][Uu][Gg]" )
	set( DEBUG_SUFFIX d )
endif()

if( "${ZLIB_TYPE}" STREQUAL "SHARED" )
	add_library( zlib ${ZLIB_TYPE} ${ZLIB_SRCS} ${ZLIB_ASMS} ${ZLIB_DLL_SRCS} ${ZLIB_PUBLIC_HDRS} ${ZLIB_PRIVATE_HDRS} )
else()
	add_library( zlib ${ZLIB_TYPE} ${ZLIB_SRCS} ${ZLIB_ASMS} ${ZLIB_PUBLIC_HDRS} ${ZLIB_PRIVATE_HDRS} )
endif()

target_include_directories( zlib
	PUBLIC "${ZLIB_INCLUDE_DIR}"
	PRIVATE "${PROJET_SROUCE_DIR}"
)

set_target_properties( zlib PROPERTIES
	ARCHIVE_OUTPUT_NAME "zlib${TYPE_SUFFIX}${DEBUG_SUFFIX}"
	RUNTIME_OUTPUT_NAME "zlib${TYPE_SUFFIX}${DEBUG_SUFFIX}"
	ARCHIVE_OUTPUT_DIRECTORY "${ZLIB_ARCHIVE_DIR}"
	RUNTIME_OUTPUT_DIRECTORY "${ZLIB_RUNTIME_DIR}"
	PROPERTIES DEFINE_SYMBOL ZLIB_DLL
	PROPERTIES SOVERSION 1
)

if(NOT CYGWIN AND "${ZLIB_TYPE}" STREQUAL "SHARED")
	set_target_properties(zlib PROPERTIES VERSION ${ZLIB_FULL_VERSION})
endif()

if(UNIX)
	if(NOT APPLE AND "${ZLIB_TYPE}" STREQUAL "SHARED" )
		set_target_properties(zlib PROPERTIES LINK_FLAGS "-Wl,--version-script,\"${CMAKE_CURRENT_SOURCE_DIR}/zlib.map\"")
	endif()
endif()

if( ZLIB_INSTALL_LIBS )
	install( TARGETS zlib
		ARCHIVE DESTINATION "${ZLIB_LIB_DESTINATION}"
	)
endif()

if( ZLIB_INSTALL_BINS )
	install( TARGETS zlib
		RUNTIME DESTINATION "${ZLIB_BIN_DESTINATION}"
	)
endif()

if( ZLIB_INSTALL_HEADERS )
	install( FILES 
		${ZLIB_PUBLIC_HDRS}
		DESTINATION "include/zlib"
	)
endif()

#============================================================================
# Example binaries
#============================================================================

if( ZLIB_BUILD_EXAMPLES )
	add_executable(example test/example.c)
	target_link_libraries(example zlib)
	add_test(example example)

	add_executable(minigzip test/minigzip.c)
	target_link_libraries(minigzip zlib)

	if(HAVE_OFF64_T)
		add_executable(example64 test/example.c)
		target_link_libraries(example64 zlib)
		set_target_properties(example64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
		add_test(example64 example64)

		add_executable(minigzip64 test/minigzip.c)
		target_link_libraries(minigzip64 zlib)
		set_target_properties(minigzip64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
endif()
endif()
