# ---------------------------------------------------------------------------------------------
#  SPDX-License-Identifier: LGPL-3.0-or-later
#  SPDX-PackageName: Fix8 Open Source FIX Engine
#  SPDX-FileCopyrightText: Copyright (C) 2010-25 David L. Dight <fix@fix8.org>
#  SPDX-FileType: SOURCE
#  SPDX-Notice: >
#   Fix8 is released under the GNU LESSER GENERAL PUBLIC LICENSE Version 3.
#
#   Fix8 is free software: you can  redistribute it and / or modify  it under the  terms of the
#   GNU Lesser General  Public License as  published  by the Free  Software Foundation,  either
#   version 3 of the License, or (at your option) any later version.
#
#   Fix8 is distributed in the hope  that it will be useful, but WITHOUT ANY WARRANTY;  without
#   even the  implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
#
#   You should  have received a copy of the GNU Lesser General Public  License along with Fix8.
#   If not, see <https://www.gnu.org/licenses/>.
#
#   BECAUSE THE PROGRAM IS  LICENSED FREE OF  CHARGE, THERE IS NO  WARRANTY FOR THE PROGRAM, TO
#   THE EXTENT  PERMITTED  BY  APPLICABLE  LAW.  EXCEPT WHEN  OTHERWISE  STATED IN  WRITING THE
#   COPYRIGHT HOLDERS AND/OR OTHER PARTIES  PROVIDE THE PROGRAM "AS IS" WITHOUT WARRANTY OF ANY
#   KIND,  EITHER EXPRESSED   OR   IMPLIED,  INCLUDING,  BUT   NOT  LIMITED   TO,  THE  IMPLIED
#   WARRANTIES  OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE.  THE ENTIRE RISK AS TO
#   THE QUALITY AND PERFORMANCE OF THE PROGRAM IS WITH YOU. SHOULD THE PROGRAM PROVE DEFECTIVE,
#   YOU ASSUME THE COST OF ALL NECESSARY SERVICING, REPAIR OR CORRECTION.
#
#   IN NO EVENT UNLESS REQUIRED  BY APPLICABLE LAW  OR AGREED TO IN  WRITING WILL ANY COPYRIGHT
#   HOLDER, OR  ANY OTHER PARTY  WHO MAY MODIFY  AND/OR REDISTRIBUTE  THE PROGRAM AS  PERMITTED
#   ABOVE,  BE  LIABLE  TO  YOU  FOR  DAMAGES,  INCLUDING  ANY  GENERAL, SPECIAL, INCIDENTAL OR
#   CONSEQUENTIAL DAMAGES ARISING OUT OF THE USE OR INABILITY TO USE THE PROGRAM (INCLUDING BUT
#   NOT LIMITED TO LOSS OF DATA OR DATA BEING RENDERED INACCURATE OR LOSSES SUSTAINED BY YOU OR
#   THIRD PARTIES OR A FAILURE OF THE PROGRAM TO OPERATE WITH ANY OTHER PROGRAMS), EVEN IF SUCH
#   HOLDER OR OTHER PARTY HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGES.
# ---------------------------------------------------------------------------------------------
#  For Production-Grade FIX Requirements:
#   If you're  using Fix8 Community Edition and find  yourself needing higher throughput, lower
#   latency, or enterprise-grade reliability,Fix8Pro offers a robust upgrade path. Built on the
#   same  core  technology, Fix8Pro adds performance optimizations for  high-volume  messaging,
# 	 enhanced  API, professional  support  and  much  more —  making  it  ideal  for  production
#   deployments, low-latency trading, or  large-scale FIX  integrations.  It retains  near full
#   compatibility with  the Community Edition while providing  enhanced stability, scalability,
#   and  advanced  features  for demanding  environments.  If  your  project has  outgrown  the
#   Community  Edition's capabilities, you can find out and learn more about the Pro version at
#   www.fix8mt.com
# ---------------------------------------------------------------------------------------------
# 3.21 for PROJECT_IS_TOP_LEVEL: ON when fix8 is built on its own, OFF when it is added to
# another project with add_subdirectory() or FetchContent
cmake_minimum_required (VERSION 3.21)
include(cmake/buildutils.cmake)
project (fix8
   LANGUAGES CXX C
	HOMEPAGE_URL https://github.com/fix8/fix8
   DESCRIPTION "Open Source C++ FIX Engine"
   VERSION 2.0.1)

