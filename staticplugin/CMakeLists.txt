cmake_minimum_required(VERSION 3.20)

set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: Debug Release.")

project(RNBO_PLUGINS
	VERSION 0.1.0
	LANGUAGES CXX
)

include(../../../cmake/RNBOConan.cmake)
include(../../../cmake/CCache.cmake)
include(../../../cmake/MinGWSTDThreads.cmake)

set (CMAKE_CXX_STANDARD 17)

#juce requires 10.9 or later
set(CMAKE_OSX_DEPLOYMENT_TARGET 10.9)

set(PLUGIN_NAME "RNBO Plugin" CACHE STRING "The name of the audio plugin")
set(PLUGIN_VERSION "0.0.1" CACHE STRING "Plugin version string")
set(PLUGIN_VERSION_NUM "1" CACHE STRING "Plugin version number integer")

# GarageBand 10.3 requires the first letter to be upper-case, and the remaining letters to be lower-case
set(PLUGIN_CODE "Rnb0" CACHE STRING "A unique four-character plugin id with at least one upper-case character")
set(PLUGIN_MANUFACTURER_CODE "Cc74" CACHE STRING "A four-character manufacturer id with at least one upper-case character")
set(PLUGIN_MANUFACTURER_NAME "Cycling '74" CACHE STRING "Specify the name of the plugin's author")
set(PLUGIN_MANUFACTURER_WEBSITE "http://www.cycling74.com" CACHE STRING "Specify the URL of the plugin's website")
set(PLUGIN_MANUFACTURER_EMAIL "info@cycling74.com" CACHE STRING "Specify the plugin author's email address")


set(PLUGIN_VST3_MAC			OFF CACHE BOOL "Export VST3 Plugin for MacOS")
set(PLUGIN_VST3_WIN			OFF CACHE BOOL "Export VST3 Plugin for Windows")
set(PLUGIN_VST3_LINUX 	OFF CACHE BOOL "Export VST3 Plugin for Linux")
set(PLUGIN_AU_MAC				OFF CACHE BOOL "Export AU Plugin for MacOS")
set(PLUGIN_UNITY_MAC		OFF CACHE BOOL "Export Unity Plugin MacOS")
set(PLUGIN_UNITY_WIN		OFF CACHE BOOL "Export Unity Plugin Windows")
set(PLUGIN_UNITY_LINUX	OFF CACHE BOOL "Export Unity Plugin Linux")

set(PLUGIN_BUSES_PROPERTIES "" CACHE STRING "C++ code identifing the BusesProperties for the AudioProcessor")

set(PLUGIN_IS_SYNTH OFF CACHE BOOL "Is the plugin a Synth? Otherwise, it is an effect")
set(PLUGIN_NEEDS_MIDI_INPUT OFF CACHE BOOL "Does the plugin need midi input?")
set(PLUGIN_NEEDS_MIDI_OUTPUT OFF CACHE BOOL "Does the plugin need midi output?")
set(PLUGIN_IS_MIDI_EFFECT OFF CACHE BOOL "Is this plugin a MIDI effect?")
#set(PLUGIN_EDITOR_WANTS_KEYBOARD_FOCUS OFF CACHE BOOL "Does the editor need keyboard focus?")
set(PLUGIN_PARAM_DEFAULT_NOTIFY On CACHE BOOL "Should parameter changes from inside your rnbo patch send output by default? Sometimes breaks automation.")

set(CONAN_PROFILE "default" CACHE STRING "The profile to use for building conan deps, this is useful for cross compiling")

