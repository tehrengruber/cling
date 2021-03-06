#------------------------------------------------------------------------------
# CLING - the C++ LLVM-based InterpreterG :)
#
# This file is dual-licensed: you can choose to license it under the University
# of Illinois Open Source License or the GNU Lesser General Public License. See
# LICENSE.TXT for details.
#------------------------------------------------------------------------------

set(LIBS
  clangDriver
  clangFrontend
  clangParse
  clangSema
  clangAST
  clangLex
  clangSerialization
  clangCodeGen
  clangBasic

  clingUtils
)

find_library(DL_LIBRARY_PATH dl)
if (DL_LIBRARY_PATH)
  list(APPEND LIBS dl)
endif()


set( LLVM_LINK_COMPONENTS
  analysis
  core
  executionengine
  ipo
  mc
  native
  nativecodegen
  object
  option
  orcjit
  runtimedyld
  support
  target
)

add_cling_library(clingInterpreter OBJECT
  AutoSynthesizer.cpp
  AutoloadCallback.cpp
  ASTTransformer.cpp
  BackendPasses.cpp
  CheckEmptyTransactionTransformer.cpp
  CIFactory.cpp
  ClangInternalState.cpp
  ClingCodeCompleteConsumer.cpp
  ClingPragmas.cpp
  DeclCollector.cpp
  DeclExtractor.cpp
  DeclUnloader.cpp
  DynamicLibraryManager.cpp
  DynamicLookup.cpp
  DynamicExprInfo.cpp
  Exception.cpp
  ExceptionRTTI.cpp
  ExternalInterpreterSource.cpp
  ForwardDeclPrinter.cpp
  IncrementalExecutor.cpp
  IncrementalJIT.cpp
  IncrementalParser.cpp
  Interpreter.cpp
  InterpreterCallbacks.cpp
  InvocationOptions.cpp
  LookupHelper.cpp
  NullDerefProtectionTransformer.cpp
  RequiredSymbols.cpp
  Transaction.cpp
  TransactionUnloader.cpp
  ValueExtractionSynthesizer.cpp
  Value.cpp
  ValuePrinter.cpp
  ValuePrinterSynthesizer.cpp

  LINK_LIBS
  ${LIBS}
)

if(NOT WIN32)
set_source_files_properties(ExceptionRTTI.cpp COMPILE_FLAGS "-fexceptions -frtti")
endif()

#set_source_files_properties(Exception.cpp COMPILE_FLAGS " /EHsc ")
# the line above doesn't work, and it gives the following warnings:
# cl : Command line warning D9025: overriding '/EHs' with '/EHs-'
# cl : Command line warning D9025: overriding '/EHc' with '/EHc-'
# so let's change the target compiler flags instead:
if(MSVC)
  add_definitions(-UUNICODE -U_UNICODE)
  get_target_property(CI_COMPILE_FLAGS clingInterpreter COMPILE_FLAGS)
  if(${CI_COMPILE_FLAGS})
    string(REPLACE "/EHs-c-" "" CI_COMPILE_FLAGS ${CI_COMPILE_FLAGS})
    # add /EHsc exception handling flag
    set(CI_COMPILE_FLAGS "${CI_COMPILE_FLAGS} /EHsc")
  else()
    set(CI_COMPILE_FLAGS "/EHsc")
  endif()
  set_target_properties(clingInterpreter PROPERTIES
    COMPILE_FLAGS ${CI_COMPILE_FLAGS})
endif()

#add_dependencies(clangDriver ClangAttrList ClangDiagnosticDriver
#                 ClangDriverOptions ClangCC1Options ClangCC1AsOptions)

