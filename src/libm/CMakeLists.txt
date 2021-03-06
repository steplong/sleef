file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/include/)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/include/)
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

set(CMAKE_C_FLAGS "${ORG_CMAKE_C_FLAGS} ${SLEEF_C_FLAGS}")

set(EXT_ENABLE_ALIAS AVX512F ADVSIMD NEON32 VSX)

# --------------------------------------------------------------------
# sleef.h
# --------------------------------------------------------------------
# File generated for the headers
set(SLEEF_ORG_HEADER ${CMAKE_CURRENT_SOURCE_DIR}/sleeflibm_header.h.org)
set(SLEEF_ORG_FOOTER ${CMAKE_CURRENT_SOURCE_DIR}/sleeflibm_footer.h.org)
set(SLEEF_INCLUDE_HEADER ${CMAKE_BINARY_DIR}/include/sleef.h)

configure_file(${SLEEF_ORG_HEADER} ${SLEEF_INCLUDE_HEADER})

set(SLEEF_HEADER_COMMANDS "")
foreach(SIMD ${SLEEF_HEADER_LIST})
  list(APPEND SLEEF_HEADER_COMMANDS COMMAND echo Generating sleef.h: ${TARGET_MKRENAME} ${HEADER_PARAMS_${SIMD}})
  list(APPEND SLEEF_HEADER_COMMANDS COMMAND $<TARGET_FILE:${TARGET_MKRENAME}> ${HEADER_PARAMS_${SIMD}} >> ${SLEEF_INCLUDE_HEADER})
endforeach()

if((MSVC OR MINGW AND WIN32) OR SLEEF_CLANG_ON_WINDOWS)
  string(REPLACE "/" "\\" sleef_footer_input_file "${SLEEF_ORG_FOOTER}")
  list(APPEND SLEEF_HEADER_COMMANDS COMMAND type ${sleef_footer_input_file} >> ${SLEEF_INCLUDE_HEADER})
else()
  list(APPEND SLEEF_HEADER_COMMANDS COMMAND cat ${SLEEF_ORG_FOOTER} >> ${SLEEF_INCLUDE_HEADER})
endif()

add_custom_command(OUTPUT ${SLEEF_INCLUDE_HEADER}
  ${SLEEF_HEADER_COMMANDS}
  DEPENDS
    ${SLEEF_ORG_HEADER}
    ${SLEEF_ORG_FOOTER}
    ${TARGET_MKRENAME}
)

# --------------------------------------------------------------------
# TARGET_MKRENAME
# renameXXX.h for each SIMD
# --------------------------------------------------------------------
# Helper executable: generates parts of the sleef header file
add_host_executable(${TARGET_MKRENAME} mkrename.c)

# Enable Vector PCS for Advanced SIMD (if supported)
if(FORCE_AAVPCS)
  host_target_AAVPCS_definitions(${TARGET_MKRENAME})
endif()

set(HEADER_FILES_GENERATED "")
foreach(SIMD ${SLEEF_SUPPORTED_EXTENSIONS})
  if(COMPILER_SUPPORTS_${SIMD})
    # Need lowercase string for rename header
    string(TOLOWER ${SIMD} vecarch)
    set(OBJECT_${SIMD} "sleef${vecarch}")
    set(OBJECTDET_${SIMD} "sleefdet${vecarch}")
    set(HEADER_${SIMD} ${CMAKE_CURRENT_BINARY_DIR}/include/rename${vecarch}.h)
    list(APPEND HEADER_FILES_GENERATED ${HEADER_${SIMD}})

    # Generate mkrename commands
    add_custom_command(OUTPUT ${HEADER_${SIMD}}
      COMMAND echo Generating rename${vecarch}.h: ${TARGET_MKRENAME} ${RENAME_PARAMS_${SIMD}}
      COMMAND $<TARGET_FILE:${TARGET_MKRENAME}> ${RENAME_PARAMS_${SIMD}} > ${HEADER_${SIMD}}
      DEPENDS ${TARGET_MKRENAME}
    )
    add_custom_target(rename${SIMD}.h_generated DEPENDS ${HEADER_${SIMD}})
  endif()
endforeach()

# --------------------------------------------------------------------
# TARGET_MKRENAME_GNUABI
# renameXXX_gnuabi.h for each SIMD GNU Abi
# --------------------------------------------------------------------
# Helper executable: generates parts of the sleef header file gnu_abi
add_host_executable(${TARGET_MKRENAME_GNUABI} mkrename_gnuabi.c)