set(fix8_CMAKE_CXX_FLAGS_STR "${CMAKE_CXX_FLAGS}")
# ---------------------------------------------------------------------------------------------
# options
# ---------------------------------------------------------------------------------------------
if (NOT MSVC)
	fix8_setbuildtype(FIX8 Release)
endif()
message("-- ${BoldCyan}${PROJECT_NAME} version ${PROJECT_VERSION}, build type ${CMAKE_BUILD_TYPE}${Reset}")
fix8_addoption("FIX8_BUILD_UNITTESTS|enable building unit tests|${PROJECT_IS_TOP_LEVEL}")
fix8_addoption("FIX8_BUILD_EXAMPLES|enable building test applications (f8test f8print harness hftest)|${PROJECT_IS_TOP_LEVEL}")
fix8_addoption("FIX8_BUILD_ALL_WARNINGS|enable building with all warnings|true")
fix8_addoption("FIX8_BUILD_DOXYGEN_DOCS|enable building of self documentation|false")
fix8_addoption("FIX8_BUILD_MINIMAL_POCO|build only required Poco components|true")
fix8_addoption("FIX8_BUILD_EP264|enable building of FIX50SP2_EP264 schema library|false")
fix8_addoption("FIX8_BUILD_POCO_VERSION|Poco tag to download|poco-1.14.2-release")
fix8_addoption("FIX8_BUILD_TBB_VERSION|TBB tag to download|v2022.2.0")
fix8_addoption("FIX8_BUILD_GOOGLETEST_VERSION|GoogleTest tag to download|v1.18.0")
fix8_addoption("FIX8_BUILD_ZLIB_VERSION|zlib tag to download (windows only)|v1.3.1")
fix8_addoption("FIX8_TBB_MALLOC_PROXY|link TBB malloc proxy (replaces malloc/new for the whole process)|${PROJECT_IS_TOP_LEVEL}")
fix8_addoption("FIX8_INSTALL|generate install rules for fix8|${PROJECT_IS_TOP_LEVEL}")

set(fix8_all_schemas FIX40 FIX41 FIX42 FIX43 FIX44 FIX50 FIX50SP1 FIX50SP2)
if(NOT DEFINED FIX8_SCHEMAS) # a parent project may set this as a normal variable
	set(FIX8_SCHEMAS "${fix8_all_schemas}" CACHE STRING "FIX schema libraries to build")
endif()
foreach(schema ${FIX8_SCHEMAS})
	if(NOT schema IN_LIST fix8_all_schemas)
		message(FATAL_ERROR "${BoldRed}FIX8_SCHEMAS: unknown schema ${schema} (choose from ${fix8_all_schemas})${Reset}")
	endif()
endforeach()
message("-- Build: FIX schema libraries: ${Green}${FIX8_SCHEMAS}${Reset}")

# ---------------------------------------------------------------------------------------------
# deps
# a dependency whose targets already exist (e.g. the parent project found or fetched it) is
# used as-is; otherwise it is downloaded
# ---------------------------------------------------------------------------------------------
if (MSVC)
	fix8_fetch(zip madler/zlib ${FIX8_BUILD_ZLIB_VERSION})
	set(ziplib zlib)
	set(poco_ssl_libs "")
	add_library(getopt STATIC msvc/getopt/getopt.c)
	include_directories(msvc/getopt)
	set(getopt_lib $<TARGET_FILE:getopt>)
	copy_libs()
else()
	find_package(ZLIB REQUIRED)
	set(ziplib ZLIB::ZLIB)
	set(poco_ssl_libs Poco::NetSSL Poco::Crypto)
	set(getopt_lib "")
endif()

# ---------------------------------------------------------------------------------------------
# workaround for poco's c++17 check that fails for Intel compiler
if (CMAKE_CXX_COMPILER_ID MATCHES "IntelLLVM|Intel")
	function(check_for_cxx17_compiler)
		 message(STATUS "${BoldRed}Bypassing POCO's C++17 check for Intel compiler${Reset}")
		 set(CXX17_COMPILER TRUE PARENT_SCOPE)
	endfunction()
	file(WRITE "${PROJECT_BINARY_DIR}/dummy/CXX1x.cmake"
	 "# This empty file ensures our function override takes precedence")
	list(INSERT CMAKE_MODULE_PATH 0 "${PROJECT_BINARY_DIR}/dummy")
