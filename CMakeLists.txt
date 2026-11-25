cmake_minimum_required(VERSION 3.22)

# Static MSVC runtime: the EXE then runs on a clean Windows machine without the VC++ redistributable.
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "" FORCE)

project(PinksterKeyzSampleMachine VERSION 0.1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

if(WIN32)
    set(_psm_app_default ON)
else()
    set(_psm_app_default OFF)   # Linux Codespaces builds + tests the portable core by default
endif()

option(PSM_BUILD_APP   "Build the JUCE desktop application (downloads JUCE)" ${_psm_app_default})
option(PSM_BUILD_TESTS "Build unit tests and the SF2 demo tool" ON)
option(PSM_ENABLE_ASIO "Enable ASIO (requires the Steinberg ASIO SDK, which cannot be redistributed)" OFF)
set(PSM_ASIO_SDK_DIR "" CACHE PATH "Folder containing the ASIO SDK 'common' directory")
set(PSM_JUCE_TAG "8.0.4" CACHE STRING "Pinned JUCE release tag")

# ---------------------------------------------------------------- portable core (no JUCE)
add_library(psm_core STATIC
    Source/SoundFont/Sf2Writer.cpp
    Source/LoopEngine/LoopMath.cpp
    Source/AutoFill/AutoFill.cpp
    Source/Sampler/SamplingPlan.cpp)
target_include_directories(psm_core PUBLIC Source)
target_compile_features(psm_core PUBLIC cxx_std_20)
if(MSVC)
    target_compile_options(psm_core PRIVATE /W4 /permissive-)
else()
    target_compile_options(psm_core PRIVATE -Wall -Wextra)
endif()

# ---------------------------------------------------------------- tests
if(PSM_BUILD_TESTS)
    enable_testing()
    add_executable(psm_tests
        Tests/TestMain.cpp Tests/Sf2Tests.cpp Tests/MappingTests.cpp Tests/LoopSamplerTests.cpp)
    target_link_libraries(psm_tests PRIVATE psm_core)
    add_test(NAME psm_unit_tests COMMAND psm_tests)

    add_executable(psm_sf2_demo Tests/Sf2Demo.cpp)
    target_link_libraries(psm_sf2_demo PRIVATE psm_core)
    add_test(NAME psm_sf2_demo_export COMMAND psm_sf2_demo ${CMAKE_BINARY_DIR}/demo.sf2)
endif()

# ---------------------------------------------------------------- JUCE application
if(PSM_BUILD_APP)
    include(FetchContent)
    FetchContent_Declare(JUCE
        GIT_REPOSITORY https://github.com/juce-framework/JUCE.git
        GIT_TAG        ${PSM_JUCE_TAG}
        GIT_SHALLOW    ON)
    FetchContent_MakeAvailable(JUCE)

    juce_add_gui_app(PinksterKeyzSampleMachine
        PRODUCT_NAME "PinksterKeyzSampleMachine"
        COMPANY_NAME "PINKSTERKEYZ"
        VERSION      ${PROJECT_VERSION})

    target_sources(PinksterKeyzSampleMachine PRIVATE
        Source/Main.cpp
        Source/MainComponent.cpp
        Source/Audio/AudioEngine.cpp
        Source/MIDI/MidiRouter.cpp
        Source/Recording/RecordingEngine.cpp)

    target_compile_definitions(PinksterKeyzSampleMachine PRIVATE
        JUCE_WEB_BROWSER=0
        JUCE_USE_CURL=0
        JUCE_PLUGINHOST_VST3=1          # VST3 hosting (used from stage 3)
        JUCE_USE_WIN_WASAPI=1
        JUCE_DIRECTSOUND=1
        JUCE_STRICT_REFCOUNTEDPOINTER=1)

    if(PSM_ENABLE_ASIO)
        if(NOT EXISTS "${PSM_ASIO_SDK_DIR}/common/asio.h")
            message(FATAL_ERROR "PSM_ENABLE_ASIO=ON but PSM_ASIO_SDK_DIR/common/asio.h was not found")
        endif()
        target_compile_definitions(PinksterKeyzSampleMachine PRIVATE JUCE_ASIO=1)
        target_include_directories(PinksterKeyzSampleMachine PRIVATE "${PSM_ASIO_SDK_DIR}/common")
    endif()

    target_link_libraries(PinksterKeyzSampleMachine
        PRIVATE
            psm_core
            juce::juce_audio_utils
            juce::juce_audio_processors
            juce::juce_audio_formats
            juce::juce_dsp
        PUBLIC
            juce::juce_recommended_config_flags
            juce::juce_recommended_lto_flags
            juce::juce_recommended_warning_flags)
endif()