set(HEADER_GNUABI_FILES_GENERATED "")
if(ENABLE_GNUABI)
  foreach(SIMD ${SLEEF_SUPPORTED_GNUABI_EXTENSIONS})
    if(COMPILER_SUPPORTS_${SIMD})
      string(TOLOWER ${SIMD} vecarch)
      set(OBJECT_${SIMD}_dp_GNUABI "sleefgnuabi${vecarch}dp")
      set(OBJECT_${SIMD}_sp_GNUABI "sleefgnuabi${vecarch}sp")
      set(HEADER_${SIMD}_GNUABI ${CMAKE_CURRENT_BINARY_DIR}/include/rename${vecarch}_gnuabi.h)
      list(APPEND HEADER_GNUABI_FILES_GENERATED ${HEADER_${SIMD}_GNUABI})

      # Generate mkrename_gnuabi commands
      add_custom_command(OUTPUT ${HEADER_${SIMD}_GNUABI}
        COMMAND echo Generating rename${vecarch}_gnuabi.h: ${TARGET_MKRENAME_GNUABI} ${RENAME_PARAMS_GNUABI_${SIMD}}
        COMMAND $<TARGET_FILE:${TARGET_MKRENAME_GNUABI}> ${RENAME_PARAMS_GNUABI_${SIMD}} > ${HEADER_${SIMD}_GNUABI}
        DEPENDS ${TARGET_MKRENAME_GNUABI}
      )
      # set_source_files_properties(${HEADER_${SIMD}_GNUABI} PROPERTIES GENERATED TRUE)
    endif()
  endforeach()
endif()

# --------------------------------------------------------------------

# TARGET_MKMASKED_GNUABI
add_host_executable(${TARGET_MKMASKED_GNUABI} mkmasked_gnuabi.c)

# maskedXXX_YY_gnuabi.h
if(ENABLE_GNUABI)
  foreach(SIMD ${SLEEF_SUPPORTED_GNUABI_EXTENSIONS})
    if(COMPILER_SUPPORTS_${SIMD} AND MKMASKED_PARAMS_GNUABI_${SIMD}_sp)
      string(TOLOWER ${SIMD} vecarch)

      set(HEADER_GENERATED "")
      foreach(T dp sp)
	set(HEADER_MASKED_${SIMD}_${T}_GNUABI ${CMAKE_CURRENT_BINARY_DIR}/include/masked_${vecarch}_${T}_gnuabi.h)
	list(APPEND HEADER_GENERATED ${HEADER_MASKED_${SIMD}_${T}_GNUABI})

	add_custom_command(OUTPUT ${HEADER_MASKED_${SIMD}_${T}_GNUABI}
          COMMAND echo Generating ${HEADER_MASKED_${SIMD}_${T}_GNUABI}
          COMMAND $<TARGET_FILE:${TARGET_MKMASKED_GNUABI}> ${MKMASKED_PARAMS_GNUABI_${SIMD}_${T}} > ${HEADER_MASKED_${SIMD}_${T}_GNUABI}
          DEPENDS ${TARGET_MKMASKED_GNUABI}
	  )
      endforeach()
      add_custom_target(masked${SIMD}_generated DEPENDS ${HEADER_GENERATED})
    endif()
  endforeach()
endif()

# --------------------------------------------------------------------
# TARGET_HEADERS
# --------------------------------------------------------------------
add_custom_target(${TARGET_HEADERS} ALL
  DEPENDS
    ${SLEEF_INCLUDE_HEADER}            # Output only
    ${HEADER_FILES_GENERATED}          # Output only
    ${HEADER_GNUABI_FILES_GENERATED}   # Output only
)

# --------------------------------------------------------------------
# TARGET_MKALIAS
# --------------------------------------------------------------------
add_host_executable(${TARGET_MKALIAS} mkalias.c)
if(FORCE_AAVPCS)
  host_target_AAVPCS_definitions(${TARGET_MKALIAS})
endif()

# --------------------------------------------------------------------
# TARGET_MKDISP
# --------------------------------------------------------------------
# Helper executable: dispatcher for the vector extensions
add_host_executable(${TARGET_MKDISP} mkdisp.c)

# Set C standard requirement (-std=gnu99 for gcc)
set_target_properties(
  ${TARGET_MKRENAME} ${TARGET_MKRENAME_GNUABI} ${TARGET_MKDISP}
  ${TARGET_MKALIAS} ${TARGET_MKMASKED_GNUABI}
  PROPERTIES C_STANDARD 99
)

# --------------------------------------------------------------------
# TARGET_LIBSLEEF
# --------------------------------------------------------------------
# Build main library

set(COMMON_TARGET_PROPERTIES 
  C_STANDARD 99                  # -std=gnu99
  )