set(RNBO_CPP_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../../.." CACHE FILEPATH "The path to the the RNBO c++ source directory")
set(RNBO_CLASS_FILE "${CMAKE_CURRENT_SOURCE_DIR}/../../../../../codegen/xam/test/test.cpp" CACHE FILEPATH "The file that holds the generated RNBO class code")
set(RNBO_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/../../../../../codegen/xam/test/description.json" CACHE FILEPATH "path to description.json")
set(RNBO_PRESETS_FILE "${CMAKE_CURRENT_SOURCE_DIR}/../../../../../codegen/xam/test/presets.json" CACHE FILEPATH "path to presets.json")
set(RNBO_BINARY_DATA_FILE "${CMAKE_CURRENT_SOURCE_DIR}/../../../../../codegen/xam/test/rnbomatic_binary.cpp" CACHE FILEPATH "path to rnbomatic_binary.cpp if it exists")
set(RNBO_BINARY_DATA_STORAGE_NAME "rnbomatic_binary" CACHE STRING "name of the extern binary data")
set(RNBO_EXTRA_SOURCE_FILES "" CACHE STRING "A list of paths of any additional CPP files to compile into the project")

set(PLUGIN_NUM_MIDI_INPUTS 0 CACHE STRING "Number of MIDI input ports")
set(PLUGIN_NUM_MIDI_OUTPUTS 0 CACHE STRING "Number of MIDI output ports")

set(JUCE_VERSION 7.0.3)
set(JUCE_TAG c74/vst3emu)
set(JUCEAIDE_TAG c74/testing)

if (EXISTS ${RNBO_BINARY_DATA_FILE})
	add_definitions(-DRNBO_BINARY_DATA_STORAGE_NAME=${RNBO_BINARY_DATA_STORAGE_NAME})
endif()

#shouldn't cmake be doing this for us?
if (CMAKE_BUILD_TYPE STREQUAL "Release")
	add_definitions(-DNDEBUG)
	set(JUCE_DEBUG 0)
else()
	add_definitions(-DDEBUG)
	set(JUCE_DEBUG 1)
endif()

conan_cmake_configure(
	REQUIRES JUCEStaticPlugin/${JUCE_VERSION}@${JUCE_TAG}
	GENERATORS cmake_find_package
)
conan_cmake_install(
	PATH_OR_REFERENCE .
	BUILD missing
	PROFILE_HOST ${CONAN_PROFILE}
	PROFILE_BUILD default
	SETTINGS_HOST build_type=${CMAKE_BUILD_TYPE}
	SETTINGS_BUILD build_type=${CMAKE_BUILD_TYPE}
)
find_package(JUCEStaticPlugin ${JUCE_VERSION} REQUIRED)

#include juceaide so we can use it with the cmake files, specficially JUCEUtils.cmake
conan_cmake_configure(
	REQUIRES JUCEAide/${JUCE_VERSION}@${JUCEAIDE_TAG}
	GENERATORS cmake_paths
)
conan_cmake_install(
	PATH_OR_REFERENCE .
	BUILD missing
	PROFILE_HOST default
	PROFILE_BUILD default
	SETTINGS_HOST build_type=${CMAKE_BUILD_TYPE}
	SETTINGS_BUILD build_type=${CMAKE_BUILD_TYPE}
)
include(${CMAKE_CURRENT_BINARY_DIR}/conan_paths.cmake)
add_executable(juceaide IMPORTED GLOBAL)
set_target_properties(juceaide PROPERTIES IMPORTED_LOCATION "${CONAN_JUCEAIDE_ROOT}/bin/juceaide")
add_executable(juce::juceaide ALIAS juceaide)

set(JUCE_CMAKE_DIR ${JUCEStaticPlugin_LIB_DIRS}/../cmake/)

#include some cmake files so we can use juce_add_binary_data and _juce_add_unity_script_file
include(${JUCE_CMAKE_DIR}/JUCEModuleSupport.cmake)
include(${JUCE_CMAKE_DIR}/JUCEUtils.cmake)

#include description file header if description file exists
include(${RNBO_CPP_DIR}/cmake/RNBODescriptionHeader.cmake)
set(DESCRIPTION_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/include)
rnbo_write_description_header_if_exists(${RNBO_DESCRIPTION_FILE} ${DESCRIPTION_INCLUDE_DIR} ${RNBO_PRESETS_FILE})

if (LINUX)
	conan_cmake_configure(
		REQUIRES libalsa/1.2.10
		GENERATORS cmake_find_package
		OPTIONS libalsa:shared=True
	)
	conan_cmake_install(
		PATH_OR_REFERENCE .
		BUILD missing
		PROFILE_HOST ${CONAN_PROFILE}
		PROFILE_BUILD default
		SETTINGS_HOST build_type=${CMAKE_BUILD_TYPE}
		SETTINGS_BUILD build_type=${CMAKE_BUILD_TYPE}
	)
	find_package(ALSA 1.2.10 REQUIRED)

	conan_cmake_configure(
		REQUIRES freetype/2.13.2
		GENERATORS cmake_find_package
		OPTIONS freetype:shared=True
	)
	conan_cmake_install(
		PATH_OR_REFERENCE .
		BUILD missing
		PROFILE_HOST ${CONAN_PROFILE}
		PROFILE_BUILD default
		SETTINGS_HOST build_type=${CMAKE_BUILD_TYPE}
		SETTINGS_BUILD build_type=${CMAKE_BUILD_TYPE}
	)
	find_package(Freetype 2.13.2 REQUIRED)
endif()

#compute the PLUGIN_TYPE
if (PLUGIN_IS_SYNTH)
	set(PLUGIN_TYPE "AudioSynth")
elseif(PLUGIN_IS_MIDI_EFFECT)
	set(PLUGIN_TYPE "MIDIEffect")
else()
	set(PLUGIN_TYPE "AudioEffect")
endif()

if (PLUGIN_NEEDS_MIDI_INPUT AND PLUGIN_NEEDS_MIDI_OUTPUT)
	set(PLUGIN_TYPE "${PLUGIN_TYPE}MIDIInOut")
elseif(PLUGIN_NEEDS_MIDI_INPUT)
	set(PLUGIN_TYPE "${PLUGIN_TYPE}MIDIIn")
elseif(PLUGIN_NEEDS_MIDI_OUTPUT)
	set(PLUGIN_TYPE "${PLUGIN_TYPE}MIDIOut")
else()
	set(PLUGIN_TYPE "${PLUGIN_TYPE}NoMIDI")
endif()

#AU_PLUGIN_TYPE
if (PLUGIN_IS_MIDI_EFFECT)
	set(AU_PLUGIN_TYPE "aumi") # kAudioUnitType_MIDIProcessor
elseif (PLUGIN_IS_SYNTH)
	set(AU_PLUGIN_TYPE "aumu") # kAudioUnitType_MusicDevice
elseif (PLUGIN_NEEDS_MIDI_INPUT)
	set(AU_PLUGIN_TYPE "aumf") # kAudioUnitType_MusicEffect
else()
	set(AU_PLUGIN_TYPE "aufx") # kAudioUnitType_Effect
endif()

#compute bundle id
STRING(REGEX REPLACE "[^A-Za-z0-9]" "" PRODUCT_NAME_ID ${PLUGIN_NAME})
STRING(REGEX REPLACE "[^A-Za-z0-9]" "" COMPANY_NAME_ID ${PLUGIN_MANUFACTURER_NAME})
SET(MACOSX_BUNDLE_GUI_IDENTIFIER "com.${COMPANY_NAME_ID}.${PRODUCT_NAME_ID}")

#compute numeric codes
STRING(HEX ${PLUGIN_CODE} PLUGIN_CODE_HEX)
STRING(HEX ${PLUGIN_MANUFACTURER_CODE} PLUGIN_MANUFACTURER_CODE_HEX)

set(JUCE_IS_MIDI_EFFECT 0)
if (PLUGIN_IS_MIDI_EFFECT)
	set(JUCE_IS_MIDI_EFFECT 1)
endif()

set(RNBO_JUCE_PARAM_DEFAULT_NOTIFY 0)
if (PLUGIN_PARAM_DEFAULT_NOTIFY)
	set(RNBO_JUCE_PARAM_DEFAULT_NOTIFY 1)
endif()


link_directories(${JUCEStaticPlugin_LIB_DIRS} ${ALSA_LIB_DIRS} ${Freetype_LIB_DIRS})

function(setup_common target_name target_type suffix bundle_extension)
	add_library(${target_name} MODULE)
	set_target_properties(${target_name} PROPERTIES
			BUNDLE_EXTENSION ${bundle_extension}
			XCODE_ATTRIBUTE_WRAPPER_EXTENSION ${bundle_extension}
			BUNDLE TRUE
			XCODE_ATTRIBUTE_LIBRARY_STYLE Bundle
			XCODE_ATTRIBUTE_GENERATE_PKGINFO_FILE YES
			#plist
			MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_LIST_DIR}/${target_type}Info.plist.in
			MACOSX_BUNDLE_BUNDLE_NAME ${PLUGIN_NAME}
			MACOSX_BUNDLE_GUI_IDENTIFIER ${MACOSX_BUNDLE_GUI_IDENTIFIER}
			MACOSX_BUNDLE_COPYRIGHT ${PLUGIN_MANUFACTURER_NAME}
			MACOSX_BUNDLE_LONG_VERSION_STRING ${PLUGIN_VERSION}
			MACOSX_BUNDLE_SHORT_VERSION_STRING ${PLUGIN_VERSION}
			#custom
			PLUGIN_NAME ${PLUGIN_NAME}
			PLUGIN_MANUFACTURER_NAME ${PLUGIN_MANUFACTURER_NAME}
			PLUGIN_MANUFACTURER_CODE ${PLUGIN_MANUFACTURER_CODE}
			PLUGIN_CODE ${PLUGIN_CODE}
			AU_PLUGIN_TYPE ${AU_PLUGIN_TYPE}
	)

	target_compile_definitions(${target_name}
		PUBLIC
		PLUGIN_NAME="${PLUGIN_NAME}"
		PLUGIN_MANUFACTURER_NAME="${PLUGIN_MANUFACTURER_NAME}"
		PLUGIN_MANUFACTURER_WEBSITE="${PLUGIN_MANUFACTURER_WEBSITE}"
		PLUGIN_MANUFACTURER_EMAIL="${PLUGIN_MANUFACTURER_EMAIL}"
		PLUGIN_MANUFACTURER_CODE="${PLUGIN_MANUFACTURER_CODE}"
		PLUGIN_MANUFACTURER_CODE_HEX=0x${PLUGIN_MANUFACTURER_CODE_HEX}
		PLUGIN_CODE="${PLUGIN_CODE}"
		PLUGIN_CODE_HEX=0x${PLUGIN_CODE_HEX}
		PLUGIN_VERSION="${PLUGIN_VERSION}"
		PLUGIN_VERSION_NUM=${PLUGIN_VERSION_NUM}
		PLUGIN_NUM_MIDI_INPUTS=${PLUGIN_NUM_MIDI_INPUTS}
		PLUGIN_NUM_MIDI_OUTPUTS=${PLUGIN_NUM_MIDI_OUTPUTS}
		JUCE_GLOBAL_MODULE_SETTINGS_INCLUDED
		JUCE_STATIC_PLUGIN=1
		JUCE_DEBUG=${JUCE_DEBUG}
		JucePlugin_IsMidiEffect=${JUCE_IS_MIDI_EFFECT}
		RNBO_JUCE_PARAM_DEFAULT_NOTIFY=${RNBO_JUCE_PARAM_DEFAULT_NOTIFY}
	)

	if (PLUGIN_BUSES_PROPERTIES)
		target_compile_definitions(${target_name}
			PUBLIC
			PLUGIN_BUSES_PROPERTIES=${PLUGIN_BUSES_PROPERTIES}
		)
	endif()

	target_include_directories(${target_name}
		PRIVATE
		${RNBO_CPP_DIR}/
		${RNBO_CPP_DIR}/common/
		${RNBO_CPP_DIR}/src/
		${RNBO_CPP_DIR}/adapters/juce/
		${RNBO_CPP_DIR}/src/3rdparty/
		${JUCEStaticPlugin_INCLUDE_DIRS}
		${DESCRIPTION_INCLUDE_DIR}
		${ALSA_INCLUDE_DIRS}
		${Freetype_INCLUDE_DIRS}
	)
	target_link_libraries(${target_name}
		PRIVATE
		${PLUGIN_TYPE}_SharedCode
		${PLUGIN_TYPE}.${suffix}
		${JUCEStaticPlugin_LIBRARIES}
	)
	if (APPLE)
		target_link_libraries(${target_name}
			PRIVATE
			"-weak_framework Metal"
			"-weak_framework MetalKit"
		)
	endif()

	if (BUILD_SYSTEM_IS_MINGW)
		target_link_libraries(${target_name}
			PRIVATE
			mingw_stdthreads
			-static-libgcc
			-static-libstdc++
		)
	endif()
	set_target_properties(${target_name} PROPERTIES
		OUTPUT_NAME ${PLUGIN_NAME}
		XCODE_ATTRIBUTE_PRODUCT_NAME ${PLUGIN_NAME}
		)

	if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
		set_target_properties(${target_name} PROPERTIES PREFIX "")
	endif()

	if (target_type STREQUAL VST3 OR target_type STREQUAL AU OR APPLE)
		set(output_path "${CMAKE_CURRENT_BINARY_DIR}/${PLUGIN_NAME}.${bundle_extension}")
		if((CMAKE_SYSTEM_NAME STREQUAL "Linux") OR (CMAKE_SYSTEM_NAME MATCHES ".*BSD"))
			set_target_properties(${target_name} PROPERTIES
				SUFFIX .so
				PREFIX ""
				LIBRARY_OUTPUT_DIRECTORY "${output_path}/Contents/${CMAKE_SYSTEM_PROCESSOR}-linux")
		elseif (CMAKE_SYSTEM_NAME STREQUAL "Windows" AND suffix STREQUAL "vst3")
			set_target_properties(${target_name} PROPERTIES
				PREFIX ""
				SUFFIX .vst3
				LIBRARY_OUTPUT_DIRECTORY "${output_path}/Contents/${CMAKE_SYSTEM_PROCESSOR}-win")
		elseif (APPLE AND NOT CMAKE_GENERATOR STREQUAL "Xcode")
			#write PkgInfo
			add_custom_command(
				TARGET ${target_name} POST_BUILD
				COMMAND ${CMAKE_COMMAND} -E copy
				${CMAKE_CURRENT_SOURCE_DIR}/PkgInfo
				"${CMAKE_CURRENT_BINARY_DIR}/${PLUGIN_NAME}.${bundle_extension}/Contents/PkgInfo")
		endif()
		#bundles are directories
		#install destinations specifically setup for cloud compiler
		install(DIRECTORY ${output_path}
			DESTINATION .
		)
	else()
		install(TARGETS ${target_name}
			DESTINATION .
		)
	endif()
endfunction()

set(SRC_FILES
		"${CMAKE_CURRENT_LIST_DIR}/common.cpp"
		${RNBO_CLASS_FILE}
		${RNBO_CPP_DIR}/RNBO.cpp
		"${CMAKE_CURRENT_LIST_DIR}/../RNBO_JuceAudioProcessor.cpp"
		"${CMAKE_CURRENT_LIST_DIR}/../RNBO_JuceAudioProcessorEditor.cpp"
)

foreach(extra IN LISTS RNBO_EXTRA_SOURCE_FILES)
	#files are in a string, update to mimic behavior of FILEPATH cache var
	file(REAL_PATH ${extra} extra BASE_DIRECTORY ${CMAKE_BINARY_DIR})
	list(APPEND SRC_FILES ${extra})
endforeach()

if (EXISTS ${RNBO_BINARY_DATA_FILE})
	list(APPEND SRC_FILES ${RNBO_BINARY_DATA_FILE})
endif()

#VST3
if (
	(APPLE AND PLUGIN_VST3_MAC) OR
	(CMAKE_SYSTEM_NAME STREQUAL "Windows" AND PLUGIN_VST3_WIN) OR
	(CMAKE_SYSTEM_NAME STREQUAL "Linux" AND PLUGIN_VST3_LINUX)
)
	setup_common(RNBOVST3Plugin VST3 vst3 vst3)
	target_sources(RNBOVST3Plugin
		PRIVATE
		"${CMAKE_CURRENT_LIST_DIR}/vst3.cpp"
		${SRC_FILES}
	)
endif()


#Unity
if (
	(APPLE AND PLUGIN_UNITY_MAC) OR
	(CMAKE_SYSTEM_NAME STREQUAL "Windows" AND PLUGIN_UNITY_WIN) OR
	(CMAKE_SYSTEM_NAME STREQUAL "Linux" AND PLUGIN_UNITY_LINUX)
)
	setup_common(RNBOUnityPlugin Unity unity bundle)

	#configure script
	#copied/adapted from JUCE
	set(script_in "${JUCE_CMAKE_DIR}/UnityPluginGUIScript.cs.in")
	set(plugin_name "${PLUGIN_NAME}")
	set(plugin_vendor "${PLUGIN_MANUFACTURER_NAME}")
	set(plugin_description "${PLUGIN_NAME}")

	string(REGEX REPLACE " +" "_" plugin_class_name "${plugin_name}")

	_juce_add_unity_plugin_prefix_if_necessary("${plugin_name}" script_prefix)
	set(script_file "${CMAKE_BINARY_DIR}/${script_prefix}_UnityScript.cs")

	configure_file(${script_in} ${script_file})

	target_sources(RNBOUnityPlugin
		PRIVATE
		${script_file}
		"${CMAKE_CURRENT_LIST_DIR}/unity.cpp"
		${SRC_FILES}
	)

	if (APPLE)
		set_source_files_properties("${script_file}" PROPERTIES
			GENERATED TRUE
			MACOSX_PACKAGE_LOCATION Resources
		)
	else()
		install(FILES ${script_file}
			DESTINATION ${CMAKE_BINARY_DIR}/install/unity/
		)
	endif()
endif()


#AU
if(APPLE AND PLUGIN_AU_MAC)
	setup_common(RNBOAUPlugin AU component component)
	target_sources(RNBOAUPlugin
		PRIVATE
		"${CMAKE_CURRENT_LIST_DIR}/au.cpp"
		${SRC_FILES}
	)
endif()