if (UNIX)

  # Remove all -I from CMAKE_CXX_FLAGS
  string(REPLACE ";" " " __flags "${CMAKE_CXX_FLAGS}")
  string(REGEX REPLACE "-I[^ ]+" "" CLING_COMPILER_FLAGS_NO_I "${__flags}")

  option(CLING_CXX_PATH "Compiler cling will invoke for c++ headers." "")
  option(CLING_CXX_HEADERS "Path cling will use for c++ headers." "")

  function(stripNewLine strVal varName)
    string(STRIP "${strVal}" strVal)
    string(REGEX REPLACE "\\n$" "" strVal "${strVal}")
    SET(${varName} ${strVal} PARENT_SCOPE)
  endfunction()

  if(NOT CLING_CXX_PATH)
    # Remove absolute path from CMAKE_CXX_COMPILER
    get_filename_component(_name ${CMAKE_CXX_COMPILER} NAME)
    get_filename_component(_path ${CMAKE_CXX_COMPILER} PATH)

    # This should probably be more general...but how?
    if(_name STREQUAL "ccache" OR _name STREQUAL "distcc")
      separate_arguments(_arg_list UNIX_COMMAND "${CMAKE_CXX_COMPILER_ARG1}")
      if (_arg_list)
        list(GET _arg_list 0 _name)
        string(STRIP "${_name}" _name)
        if (APPLE)
          execute_process(COMMAND xcrun -f ${_name}
                          OUTPUT_VARIABLE CLING_CXX_FOUND
                          OUTPUT_STRIP_TRAILING_WHITESPACE)
          stripNewLine("${CLING_CXX_FOUND}" CLING_CXX_FOUND)
        else()
          find_program(_cling_cxx_path "${_name}")
          execute_process(COMMAND ${_cling_cxx_path} -xc++ -E -v /dev/null
                          OUTPUT_QUIET ERROR_VARIABLE _cling_cxx_path)

          if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
            execute_process(
              COMMAND echo ${_cling_cxx_path}
              COMMAND grep "COLLECT_GCC="
              OUTPUT_VARIABLE _cling_cxx_path)
              string(REPLACE "COLLECT_GCC=" "" _cling_cxx_path "${_cling_cxx_path}")

          elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
            execute_process(
              COMMAND echo ${_cling_cxx_path}
              COMMAND grep "/${_name}\" -cc1"
              OUTPUT_VARIABLE _cling_clng_path)

            if(NOT _cling_clng_path)
              execute_process(
                COMMAND echo ${_cling_cxx_path}
                COMMAND grep "/clang\" -cc1"
                OUTPUT_VARIABLE _cling_clng_path)
            endif()

            separate_arguments(_arg_list UNIX_COMMAND "${_cling_clng_path}")
            if (_arg_list)
              list(GET _arg_list 0 _cling_cxx_path)
            endif()
          endif()

          stripNewLine("${_cling_cxx_path}" _cling_cxx_path)
          set(CLING_CXX_FOUND "${_cling_cxx_path}")
        endif()

        if (NOT EXISTS "${CLING_CXX_FOUND}")
          find_program(CLING_CXX_FOUND "${_name}")
        endif()
      else()
        set(CLING_CXX_FOUND "")
        set(_name "")
      endif()

      if (EXISTS ${CLING_CXX_FOUND})
        set(CLING_CXX_PATH ${CLING_CXX_FOUND})
        get_filename_component(_name ${CLING_CXX_PATH} NAME)
        get_filename_component(_path ${CLING_CXX_PATH} PATH)
      else()
        set(CLING_CXX_PATH "${CMAKE_CXX_COMPILER} ${CMAKE_CXX_COMPILER_ARG1}")
        if(_name)
          set(CLING_CXX_RLTV "${_name}")
        endif()
        set(_path "__THISREALLYBETTERNOTBEINPATH_THANKS__")
      endif()
    else()
      get_filename_component(_path ${CMAKE_CXX_COMPILER} PATH)
    endif()

    if("$ENV{PATH}" MATCHES ${_path})
      set(CLING_CXX_RLTV ${_name})
    elseif(NOT CLING_CXX_PATH)
      set(CLING_CXX_PATH ${CMAKE_CXX_COMPILER})
    endif()
  endif()

  if(NOT CLING_CXX_HEADERS)
    if (CLING_CXX_PATH)
      execute_process(COMMAND ${CLING_CXX_PATH} -xc++ -E -v /dev/null
                      OUTPUT_QUIET ERROR_VARIABLE CLING_CXX_HEADERS)
    else()
      execute_process(COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_COMPILER_ARG1} -xc++ -E -v /dev/null
                      OUTPUT_QUIET ERROR_VARIABLE CLING_CXX_HEADERS)
    endif()

    execute_process(
      COMMAND echo ${CLING_CXX_HEADERS}
      COMMAND awk "/^#include </,/^End of search/{if (!/^#include </ && !/^End of search/){ print }}"
      COMMAND grep -E "(c|g)\\+\\+"
      OUTPUT_VARIABLE CLING_CXX_HEADERS)

    stripNewLine("${CLING_CXX_HEADERS}" CLING_CXX_HEADERS)
  endif()

  if (NOT EXISTS ${CLING_CXX_HEADERS})
    string(REPLACE "\n" ";" _cxx_inc_paths ${CLING_CXX_HEADERS})
    foreach(_cxx_inc_path ${_cxx_inc_paths})
      string(STRIP "${_cxx_inc_path}" _cxx_inc_path)
      if (NOT EXISTS ${_cxx_inc_path})
        set(_cxx_inc_join "")
        break()
      endif()
      if(_cxx_inc_join)
        set(_cxx_inc_join "${_cxx_inc_join}:${_cxx_inc_path}")
      else()
        set(_cxx_inc_join "${_cxx_inc_path}")
      endif()
    endforeach()
    set(CLING_CXX_HEADERS "${_cxx_inc_join}")
    if (NOT CLING_CXX_HEADERS)
      MESSAGE(WARNING "Cannot determine location of C++ headers for runtime.")
    endif()
  endif()

  MESSAGE(STATUS "Cling will look for C++ headers in '${CLING_CXX_HEADERS}' at runtime.")

  if (CLING_CXX_PATH)
    MESSAGE(STATUS "And if not found, will invoke: '${CLING_CXX_PATH}' for them.")
    if (CLING_CXX_RLTV)
      MESSAGE(STATUS "And then fallback to: '${CLING_CXX_RLTV}'")
      file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h.in
        "
        #define CLING_CXX_INCL \"${CLING_CXX_HEADERS}\"
        #define CLING_CXX_PATH \"${CLING_CXX_PATH} ${CMAKE_CXX_FLAGS_NO_I} ${CMAKE_CXX_FLAGS_${uppercase_CMAKE_BUILD_TYPE}}\"
        #define CLING_CXX_RLTV \"${CLING_CXX_RLTV} ${CMAKE_CXX_FLAGS_NO_I} ${CMAKE_CXX_FLAGS_${uppercase_CMAKE_BUILD_TYPE}}\"
        #define CLING_INCLUDE_PATHS \"${CLING_INCLUDE_PATHS}\"
      ")
    else()
      file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h.in
        "
        #define CLING_CXX_INCL \"${CLING_CXX_HEADERS}\"
        #define CLING_CXX_PATH \"${CLING_CXX_PATH} ${CMAKE_CXX_FLAGS_NO_I} ${CMAKE_CXX_FLAGS_${uppercase_CMAKE_BUILD_TYPE}}\"
        #define CLING_INCLUDE_PATHS \"${CLING_INCLUDE_PATHS}\"
      ")
    endif()
  else()
    MESSAGE(STATUS "And if not found, will invoke: '${CLING_CXX_RLTV}' for them.")
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h.in
      "
      #define CLING_CXX_INCL \"${CLING_CXX_HEADERS}\"
      #define CLING_CXX_RLTV \"${CLING_CXX_RLTV} ${CMAKE_CXX_FLAGS_NO_I} ${CMAKE_CXX_FLAGS_${uppercase_CMAKE_BUILD_TYPE}}\"
      #define CLING_INCLUDE_PATHS \"${CLING_INCLUDE_PATHS}\"
    ")
  endif()

else()
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h.in
      "
      #define CLING_INCLUDE_PATHS \"${CLING_INCLUDE_PATHS}\"
    ")
endif()

# Make sure this goes last so so we can pick up any changes that occured
# Also means cling-compiledata.h.in should be edited never cling-compiledata.h

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h
                   COMMAND ${CMAKE_COMMAND} -E copy_if_different
                   ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h.in
                   ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h
                   MAIN_DEPENDENCY ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h.in
                   COMMENT "Updating cling-compiledata.h")

add_file_dependencies(${CMAKE_CURRENT_SOURCE_DIR}/CIFactory.cpp
                      ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h)