if (BUILD_SHARED_LIBS)
  list(APPEND COMMON_TARGET_PROPERTIES POSITION_INDEPENDENT_CODE ON)   # -fPIC
endif()

if (ENABLE_LTO)
  list(APPEND COMMON_TARGET_PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)  # -flto
endif()

# Original sleef sources
set(STANDARD_SOURCES sleefdp.c sleefsp.c rempitab.c)

# Check for different precision support and add sources accordingly
if(COMPILER_SUPPORTS_LONG_DOUBLE)
  list(APPEND STANDARD_SOURCES sleefld.c)
endif()

add_library(${TARGET_LIBSLEEF} ${STANDARD_SOURCES})
set_target_properties(${TARGET_LIBSLEEF} PROPERTIES
  VERSION ${SLEEF_VERSION}
  SOVERSION ${SLEEF_SOVERSION}
  ${COMMON_TARGET_PROPERTIES}
)

target_compile_definitions(${TARGET_LIBSLEEF}
  PRIVATE DORENAME=1 ${COMMON_TARGET_DEFINITIONS}
)

if(COMPILER_SUPPORTS_FLOAT128)
  # TODO: Not supported for LLVM bitcode gen as it has a specific compilation flags
  target_sources(${TARGET_LIBSLEEF} PRIVATE sleefqp.c)
  target_compile_definitions(${TARGET_LIBSLEEF}
    PRIVATE ENABLEFLOAT128=1 ${COMMON_TARGET_DEFINITIONS})
endif()

if(COMPILER_SUPPORTS_BUILTIN_MATH)
  target_compile_definitions(${TARGET_LIBSLEEF} PRIVATE ENABLE_BUILTIN_MATH=1)
endif()

# Compile SIMD versions
# Single precision and double precision

# Include symbols for each SIMD architecture (if supported by the platform)
# Note: adds object file as sources via cmake conditional generator expression
foreach(SIMD ${SLEEF_SUPPORTED_EXTENSIONS})
  if(COMPILER_SUPPORTS_${SIMD})
    list(FIND EXT_ENABLE_ALIAS ${SIMD} INDEX_ALIAS)
    string(TOLOWER ${SIMD} SIMDLC)

    if (${SIMD} STREQUAL "NEON32" OR ${SIMD} STREQUAL "NEON32VFPV4")
      set(SIMD_SOURCES sleefsimdsp.c)
    else()
      set(SIMD_SOURCES sleefsimdsp.c sleefsimddp.c)
    endif()

    # Create a library
    add_library(${OBJECT_${SIMD}} OBJECT ${SIMD_SOURCES} ${HEADER_${SIMD}})    
    add_library(${OBJECTDET_${SIMD}} OBJECT ${SIMD_SOURCES} ${HEADER_${SIMD}})    
    target_compile_definitions(${OBJECTDET_${SIMD}} PRIVATE DETERMINISTIC=1)

    if(COMPILER_SUPPORTS_BUILTIN_MATH)
      target_compile_definitions(${OBJECT_${SIMD}} PRIVATE ENABLE_BUILTIN_MATH=1)
      target_compile_definitions(${OBJECTDET_${SIMD}} PRIVATE ENABLE_BUILTIN_MATH=1)
    endif()

    if(ENABLE_ALTDIV)
      target_compile_definitions(${OBJECT_${SIMD}} PRIVATE ENABLE_ALTDIV=1)
      target_compile_definitions(${OBJECTDET_${SIMD}} PRIVATE ENABLE_ALTDIV=1)
    endif()

    if(ENABLE_ALTSQRT)
      target_compile_definitions(${OBJECT_${SIMD}} PRIVATE ENABLE_ALTSQRT=1)
      target_compile_definitions(${OBJECTDET_${SIMD}} PRIVATE ENABLE_ALTSQRT=1)
    endif()

    if (INDEX_ALIAS EQUAL -1)
      target_compile_definitions(${OBJECT_${SIMD}} PRIVATE
	ENABLE_${SIMD}=1
	DORENAME=1
	${COMMON_TARGET_DEFINITIONS}
	)
      target_compile_definitions(${OBJECTDET_${SIMD}} PRIVATE
	ENABLE_${SIMD}=1
	DORENAME=1
	${COMMON_TARGET_DEFINITIONS}
	)
    else()
      add_custom_command(
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/include/alias_${SIMDLC}.h
       	COMMENT "Generating alias_${SIMDLC}.h"
       	COMMAND $<TARGET_FILE:${TARGET_MKALIAS}> ${ALIAS_PARAMS_${SIMD}_SP} >  ${CMAKE_CURRENT_BINARY_DIR}/include/alias_${SIMDLC}.h
       	COMMAND $<TARGET_FILE:${TARGET_MKALIAS}> ${ALIAS_PARAMS_${SIMD}_DP} >> ${CMAKE_CURRENT_BINARY_DIR}/include/alias_${SIMDLC}.h
       	DEPENDS ${TARGET_MKALIAS}
       	)
      add_custom_target(alias_${SIMDLC}.h_generated SOURCES ${CMAKE_CURRENT_BINARY_DIR}/include/alias_${SIMDLC}.h)
      add_dependencies(${OBJECT_${SIMD}} alias_${SIMDLC}.h_generated)
      add_dependencies(${OBJECTDET_${SIMD}} alias_${SIMDLC}.h_generated)
      target_compile_definitions(${OBJECT_${SIMD}} PRIVATE
	ENABLE_${SIMD}=1
	DORENAME=1
	${COMMON_TARGET_DEFINITIONS}
	ALIAS_NO_EXT_SUFFIX=\"alias_${SIMDLC}.h\"
	)
      target_compile_definitions(${OBJECTDET_${SIMD}} PRIVATE
	ENABLE_${SIMD}=1
	DORENAME=1
	${COMMON_TARGET_DEFINITIONS}
	ALIAS_NO_EXT_SUFFIX=\"alias_${SIMDLC}.h\"
	)
    endif()

    # Enable Vector PCS for Advanced SIMD (if supported)
    if(FORCE_AAVPCS AND ${SIMD} STREQUAL "ADVSIMD")
      target_compile_definitions(${OBJECT_${SIMD}} PRIVATE
        ENABLE_AAVPCS=1
      )
    endif()

    add_dependencies(${OBJECT_${SIMD}} rename${SIMD}.h_generated)
    add_dependencies(${OBJECTDET_${SIMD}} rename${SIMD}.h_generated)

    set_target_properties(${OBJECT_${SIMD}} PROPERTIES
      ${COMMON_TARGET_PROPERTIES}
      )

    set_target_properties(${OBJECTDET_${SIMD}} PROPERTIES
      ${COMMON_TARGET_PROPERTIES}
      )

    target_compile_options(${OBJECT_${SIMD}} PRIVATE
      ${FLAGS_ENABLE_${SIMD}})

    target_compile_options(${OBJECTDET_${SIMD}} PRIVATE
      ${FLAGS_ENABLE_${SIMD}})

    target_sources(${TARGET_LIBSLEEF} PRIVATE $<TARGET_OBJECTS:${OBJECT_${SIMD}}> $<TARGET_OBJECTS:${OBJECTDET_${SIMD}}>)
  endif(COMPILER_SUPPORTS_${SIMD})
