# CMakeLists.txt for ucpp
# Adapted from Makefile
#
# (c) Thomas Pornin 1999 - 2002
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
# 1. Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
# 4. The name of the authors may not be used to endorse or promote
#    products derived from this software without specific prior written
#    permission.
#
# THIS SOFTWARE IS PROVIDED ``AS IS'' AND WITHOUT ANY EXPRESS OR 
# IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
# WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE AUTHORS OR CONTRIBUTORS BE
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT
# OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR 
# BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
# WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE 
# OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE,
# EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

project( ucpp C )
cmake_minimum_required( VERSION 2.6 )

# ----- user configurable part -----

# Edit the variables to suit your system.
#
#   use -DAUDIT to enable some internal sanity checks
#   use -DMEM_CHECK to check the return value of malloc()
#      (superseded by AUDIT)
#   use -DMEM_DEBUG to enable memory leak research (warning: this
#      slows down ucpp a bit, and greatly increases memory consumption)
#   use -DINLINE=foobar to enable use of the 'foobar'
#      non standard qualifier, as an equivalent to the C99 'inline'
#      qualifier. See tune.h for details.
#
# Two FLAGS lines are given for each system type; chose the first one for
# debug, the second one for a fast binary.

set( CMAKE_C_FLAGS       "${CMAKE_C_FLAGS} -O3 -W -Wall -ansi" )
set( CMAKE_C_FLAGS_DEBUG 
     "${CMAKE_C_FLAGS_DEBUG} -g  -W -Wall -ansi -DAUDIT -DMEM_DEBUG" )

set( CMAKE_C_FLAGS_RELEASE 
    "${CMAKE_C_FLAGS_RELEASE} -O3 -W -Wall -ansi -mcpu=pentiumpro -fomit-frame-pointer -DMEM_CHECK" )
set( CMAKE_C_FLAGS_PROFILE 
     "${CMAKE_C_FLAGS_PROFILE} -O  -W -Wall -ansi -DMEM_CHECK -pg" )

set( ucpp_SOURCE arith.c
                 arith.h
                 assert.c 
                 config.h
                 cpp.c
                 cpp.h 
                 eval.c
                 lexer.c 
                 macro.c 
                 mem.c 
                 mem.h
                 nhash.c 
                 nhash.h 
                 ucppi.h )

set_source_files_properties( arith.c PROPERTIES HEADER_FILE_ONLY TRUE )
set_source_files_properties( arith.h PROPERTIES HEADER_FILE_ONLY TRUE )
set_source_files_properties( config.h PROPERTIES GENERATED TRUE )

include( CheckFunctionExists )

option( STAND_ALONE              "Build the stand alone version of ucpp" OFF )
option( LOW_MEM                  "Build the low memory version of ucpp" ON )
option( AMIGA                    "Amiga file paths (drive letters)" OFF )
option( MSDOS                    "MsDos file paths (backslashes)" OFF )
option( PRAGMA_TOKENIZE          "Generate tokenized PRAGMA tokens" ON )
option( PRAGMA_DUMP              "Include pragma directives in output in non-lexer mode" ON )
option( NO_PRAGMA_IN_DIRECTIVE   "Disable evaluation of _Pragma() in directives" OFF )
option( DSHARP_TOKEN_MERGE       "Merge incompatible tokens" OFF )
option( INMACRO_FLAG             "Include macro replacement flags for tokens in lexer_state" OFF )
option( DEFAULT_STD_INCLUDE_PATH "Use a sensible default include path" OFF )
option( NATIVE_MATH              "Use the native types for #if arithmetic" ON )
option( ARITHMETIC_CHECKS        "Perform checks on arithmetic" ON )
set( PRAGMA_TOKEN_END     "((unsigned char)'\\n')" CACHE STRING "Character to mark the end of strings in a PRAGMA" )
set( WCHAR_SIGNEDNESS     ""                      CACHE STRING "Override the signedness for wchars" )
set( STD_ASSERT          "0"                      CACHE STRING "Standard assertions" )
set( STD_MACROS          "0"                      CACHE STRING "Standard macros" )
set( DEFAULT_CPP_FLAGS   "(DISCARD_COMMENTS | WARN_STANDARD | WARN_PRAGMA | FAIL_SHARP | MACRO_VAARG | CPLUSPLUS_COMMENTS | LINE_NUM | TEXT_OUTPUT | KEEP_OUTPUT | HANDLE_TRIGRAPHS | HANDLE_ASSERTIONS)"   CACHE STRING "Default cpp flags" )
set( DEFAULT_LEXER_FLAGS "(DISCARD_COMMENTS | WARN_STANDARD | FAIL_SHARP | MACRO_VAARG | CPLUSPLUS_COMMENTS | LEXER | HANDLE_TRIGRAPHS | HANDLE_ASSERTIONS)" CACHE STRING "Default lexer flags" )
option( POSIX_JMP               "use sigsetjmp()/siglongjmp() instead of setjmp()/longjmp()" OFF )
set( MAX_CHAR_VAL        "128"                    CACHE STRING "Maximum value (plus one) of a character handled by the lexer" )
set( UNBREAKABLE_SPACE   ""                       CACHE STRING "Extra character to be considered as whitespace. On ISO-8859-1 machines, 160 is the code for the unbreakable space" )
option( SEMPER_FIDELIS   "Record whitespace tokens" OFF )


#
# Variables derived from the above options
#
set( NATIVE_SIGNED        "" CACHE STRING "Native signed type" )
set( NATIVE_UNSIGNED      "" CACHE STRING "Native unsigned type" )
set( NATIVE_UNSIGNED_BITS "" CACHE STRING "Native unsigned bits" )
set( NATIVE_SIGNED_MIN    "" CACHE STRING "Native signed min value" )
set( NATIVE_SIGNED_MAX    "" CACHE STRING "Native signed max value" )
set( SIMUL_ARITH_SUBTYPE  "" CACHE STRING "The simulatated arithmetic subtype" )
set( SIMUL_SUBTYPE_BITS   "" CACHE STRING "The simulatated arithmetic subtype size in bits" )
set( SIMUL_NUMBITS        "" CACHE STRING "The simulatated arithmetic size in bits" )
if( NATIVE_MATH )
    set( NATIVE_SIGNED        "long long" )
    set( NATIVE_UNSIGNED      "unsigned long long" )
    set( NATIVE_UNSIGNED_BITS "64" )
    set( NATIVE_SIGNED_MIN    "(-9223372036854775807LL - 1)" )
    set( NATIVE_SIGNED_MAX    "9223372036854775807LL" )
else()
    set( SIMUL_ARITH_SUBTYPE  "unsigned long" )
    set( SIMUL_SUBTYPE_BITS   "32" )
    set( SIMUL_NUMBITS        "64" )
endif()

set( STD_INCLUDE_PATH "0" CACHE STRING "The standard system include path" )
if( DEFAULT_STD_INCLUDE_PATH )
    set( STD_INCLUDE_PATH "\"/usr/local/include\", \"/usr/include\"" )
endif()

check_function_exists( strftime HAVE_STRFTIME )
if( NOT HAVE_STRFTIME )
    set( NOSTRFTIME ON CACHE BOOL "The system doesn't have strftime" )
endif()

CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_SOURCE_DIR}/config.h)

if( UCPP_STAND_ALONE )
    add_definitions( -DSTAND_ALONE )
    add_executable( ucpp ${ucpp_SOURCE} )
else()
    remove_definitions( -DSTAND_ALONE )
    add_library( ucpp ${ucpp_SOURCE} )
endif()