endif()
if(NOT TARGET Poco::Foundation)
	# Poco's generic option names would otherwise pick up a parent project's ENABLE_TESTS etc.
	set(ENABLE_TESTS OFF)
	set(ENABLE_SAMPLES OFF)
	if(FIX8_BUILD_MINIMAL_POCO)
		# normal (not cache) variables: Poco's option() calls honour them (CMP0077) and they stay
		# scoped to this directory instead of overriding Poco settings for the whole build
		foreach(poco_opt ACTIVERECORD ACTIVERECORD_COMPILER PAGECOMPILER PAGECOMPILER_FILE2PAGE DATA
				MONGODB PROMETHEUS REDIS SQLITE DATA_SQLITE DATA_MYSQL DATA_ODBC DATA_POSTGRESQL
				APACHECONNECTOR FOUNDATION_APR JWT ZIP ENCODINGS JSON XML)
			set(ENABLE_${poco_opt} OFF)
		endforeach()
	endif()
	set(fix8_tmp_cache_vars "")
	if(NOT PROJECT_IS_TOP_LEVEL)
		# Poco declares option(BUILD_SHARED_LIBS ON) and caches CMAKE_<KIND>_OUTPUT_DIRECTORY under
		# the top-level binary dir, both of which would leak into the parent project. Shadow
		# BUILD_SHARED_LIBS here (fix8 is built and tested against shared deps), and pre-seed any
		# output directory the parent has not set, removing it again once Poco is configured.
		set(BUILD_SHARED_LIBS ON)
		foreach(kind RUNTIME LIBRARY ARCHIVE)
			foreach(config "" ${CMAKE_CONFIGURATION_TYPES})
				if(config STREQUAL "")
					set(var CMAKE_${kind}_OUTPUT_DIRECTORY)
				else()
					string(TOUPPER ${config} config_upper)
					set(var CMAKE_${kind}_OUTPUT_DIRECTORY_${config_upper})
				endif()
				if(NOT DEFINED ${var})
					if(kind STREQUAL "RUNTIME")
						set(${var} ${PROJECT_BINARY_DIR}/bin CACHE PATH "")
					else()
						set(${var} ${PROJECT_BINARY_DIR}/lib CACHE PATH "")
					endif()
					list(APPEND fix8_tmp_cache_vars ${var})
				endif()
			endforeach()
		endforeach()
	endif()
	fix8_fetch(poco pocoproject/poco ${FIX8_BUILD_POCO_VERSION})
	foreach(var ${fix8_tmp_cache_vars})
		set(value $CACHE{${var}})
		unset(${var} CACHE)
		set(${var} ${value}) # keep fix8's own outputs in the same place, scoped to fix8
	endforeach()
endif()
if(NOT TARGET TBB::tbb)
	set(TBB_TEST OFF)
	# oneTBB forces CMAKE_BUILD_TYPE=RelWithDebInfo into the cache when it is empty; as a
	# subproject, put the parent's (empty) build type back afterwards
	set(fix8_parent_build_type "$CACHE{CMAKE_BUILD_TYPE}")
	get_property(fix8_build_type_doc CACHE CMAKE_BUILD_TYPE PROPERTY HELPSTRING)
	fix8_fetch(tbb uxlfoundation/oneTBB ${FIX8_BUILD_TBB_VERSION})
	if(NOT PROJECT_IS_TOP_LEVEL AND NOT "$CACHE{CMAKE_BUILD_TYPE}" STREQUAL "${fix8_parent_build_type}")
		set(CMAKE_BUILD_TYPE "${fix8_parent_build_type}" CACHE STRING "${fix8_build_type_doc}" FORCE)
	endif()
endif()

# ---------------------------------------------------------------------------------------------
# compiler settings
# ---------------------------------------------------------------------------------------------
cpp_opts()

# ---------------------------------------------------------------------------------------------
# runtime
# ---------------------------------------------------------------------------------------------
set(fix8_libs Poco::Foundation Poco::Net Poco::Util ${poco_ssl_libs} ${ziplib} TBB::tbb)
macro(build_lib name type)
	add_library(${name} ${type} ${rsrc})
	add_library(fix8::${name} ALIAS ${name})
	target_link_libraries(${name} PUBLIC ${fix8_libs})
	# the generated f8config.h (binary dir) comes first so a stale copy left in the source tree
	# by an older build cannot shadow it
	target_include_directories(${name} PUBLIC
		$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
		$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
		$<INSTALL_INTERFACE:include>)
	target_compile_features(${name} PUBLIC cxx_std_17)
	if (MSVC)
		add_dependencies(${name} lib_copy ${fix8_libs})
	endif()
	comp_opts(${name})
	if(FIX8_INSTALL)
		install(TARGETS ${name})
	endif()
endmacro()

