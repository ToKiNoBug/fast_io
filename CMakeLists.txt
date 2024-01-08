cmake_minimum_required(VERSION 3.15)

project(fast_io VERSION 1.0 LANGUAGES C CXX)

INSTALL (DIRECTORY ${CMAKE_SOURCE_DIR}/include/ DESTINATION include)
INSTALL (DIRECTORY ${CMAKE_SOURCE_DIR}/man/ DESTINATION man/man3)

option(BUILD_I18N_DLLS "build i18n runtime dlls" OFF)
option(BUILD_TESTS "Build fast_io test" ON)

if(BUILD_I18N_DLLS)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_SHARED_MODULE_PREFIX "fast_io_i18n.locale.")
file(GLOB localefiles ${CMAKE_SOURCE_DIR}/src/i18n_data/locale/*.cc)

include(CheckCXXCompilerFlag)

function(compilelocaleencoding encoding localelist)

if(${encoding} STREQUAL "UTF-8")
	if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
		set(compilerencodingtoggle "/utf-8")
		CHECK_CXX_COMPILER_FLAG(${compilerencodingtoggle} COMPILER_SUPPORT_EXECUTION_CHARSET_${encoding})
	else()
		set(compilerencodingtoggle "")
		set(COMPILER_SUPPORT_EXECUTION_CHARSET_UTF-8 1)
	endif()
else()
	if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
		set(compilerencodingtoggle "/execution-charset:${encoding}")
	else()
		set(compilerencodingtoggle "-fexec-charset=${encoding}")
	endif()
	CHECK_CXX_COMPILER_FLAG(${compilerencodingtoggle} COMPILER_SUPPORT_EXECUTION_CHARSET_${encoding})
endif()

if(I18N_INSTALL_DIR)
else()
	set(I18N_INSTALL_DIR lib)
endif()

if(COMPILER_SUPPORT_EXECUTION_CHARSET_${encoding})
	if(localelist)
		foreach(filename ${localelist})
			string(REPLACE "@" "_" purfilename ${filename})
			add_library(${purfilename}.${encoding} MODULE "${CMAKE_SOURCE_DIR}/src/i18n_data/locale/${localelist}.cc")
			set_target_properties(${purfilename}.${encoding} PROPERTIES OUTPUT_NAME ${filename}.${encoding})
			if (NOT ${compilerencodingtoggle} STREQUAL "")
				target_compile_options(${purfilename}.${encoding} PRIVATE ${compilerencodingtoggle})
			endif()
			target_compile_definitions(${purfilename}.${encoding} PUBLIC
				FAST_IO_LOCALE_ENCODING="${encoding}"
				FAST_IO_LOCALE_LENCODING=L"${encoding}"
				FAST_IO_LOCALE_u8ENCODING=u8"${encoding}"
				FAST_IO_LOCALE_uENCODING=u"${encoding}"
				FAST_IO_LOCALE_UENCODING=U"${encoding}")
			install(TARGETS ${purfilename}.${encoding}
				LIBRARY DESTINATION ${I18N_INSTALL_DIR})
		endforeach()

	else()
		foreach(filepath ${localefiles})
			get_filename_component(filename ${filepath} NAME_WE)
			string(REPLACE "@" "_" purfilename ${filename})
			add_library(${purfilename}.${encoding} MODULE ${filepath})
			set_target_properties(${purfilename}.${encoding} PROPERTIES OUTPUT_NAME ${filename}.${encoding})
			if (NOT ${compilerencodingtoggle} STREQUAL "")
				target_compile_options(${purfilename}.${encoding} PRIVATE ${compilerencodingtoggle})
			endif()
			target_compile_definitions(${purfilename}.${encoding} PUBLIC
				FAST_IO_LOCALE_ENCODING="${encoding}"
				FAST_IO_LOCALE_LENCODING=L"${encoding}"
				FAST_IO_LOCALE_u8ENCODING=u8"${encoding}"
				FAST_IO_LOCALE_uENCODING=u"${encoding}"
				FAST_IO_LOCALE_UENCODING=U"${encoding}")
			install(TARGETS ${purfilename}.${encoding}
				LIBRARY DESTINATION ${I18N_INSTALL_DIR})
		endforeach()
	endif()
else()
	message("Compiler does not support \"${compilerencodingtoggle}\"; locale under ${encoding} execution charset will not build.")
endif()
endfunction()

compilelocaleencoding("UTF-8" false)
compilelocaleencoding("GB18030" false)
if(I18N_LOCALE_ENCODINGS)
foreach(encoding ${I18N_LOCALE_ENCODINGS})
	if (NOT proj STREQUAL "UTF-8" AND NOT encoding STREQUAL "GB18030")
		if(I18N_LOCALE_LIST)
			compilelocaleencoding(${encoding} ${I18N_LOCALE_LIST})
		else()
			compilelocaleencoding(${encoding} false)
		endif()
	endif()
endforeach()
else()
endif()

endif()

add_library(fast_io INTERFACE)
add_library(fast_io::fast_io ALIAS fast_io)
target_include_directories(fast_io INTERFACE
		$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
		$<INSTALL_INTERFACE:include>)
target_compile_features(fast_io INTERFACE cxx_std_20)
#if(${MSVC})
#	target_compile_options(fast_io INTERFACE "/Zc:__cplusplus")
#endif ()


if (BUILD_TESTS)
	file(GLOB test_dirs LIST_DIRECTORIES true "tests/*")
	add_custom_target("fast_io-tests-all")
	#message(STATUS "test_dirs = ${test_dirs}")
	foreach (dir ${test_dirs})
		cmake_path(GET dir FILENAME test_type)
		message("- ${test_type}")
		file(GLOB test_sources "${dir}/*.cc")

		add_custom_target(${test_type}
			COMMENT "Empty target to build all test below ${test_type}")
		add_dependencies("fast_io-tests-all" ${test_type})

		foreach (src_file ${test_sources})
			#message("  - ${src_file}")
			cmake_path(GET src_file STEM src_name_stem)
			set(target_name "${test_type}-${src_name_stem}")
			message("  - ${target_name}")
			add_executable(${target_name} ${src_file})
			target_link_libraries(${target_name} PRIVATE fast_io)

			set_target_properties(${target_name} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/tests/${test_type}")

			add_dependencies(${test_type} ${target_name})
		endforeach ()
	endforeach ()
endif ()

install(TARGETS fast_io
	EXPORT fast_io-targets)
install(EXPORT fast_io-targets
	NAMESPACE fast_io::
	DESTINATION lib/cmake/fast_io)

include(CMakePackageConfigHelpers)
write_basic_package_version_file("${CMAKE_BINARY_DIR}/fast_io-config.cmake"
	VERSION ${PROJECT_VERSION}
	COMPATIBILITY SameMajorVersion
	)
install(FILES "${CMAKE_BINARY_DIR}/fast_io-config.cmake"
	DESTINATION lib/cmake/fast_io)