endforeach()

#

if(BUILD_INLINE_HEADERS)
  file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/include/)
  set(INLINE_HEADER_FILES_GENERATED "")

  if (SED_COMMAND)
    foreach(SIMD ${SLEEF_SUPPORTED_EXTENSIONS})
      if(COMPILER_SUPPORTS_${SIMD})
	string(TOLOWER ${SIMD} SIMDLC)

	set(INLINE_HEADER_FILE ${PROJECT_BINARY_DIR}/include/sleefinline_${SIMDLC}.h)
	add_custom_command(
	  OUTPUT ${INLINE_HEADER_FILE} 

	  COMMAND echo Generating sleefinline_${SIMDLC}.h
  
	  # Preprocess sleefsimddp.c with SLEEF_GENHEADER defined, comments are preserved
	  COMMAND "${CMAKE_C_COMPILER}"  ${FLAG_PREPROCESS} ${FLAG_PRESERVE_COMMENTS}                      # gcc -E -C 
	    ${FLAG_INCLUDE}${PROJECT_SOURCE_DIR}/src/common ${FLAG_INCLUDE}${PROJECT_SOURCE_DIR}/src/arch  # -I/sleef/src/common -I/sleef/src/arch
	    ${FLAG_INCLUDE}${CMAKE_CURRENT_BINARY_DIR}/include/                                            # -I/build/src/libm/include
	    ${FLAG_DEFINE}SLEEF_GENHEADER ${FLAG_DEFINE}ENABLE_${SIMD} ${FLAG_DEFINE}DORENAME              # -DSLEEF_GENHEADER -DENABLE_SSE2 -DDORENAME
	    ${CMAKE_CURRENT_SOURCE_DIR}/sleefsimddp.c > ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.tmp1    # /sleef/libm/sleefsimddp.c > /build/libm/sleefSSE2.h.tmp1

	  # Remove all lines except those begin with "//@"
	  COMMAND ${SED_COMMAND} -n -e "/^\\/\\/@#.*$/p" ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.tmp1   # sed -n -e "/^\\/\\/@#.*$/p" /build/src/libm/sleefSSE2.h.tmp1
	    > ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.tmp2                                              # > /build/src/libm/sleefSSE2.h.tmp2

	  # Remove "//@"
	  COMMAND ${SED_COMMAND} -e "s/^\\/\\/@#/#/g" ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.tmp2      # sed -e "s/^\\/\\/@#/#/g" /build/src/libm/sleefSSE2.h.tmp2
	    > ${CMAKE_CURRENT_BINARY_DIR}/include/macroonly${SIMD}.h                                       # > /build/src/libm/include/macroonlySSE2.h

	  # Preprocess sleefsimdsp.c with SLEEF_GENHEADER defined. Include macroonly*.h instead of helper*.h.
	  COMMAND "${CMAKE_C_COMPILER}" ${FLAG_PREPROCESS} ${FLAG_PRESERVE_COMMENTS}                       # gcc -E -C 
	    ${FLAG_INCLUDE}${PROJECT_SOURCE_DIR}/src/common ${FLAG_INCLUDE}${PROJECT_SOURCE_DIR}/src/arch  # -I/sleef/src/common -I/sleef/src/arch
	    ${FLAG_INCLUDE}${CMAKE_CURRENT_BINARY_DIR}/include/                                            # -I/build/src/libm/include
	    ${FLAG_DEFINE}SLEEF_GENHEADER ${FLAG_DEFINE}ENABLE_${SIMD} ${FLAG_DEFINE}DORENAME              # -DSLEEF_GENHEADER -DENABLE_SSE2 -DDORENAME
	    ${CMAKE_CURRENT_SOURCE_DIR}/sleefsimdsp.c >> ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.tmp1   # /sleef/libm/sleefsimdsp.c >> /build/libm/sleefSSE2.h.tmp1

	  # Remove lines beginning with "#" so that the resulting file can be preprocessed again.
	  COMMAND ${SED_COMMAND} -e "s/^#.*//g" ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.tmp1            # sed -e "s/^#.*//g" /build/src/libm/sleefSSE2.h.tmp1
	    > ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.c                                                 # > /build/src/libm/sleefSSE2.h.c

	  # Preprocess the intemediate file again to remove comments
	  COMMAND "${CMAKE_C_COMPILER}" ${FLAG_PREPROCESS}                                                 # gcc -E
	    ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.c                                                   # /build/src/libm/sleefSSE2.h.c
	    > ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.tmp3                                              # > /build/src/libm/sleefSSE2.h.tmp3

	  # Embed version number into the header
	  COMMAND ${SED_COMMAND} -e                                                                        # sed -e
	    "s/%VERSION%/${SLEEF_VERSION_MAJOR}.${SLEEF_VERSION_MINOR}.${SLEEF_VERSION_PATCHLEVEL}/g"      # "s/%VERSION%/3.5.0/g"
	    ${CMAKE_CURRENT_SOURCE_DIR}/sleefinline_header.h.org                                           # /sleef/libm/sleefinline_header.h.org
	    > ${PROJECT_BINARY_DIR}/include/sleefinline_${SIMDLC}.h                                        # > /build/include/sleefinline_sse2.h

	  # Remove lines beginning with "#"
	  COMMAND ${SED_COMMAND} -e "s/^#.*//g" ${CMAKE_CURRENT_BINARY_DIR}/sleef${SIMD}.h.tmp3            # sed -e "s/^#.*//g" /build/src/libm/sleefSSE2.h.tmp3
	    >> ${PROJECT_BINARY_DIR}/include/sleefinline_${SIMDLC}.h                                       # >> /build/include/sleefinline_sse2.h

	  MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/sleefsimddp.c ${CMAKE_CURRENT_SOURCE_DIR}/sleefsimdsp.c ${HEADER_${SIMD}}
	  DEPENDS ${HEADER_${SIMD}}
	  VERBATIM
	  )

	list(APPEND INLINE_HEADER_FILES_GENERATED ${INLINE_HEADER_FILE})
      endif()
    endforeach()

    add_custom_target(${TARGET_INLINE_HEADERS} ALL
      DEPENDS
      ${INLINE_HEADER_FILES_GENERATED}
      )
    install(FILES ${INLINE_HEADER_FILES_GENERATED} DESTINATION include)
  endif(SED_COMMAND)

  add_library(${TARGET_LIBINLINE} STATIC rempitab.c)
  install(TARGETS ${TARGET_LIBINLINE} DESTINATION lib)