math(EXPR fix8_MAGIC_NUM "${fix8_VERSION_MAJOR} << 24 | ${fix8_VERSION_MINOR} << 12 | ${fix8_VERSION_PATCH}")
string(TIMESTAMP fix8_SDATE "%Y/%m/%d")
string(TIMESTAMP fix8_TIMESTAMP)
configure_file(${PROJECT_SOURCE_DIR}/f8config.h.in ${PROJECT_BINARY_DIR}/include/fix8/f8config.h @ONLY)
set(rtdir ${PROJECT_SOURCE_DIR}/runtime)
file(GLOB rsrc "${rtdir}/*.cpp" "${rtdir}/*.c")
build_lib(fix8 SHARED)
set_target_properties(fix8 PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})
target_compile_definitions(fix8 PRIVATE BUILD_F8API)
build_lib(fix8stat STATIC)
# consumers of the static library also need F8API to expand to nothing on msvc
target_compile_definitions(fix8stat PUBLIC BUILD_F8API_STATIC)
if (NOT MSVC)
	set_target_properties(fix8stat PROPERTIES OUTPUT_NAME fix8)
endif()
if(FIX8_INSTALL)
	install(DIRECTORY include DESTINATION .)
	install(FILES ${PROJECT_BINARY_DIR}/include/fix8/f8config.h DESTINATION include/fix8)
	install(DIRECTORY schema DESTINATION share)
	install(DIRECTORY contrib DESTINATION share)
endif()

# ---------------------------------------------------------------------------------------------
# compiler
# ---------------------------------------------------------------------------------------------
set(compdir ${PROJECT_SOURCE_DIR}/compiler)
file(GLOB csrc "${compdir}/*.cpp")
add_executable(f8c ${csrc})
add_executable(fix8::f8c ALIAS f8c)
target_link_libraries(f8c PUBLIC fix8 ${getopt_lib})
target_include_directories(f8c PRIVATE ${compdir} include)
comp_opts(f8c)
if(FIX8_INSTALL)
	install(TARGETS f8c)
endif()

# ---------------------------------------------------------------------------------------------
# util
# ---------------------------------------------------------------------------------------------
set(utildir ${PROJECT_SOURCE_DIR}/util)
add_executable(seqedit ${utildir}/seqedit.cpp)
target_link_libraries(seqedit PUBLIC fix8 ${getopt_lib})
target_include_directories(seqedit PRIVATE ${utildir} include)
comp_opts(seqedit)
if(FIX8_INSTALL)
	install(TARGETS seqedit)
endif()

# ---------------------------------------------------------------------------------------------
# shared standard schema libs (FIX8_SCHEMAS selects which)
# ---------------------------------------------------------------------------------------------
foreach(schema ${FIX8_SCHEMAS})
	if(schema MATCHES "^FIX5")
		fix8_gen_shared_library(${schema} schema/${schema}.xml --verbose --namespace ${schema} -p ${schema} --fixt ${PROJECT_SOURCE_DIR}/schema/FIXT11.xml)
	else()
		fix8_gen_shared_library(${schema} schema/${schema}.xml --verbose --namespace ${schema} -p ${schema})
	endif()
endforeach()
if(FIX8_INSTALL AND FIX8_SCHEMAS)
	install(TARGETS ${FIX8_SCHEMAS})
endif()
if(FIX8_BUILD_EP264)
	fix8_gen_shared_library(FIX50SP2_EP264 schema/FIX50SP2_EP264.xml --verbose --namespace FIX50SP2_EP264 -p FIX50SP2_EP264 --fixt ${PROJECT_SOURCE_DIR}/schema/FIXT11_EP264.xml)
	if(FIX8_INSTALL)
		install(TARGETS FIX50SP2_EP264)
	endif()
endif()

# ---------------------------------------------------------------------------------------------
# unit tests
# ---------------------------------------------------------------------------------------------
if((FIX8_BUILD_UNITTESTS OR FIX8_BUILD_EXAMPLES) AND NOT TARGET GTest::gtest)
	if(NOT FIX8_INSTALL)
		set(INSTALL_GTEST OFF)
	endif()
	fix8_fetch(googletest google/googletest ${FIX8_BUILD_GOOGLETEST_VERSION})
