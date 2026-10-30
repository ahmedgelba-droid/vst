cmake_minimum_required(VERSION 3.22)

# Static MSVC runtime => no "VCRUNTIME140.dll missing" problems on other PCs / FL Studio.
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

project(MizmarSampler VERSION 0.1.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ---------------------------------------------------------------------------
# JUCE: use a local checkout if you pass -DJUCE_PATH=C:/dev/JUCE,
# otherwise download it automatically.
# ---------------------------------------------------------------------------
if(DEFINED JUCE_PATH)
    add_subdirectory(${JUCE_PATH} JUCE)
else()
    include(FetchContent)
    FetchContent_Declare(JUCE
        GIT_REPOSITORY https://github.com/juce-framework/JUCE.git
        GIT_TAG        8.0.4
        GIT_SHALLOW    ON)
    FetchContent_MakeAvailable(JUCE)
endif()

# ---------------------------------------------------------------------------
# Plugin target
# ---------------------------------------------------------------------------
juce_add_plugin(MizmarSampler
    COMPANY_NAME                "YourName"
    PLUGIN_MANUFACTURER_CODE    Mzmr
    PLUGIN_CODE                 Mzsm
    FORMATS                     VST3 Standalone
    PRODUCT_NAME                "Mizmar Sampler"
    IS_SYNTH                    TRUE
    NEEDS_MIDI_INPUT            TRUE
    NEEDS_MIDI_OUTPUT           FALSE
    IS_MIDI_EFFECT              FALSE
    EDITOR_WANTS_KEYBOARD_FOCUS FALSE
    # Copying into C:/Program Files/Common Files/VST3 needs admin rights.
    # Leave FALSE and copy by hand, or run your IDE as administrator and set TRUE.
    COPY_PLUGIN_AFTER_BUILD     FALSE
    VST3_CATEGORIES             Instrument Synth)

target_sources(MizmarSampler PRIVATE
    Source/PluginProcessor.cpp
    Source/PluginEditor.cpp
    Source/SamplerEngine.cpp
    Source/MicrotonalManager.cpp
    Source/PresetManager.cpp)

target_compile_definitions(MizmarSampler PUBLIC
    JUCE_WEB_BROWSER=0
    JUCE_USE_CURL=0
    JUCE_VST3_CAN_REPLACE_VST2=0)

target_link_libraries(MizmarSampler
    PRIVATE
        juce::juce_audio_utils      # pulls in audio_processors, audio_formats, gui_basics...
    PUBLIC
        juce::juce_recommended_config_flags
        juce::juce_recommended_lto_flags
        juce::juce_recommended_warning_flags)