endif(BUILD_INLINE_HEADERS)

# On some systems we need to explicitly link libsleef against libm to
# use some of the math functions used in the scalar code (for example
# sqrt).
if(LIBM AND NOT COMPILER_SUPPORTS_BUILTIN_MATH)
  target_link_libraries(${TARGET_LIBSLEEF} ${LIBM})
endif()

target_sources(${TARGET_LIBSLEEF} PRIVATE $<TARGET_OBJECTS:${TARGET_LIBCOMMON_OBJ}>)

# --------------------------------------------------------------------

if (SLEEF_ARCH_X86)
  # Target dispsse.c

  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/dispsse.c
    COMMENT "Generating dispsse.c"
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/dispsse.c.org ${CMAKE_CURRENT_BINARY_DIR}/dispsse.c
    COMMAND $<TARGET_FILE:${TARGET_MKDISP}> 2 4 __m128d __m128 __m128i sse2 sse4 avx2128 >> ${CMAKE_CURRENT_BINARY_DIR}/dispsse.c
    DEPENDS ${TARGET_MKDISP}
    )
  add_custom_target(dispsse.c_generated SOURCES ${CMAKE_CURRENT_BINARY_DIR}/dispsse.c)

  # Target renamedsp128.h

  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/include/renamedsp128.h
    COMMENT "Generating renamedsp128.h"
    COMMAND $<TARGET_FILE:${TARGET_MKRENAME}> - 2 4 > ${CMAKE_CURRENT_BINARY_DIR}/include/renamedsp128.h
    DEPENDS ${TARGET_MKRENAME}
    )
  add_custom_target(renamedsp128.h_generated SOURCES ${CMAKE_CURRENT_BINARY_DIR}/include/renamedsp128.h)

  # Target dispavx.c

  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/dispavx.c
    COMMENT "Generating dispavx.c"
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/dispavx.c.org ${CMAKE_CURRENT_BINARY_DIR}/dispavx.c
    COMMAND $<TARGET_FILE:${TARGET_MKDISP}> 4 8 __m256d __m256 __m128i avx fma4 avx2 >> ${CMAKE_CURRENT_BINARY_DIR}/dispavx.c
    DEPENDS ${TARGET_MKDISP}
    )
  add_custom_target(dispavx.c_generated SOURCES ${CMAKE_CURRENT_BINARY_DIR}/dispavx.c)

  # Target renamedsp256.h

  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/include/renamedsp256.h
    COMMENT "Generating renamedsp256.h"
    COMMAND $<TARGET_FILE:${TARGET_MKRENAME}> - 4 8 > ${CMAKE_CURRENT_BINARY_DIR}/include/renamedsp256.h
    DEPENDS ${TARGET_MKRENAME}
    )
  add_custom_target(renamedsp256.h_generated SOURCES ${CMAKE_CURRENT_BINARY_DIR}/include/renamedsp256.h)

  # Target dispsse_obj

  if (COMPILER_SUPPORTS_FMA4)
    set(DISPATCHER_DEFINITIONS ${DISPATCHER_DEFINITIONS} ENABLE_FMA4=1)
  endif()

  if (COMPILER_SUPPORTS_AVX2)
    set(DISPATCHER_DEFINITIONS ${DISPATCHER_DEFINITIONS} ENABLE_AVX2=1)
  endif()

  if(COMPILER_SUPPORTS_SSE2)
    add_library(dispsse_obj OBJECT dispsse.c)
    target_compile_options(dispsse_obj PRIVATE ${FLAGS_ENABLE_SSE2})
    set_target_properties(dispsse_obj PROPERTIES ${COMMON_TARGET_PROPERTIES})
    target_compile_definitions(dispsse_obj PRIVATE ${COMMON_TARGET_DEFINITIONS} ${DISPATCHER_DEFINITIONS})
    target_include_directories(dispsse_obj PRIVATE ${CMAKE_BINARY_DIR}/include)
    add_dependencies(dispsse_obj dispsse.c_generated renamedsp128.h_generated ${TARGET_HEADERS})
    target_sources(${TARGET_LIBSLEEF} PRIVATE $<TARGET_OBJECTS:dispsse_obj>)
  endif()

  # Target dispavx_obj

  if(COMPILER_SUPPORTS_AVX)
    add_library(dispavx_obj OBJECT dispavx.c)
    target_compile_options(dispavx_obj PRIVATE ${FLAGS_ENABLE_AVX})
    set_target_properties(dispavx_obj PROPERTIES ${COMMON_TARGET_PROPERTIES})
    target_compile_definitions(dispavx_obj PRIVATE ${COMMON_TARGET_DEFINITIONS} ${DISPATCHER_DEFINITIONS})
    target_include_directories(dispavx_obj PRIVATE ${CMAKE_BINARY_DIR}/include)
    add_dependencies(dispavx_obj dispavx.c_generated renamedsp256.h_generated ${TARGET_HEADERS})
    target_sources(${TARGET_LIBSLEEF} PRIVATE $<TARGET_OBJECTS:dispavx_obj>)
  endif()