endif()
if(FIX8_BUILD_UNITTESTS)
	set(EXTRA_FIELDS "<field number='9999' name='SampleUserField'  type='STRING' messages='NewOrderSingle:N ExecutionReport:N OrderCancelRequest:Y' />\
<field number='9991' name='SampleUserField2' type='STRING' messages='NewOrderSingle:N ExecutionReport:N OrderCancelRequest:Y' />")
	fix8_gen_shared_library(utest schema/FIX42UTEST.xml --verbose -F ${EXTRA_FIELDS} -p utest --namespace UTEST)
	include(GoogleTest)
	enable_testing()
	build_test(message_test "utests/message_test.cpp")
	build_test(fileLogger_test "utests/fileLogger_test.cpp")
	build_test(filePersister_test "utests/filePersister_test.cpp")
	build_test(session_test "utests/session_test.cpp;utests/mockConnection.cpp")
endif()

# ---------------------------------------------------------------------------------------------
# test apps
# ---------------------------------------------------------------------------------------------
if(FIX8_BUILD_EXAMPLES)
	add_executable(f8test test/myfix.cpp)
	fix8_gen_shared_library(myfix schema/FIX50SP2.xml --verbose --namespace MYFIX --prefix myfix --fixt ${PROJECT_SOURCE_DIR}/schema/FIXT11.xml)
	target_link_libraries(f8test PUBLIC fix8 myfix GTest::gtest GTest::gtest_main ${getopt_lib})
	target_include_directories(f8test PRIVATE test include ${PROJECT_BINARY_DIR}/generated/myfix)
	target_compile_definitions(f8test PRIVATE F8_MYFIX_API_SHARED)
	comp_opts(f8test)

	add_executable(f8print test/myprint.cpp)
	target_link_libraries(f8print PUBLIC fix8 myfix GTest::gtest GTest::gtest_main ${getopt_lib})
	target_include_directories(f8print PRIVATE test include ${PROJECT_BINARY_DIR}/generated/myfix)
	target_compile_definitions(f8print PRIVATE F8_MYFIX_API_SHARED)
	comp_opts(f8print)

	add_executable(harness test/harness.cpp)
	target_link_libraries(harness PUBLIC fix8 myfix GTest::gtest GTest::gtest_main ${getopt_lib})
	target_include_directories(harness PRIVATE test include ${PROJECT_BINARY_DIR}/generated/myfix)
	target_compile_definitions(harness PRIVATE F8_MYFIX_API_SHARED)
	comp_opts(harness)

	add_executable(hftest test/hftest.cpp)
	fix8_gen_shared_library(perf schema/FIX42PERF.xml --verbose --namespace PERF --second --noconst --prefix perf --norealm)
	target_link_libraries(hftest PUBLIC fix8 perf GTest::gtest GTest::gtest_main ${getopt_lib})
	target_include_directories(hftest PRIVATE test include ${PROJECT_BINARY_DIR}/generated/perf)
	target_compile_definitions(hftest PRIVATE F8_PERF_API_SHARED)
	comp_opts(hftest)

	if(FIX8_INSTALL)
		install(TARGETS f8test f8print harness harness)
		install(FILES ${PROJECT_SOURCE_DIR}/test/hf_client.xml ${PROJECT_SOURCE_DIR}/test/myfix_client.xml ${PROJECT_SOURCE_DIR}/test/hf_client_include.xml
		   ${PROJECT_SOURCE_DIR}/test/hf_server.xml ${PROJECT_SOURCE_DIR}/test/myfix_server.xml DESTINATION share/test)
	endif()
endif()

# ---------------------------------------------------------------------------------------------
# generated docs
# ---------------------------------------------------------------------------------------------
if(FIX8_BUILD_DOXYGEN_DOCS)
	find_package(Doxygen REQUIRED)
	find_package(FLEX REQUIRED)
	add_subdirectory(doc)
	if(TARGET myfix)
		add_dependencies(fix8_doxygen myfix)
	endif()
	if(FIX8_INSTALL)
		install(DIRECTORY ${PROJECT_BINARY_DIR}/doc/html DESTINATION share/doc)
	endif()
endif()

# ---------------------------------------------------------------------------------------------
# to dump all non-empty cmake variables
# cmake -DFIX8_DUMP_VARS=true ..
# ---------------------------------------------------------------------------------------------
if(PROJECT_IS_TOP_LEVEL AND NOT DEFINED FIX8_DUMP_VARS AND DEFINED CMAKE_DUMP_VARS)
	set(FIX8_DUMP_VARS ${CMAKE_DUMP_VARS}) # legacy name
endif()
option(FIX8_DUMP_VARS "dump all cmake vars" false)
if(FIX8_DUMP_VARS)
   get_cmake_property(varname VARIABLES)
   foreach (var ${varname})
      if(NOT ${var} STREQUAL "")
         message(STATUS "${var}=${${var}}")
      endif()
   endforeach()
endif()