endif(SLEEF_ARCH_X86)

# --------------------------------------------------------------------
# TARGET_LIBSLEEFGNUABI
# Compile SIMD versions for GNU Abi
# --------------------------------------------------------------------
# Build gnuabi version from just simd object files
if(ENABLE_GNUABI)
  set(TARGET_LIBSLEEFGNUABI_OBJECTS "")
  foreach(SIMD ${SLEEF_SUPPORTED_GNUABI_EXTENSIONS})
    if(COMPILER_SUPPORTS_${SIMD})
      # Need lowercase string for rename header
      string(TOLOWER ${SIMD} vecarch)

      foreach(T dp sp)
	add_library(${OBJECT_${SIMD}_${T}_GNUABI} OBJECT sleefsimd${T}.c ${HEADER_${SIMD}_GNUABI})
	target_compile_definitions(${OBJECT_${SIMD}_${T}_GNUABI} PRIVATE
          ENABLE_${SIMD}=1
          DORENAME=1
          ENABLE_GNUABI=1
	  )

        if(FORCE_AAVPCS AND ${SIMD} STREQUAL "ADVSIMD")
          target_compile_definitions(${OBJECT_${SIMD}_${T}_GNUABI} PRIVATE
            ENABLE_AAVPCS=1
          )
        endif()

	if(ENABLE_ALTDIV)
	  target_compile_definitions(${OBJECT_${SIMD}_${T}_GNUABI} PRIVATE ENABLE_ALTDIV=1)
	endif()

	if(ENABLE_ALTSQRT)
	  target_compile_definitions(${OBJECT_${SIMD}_${T}_GNUABI} PRIVATE ENABLE_ALTSQRT=1)
	endif()

	set_target_properties(${OBJECT_${SIMD}_${T}_GNUABI} PROPERTIES
          ${COMMON_TARGET_PROPERTIES}
	  )

	target_compile_options(${OBJECT_${SIMD}_${T}_GNUABI} PRIVATE ${FLAGS_ENABLE_${SIMD}})
	if (COMPILER_SUPPORTS_WEAK_ALIASES)
          target_compile_options(${OBJECT_${SIMD}_${T}_GNUABI} PRIVATE -DENABLE_GNUABI=1)
	endif(COMPILER_SUPPORTS_WEAK_ALIASES)
	list(APPEND TARGET_LIBSLEEFGNUABI_OBJECTS $<TARGET_OBJECTS:${OBJECT_${SIMD}_${T}_GNUABI}>)

	if(MKMASKED_PARAMS_GNUABI_${SIMD}_${T})
	  target_compile_definitions(${OBJECT_${SIMD}_${T}_GNUABI} PRIVATE
	    HEADER_MASKED=\"masked_${vecarch}_${T}_gnuabi.h\")
	  add_dependencies(${OBJECT_${SIMD}_${T}_GNUABI} masked${SIMD}_generated)
	endif()
      endforeach()
    endif(COMPILER_SUPPORTS_${SIMD})
  endforeach()

  # Create library 
  add_library(${TARGET_LIBSLEEFGNUABI} ${TARGET_LIBSLEEFGNUABI_OBJECTS} rempitab.c)
  
  # Library properties
  set_target_properties(${TARGET_LIBSLEEFGNUABI} PROPERTIES
    VERSION ${SLEEF_VERSION_MAJOR}.${SLEEF_VERSION_MINOR}
    SOVERSION ${SLEEF_SOVERSION}
    POSITION_INDEPENDENT_CODE ON   # -fPIC
    C_STANDARD 99                  # -std=gnu99
    LINKER_LANGUAGE C
  )

# On some systems we need to explicitly link libsleefgnuabi against
# libm to use some of the math functions used in the scalar code (for
# example sqrt).
if(LIBM AND NOT COMPILER_SUPPORTS_BUILTIN_MATH)
  target_link_libraries(${TARGET_LIBSLEEFGNUABI} ${LIBM})
endif()
endif(ENABLE_GNUABI)

# --------------------------------------------------------------------
# TARGET_LLVM_BITCODE
# Generate LLVM bitcode
# --------------------------------------------------------------------
if(CLANG_EXE_PATH AND SLEEF_ENABLE_LLVM_BITCODE)
  set(SLEEP_LLVM_BITCODE_INCLUDES "")
  get_property(SLEEP_LLVM_BITCODE_INCLUDES_LIST DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY INCLUDE_DIRECTORIES)
  foreach(INCLUDE_DIRECTORY ${SLEEP_LLVM_BITCODE_INCLUDES_LIST})
    set(SLEEP_LLVM_BITCODE_INCLUDES "${SLEEP_LLVM_BITCODE_INCLUDES} -I ${INCLUDE_DIRECTORY}")
  endforeach()

  separate_arguments(SLEEP_LLVM_BITCODE_INCLUDES_CLANG WINDOWS_COMMAND "${SLEEP_LLVM_BITCODE_INCLUDES}")
  set(SLEEF_CLANG_LLVM_BITCODE_OPTIONS -O3 -S -emit-llvm -D NDEBUG -D DORENAME=1)
  set(LLVM_BITCODE_OUTPUTS "")

  # Generate LLVM bitcode for regular SLEEF
  foreach(STANDARD_SOURCE ${STANDARD_SOURCES})
    get_filename_component(SRC_WITHOUT_EXT ${STANDARD_SOURCE} NAME_WE)
    set(LLVM_BITCODE_INPUT ${CMAKE_CURRENT_SOURCE_DIR}/${SRC_WITHOUT_EXT}.c)
    set(LLVM_BITCODE_OUTPUT ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/${SRC_WITHOUT_EXT}.ll)
    add_custom_command(OUTPUT ${LLVM_BITCODE_OUTPUT}
      COMMAND ${CLANG_EXE_PATH} ${SLEEF_CLANG_LLVM_BITCODE_OPTIONS} -o ${LLVM_BITCODE_OUTPUT} ${LLVM_BITCODE_INPUT} ${SLEEP_LLVM_BITCODE_INCLUDES_CLANG}
      DEPENDS
        ${LLVM_BITCODE_INPUT}
    )
    list(APPEND LLVM_BITCODE_OUTPUTS ${LLVM_BITCODE_OUTPUT})
  endforeach()

  # Generate LLVM bitcode for SIMD SLEEF
  foreach(SIMD ${SLEEF_SUPPORTED_EXTENSIONS})
    if (${SIMD} STREQUAL "NEON32" OR ${SIMD} STREQUAL "NEON32VFPV4")
      set(SIMD_SOURCES sleefsimdsp.c)
    else()
      set(SIMD_SOURCES sleefsimdsp.c sleefsimddp.c)
    endif()

    if(COMPILER_SUPPORTS_${SIMD})
      foreach(SIMD_SOURCE ${SIMD_SOURCES})
        get_filename_component(SIMD_SOURCE_WITHOUT_EXT ${SIMD_SOURCE} NAME_WE)
        set(LLVM_BITCODE_INPUT ${CMAKE_CURRENT_SOURCE_DIR}/${SIMD_SOURCE})
        set(LLVM_BITCODE_OUTPUT ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/${SIMD_SOURCE_WITHOUT_EXT}_${SIMD}.ll)
        add_custom_command(OUTPUT ${LLVM_BITCODE_OUTPUT}
          COMMAND  ${CLANG_EXE_PATH} ${CLANG_FLAGS_ENABLE_${SIMD}} ${SLEEF_CLANG_LLVM_BITCODE_OPTIONS} -D ENABLE_${SIMD}=1 -o ${LLVM_BITCODE_OUTPUT} ${LLVM_BITCODE_INPUT} ${SLEEP_LLVM_BITCODE_INCLUDES_CLANG}
          DEPENDS
            ${LLVM_BITCODE_INPUT}
        )
        list(APPEND LLVM_BITCODE_OUTPUTS ${LLVM_BITCODE_OUTPUT})
      endforeach()
    endif()
  endforeach()

  file(MAKE_DIRECTORY ${CMAKE_LIBRARY_OUTPUT_DIRECTORY})
  add_custom_target(${TARGET_LLVM_BITCODE} ALL
    DEPENDS
      ${LLVM_BITCODE_OUTPUTS}
  )
  add_dependencies(${TARGET_LLVM_BITCODE} ${TARGET_HEADERS})

  install(FILES ${LLVM_BITCODE_OUTPUTS} DESTINATION lib)
endif()

# --------------------------------------------------------------------
# Install
# --------------------------------------------------------------------
# Install libsleef and sleef.h
install(FILES ${SLEEF_INCLUDE_HEADER} DESTINATION include)
install(TARGETS ${TARGET_LIBSLEEF} DESTINATION lib)

if(ENABLE_GNUABI)
  install(TARGETS ${TARGET_LIBSLEEFGNUABI} DESTINATION lib)
endif()
