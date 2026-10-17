cmake_minimum_required(VERSION 3.22.1)

project(Winlator)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2 -Wno-unused-function -Wimplicit-function-declaration")

add_subdirectory(OpenXR-SDK)
add_subdirectory(patchelf)
add_subdirectory(adrenotools)

# Add Winlator shared library
add_library(winlator SHARED
        xr/engine.c
        xr/framebuffer.c
        xr/input.c
        xr/main.c
        xr/math.c
        xr/renderer.c
        winlator/drawable.c
        winlator/gpu_image.c
        winlator/process_helper.c
        winlator/sysvshared_memory.c
        winlator/xconnector_epoll.c
        winlator/alsa_client.c
        winlator/vulkan.c
        winlator/renderer/egl.cpp
        winlator/renderer/renderer_jni.cpp
        winlator/renderer/shader.cpp
        winlator/renderer/window.cpp
        winlator/renderer/cursor.cpp
        winlator/renderer/xform.cpp
        winlator/renderer/view_transformation.cpp
        winlator/renderer/displayx.cpp
        winlator/renderer/effectcomposer/effect_composer.cpp
)

target_compile_options(winlator PRIVATE -Wall -Wextra)
target_include_directories(winlator PRIVATE winlator/renderer winlator/renderer/effectcomposer)
# DisplayX resolves the API 29+ ASurfaceTransaction calls with dlsym; NDKs before r29 still mark
# even their callback typedefs unavailable below minSdk.
if(ANDROID_NDK_MAJOR AND ANDROID_NDK_MAJOR LESS 29)
    set_source_files_properties(winlator/renderer/displayx.cpp PROPERTIES
            COMPILE_DEFINITIONS __ANDROID_UNAVAILABLE_SYMBOLS_ARE_WEAK__)
endif()

target_link_libraries(winlator
        log
        android
        jnigraphics
        openxr_loader
        aaudio
        EGL
        GLESv2
        GLESv3
        adrenotools
        vulkan
)

# Add Fake Input library for controller emulation (loaded via LD_PRELOAD)
add_library(fakeinput SHARED
        winlator/fakeinput.cpp
)

target_compile_options(fakeinput PRIVATE -Wall -Wextra -fvisibility=hidden)

target_link_libraries(fakeinput
        log
        dl
)

# Keep the Pypetto DisplayX/EGL target above untouched.  The optional Vulkan
# renderer is a separate JNI library so selecting it cannot change the native
# presentation path used by DisplayX or EGL.
set(VULKAN_RENDERER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/winlator/renderer/vulkan)

# glslc ships with the NDK; glslangValidator is used instead when it is on PATH.
if(CMAKE_HOST_WIN32)
    set(GLSLC_HOST_TAG "windows-x86_64")
    set(GLSLC_EXE_NAME "glslc.exe")
elseif(CMAKE_HOST_APPLE)
    set(GLSLC_HOST_TAG "darwin-x86_64")
    set(GLSLC_EXE_NAME "glslc")
else()
    set(GLSLC_HOST_TAG "linux-x86_64")
    set(GLSLC_EXE_NAME "glslc")
endif()
set(GLSLC "${ANDROID_NDK}/shader-tools/${GLSLC_HOST_TAG}/${GLSLC_EXE_NAME}")
if(NOT EXISTS "${GLSLC}")
    message(FATAL_ERROR "glslc not found at ${GLSLC}; check the NDK installation")
endif()
find_program(GLSLANG_VALIDATOR glslangValidator)
set(SPIRV_BIN2C_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/wfg/bin2c.cmake)

function(window_shader_header source var)
    set(output ${CMAKE_CURRENT_BINARY_DIR}/${var}.h)
    if(GLSLANG_VALIDATOR)
        add_custom_command(
            OUTPUT ${output}
            COMMAND ${GLSLANG_VALIDATOR} -V --vn ${var}_code -x -o ${output}
                    ${VULKAN_RENDERER_DIR}/${source}
            DEPENDS ${VULKAN_RENDERER_DIR}/${source}
            COMMENT "Compiling ${source} -> ${var}.h"
        )
    else()
        add_custom_command(
            OUTPUT ${output}
            COMMAND ${GLSLC} -o ${output}.spv ${VULKAN_RENDERER_DIR}/${source}
            COMMAND ${CMAKE_COMMAND} -DINPUT_FILE=${output}.spv -DOUTPUT_FILE=${output}
                    -DVAR_NAME=${var}_code -P ${SPIRV_BIN2C_SCRIPT}
            DEPENDS ${VULKAN_RENDERER_DIR}/${source} ${SPIRV_BIN2C_SCRIPT}
            COMMENT "Compiling ${source} -> ${var}.h"
            VERBATIM
        )
    endif()
endfunction()

window_shader_header(window.vert         window_vert)
window_shader_header(window.frag         window_frag)
window_shader_header(window_sgsr.frag    window_sgsr_frag)
window_shader_header(window_fsr1.frag    window_fsr1_frag)
window_shader_header(window_lanczos.frag window_lanczos_frag)
window_shader_header(window_stretch.frag window_stretch_frag)
window_shader_header(window_postfx.frag  window_postfx_frag)

add_subdirectory(wfg)

set(VULKAN_CONTEXT_SOURCE ${VULKAN_RENDERER_DIR}/VulkanRendererContext.cpp)
set(VULKAN_CONTEXT_LEGACY ${CMAKE_CURRENT_BINARY_DIR}/VulkanRendererContext_legacy.cpp)
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS ${VULKAN_CONTEXT_SOURCE})
file(READ ${VULKAN_CONTEXT_SOURCE} VULKAN_CONTEXT_TEXT)

set(PIPELINE_OPT_INCLUDE_MARKER "#include <dlfcn.h>\n")
string(FIND "${VULKAN_CONTEXT_TEXT}" "${PIPELINE_OPT_INCLUDE_MARKER}" PIPELINE_OPT_INCLUDE_POS)
if(PIPELINE_OPT_INCLUDE_POS EQUAL -1)
    message(FATAL_ERROR "VulkanRendererContext.cpp changed: pipeline-opt include marker not found")
endif()
string(REPLACE
    "${PIPELINE_OPT_INCLUDE_MARKER}"
    "#include <dlfcn.h>\n#include \"VulkanRendererPipelineOpt.h\"\n"
    VULKAN_CONTEXT_TEXT
    "${VULKAN_CONTEXT_TEXT}")

set(PIPELINE_OPT_DESTROY_MARKER "    vk_.DeviceWaitIdle(device);\n    for (auto& [id, wt] : texMap) destroyWinTex(wt);")
string(FIND "${VULKAN_CONTEXT_TEXT}" "${PIPELINE_OPT_DESTROY_MARKER}" PIPELINE_OPT_DESTROY_POS)
if(PIPELINE_OPT_DESTROY_POS EQUAL -1)
    message(FATAL_ERROR "VulkanRendererContext.cpp changed: pipeline-opt destroy marker not found")
endif()
string(REPLACE
    "${PIPELINE_OPT_DESTROY_MARKER}"
    "    vk_.DeviceWaitIdle(device);\n    VulkanRendererDestroyOpaquePipeline(this);\n    for (auto& [id, wt] : texMap) destroyWinTex(wt);"
    VULKAN_CONTEXT_TEXT
    "${VULKAN_CONTEXT_TEXT}")

set(LANCZOS_DESTROY_MARKER "    if (fsr1Pipeline != VK_NULL_HANDLE) vk_.DestroyPipeline(device, fsr1Pipeline, nullptr);\n")
string(FIND "${VULKAN_CONTEXT_TEXT}" "${LANCZOS_DESTROY_MARKER}" LANCZOS_DESTROY_POS)
if(LANCZOS_DESTROY_POS EQUAL -1)
    message(FATAL_ERROR "VulkanRendererContext.cpp changed: Lanczos destroy marker not found")
endif()
string(REPLACE
    "${LANCZOS_DESTROY_MARKER}"
    "    if (fsr1Pipeline != VK_NULL_HANDLE) vk_.DestroyPipeline(device, fsr1Pipeline, nullptr);\n    if (lanczosPipeline != VK_NULL_HANDLE) vk_.DestroyPipeline(device, lanczosPipeline, nullptr);\n"
    VULKAN_CONTEXT_TEXT
    "${VULKAN_CONTEXT_TEXT}")

set(RECORD_CMD_MARKER "void VulkanRendererContext::recordCmdBuf(VkCommandBuffer cb, uint32_t imgIdx,")
set(RENDER_LOOP_MARKER "\nvoid VulkanRendererContext::renderLoop() {")
string(FIND "${VULKAN_CONTEXT_TEXT}" "${RECORD_CMD_MARKER}" RECORD_CMD_POS)
string(FIND "${VULKAN_CONTEXT_TEXT}" "${RENDER_LOOP_MARKER}" RENDER_LOOP_POS)
if(RECORD_CMD_POS EQUAL -1 OR RENDER_LOOP_POS EQUAL -1)
    message(FATAL_ERROR "VulkanRendererContext.cpp changed: unable to isolate legacy recordCmdBuf")
endif()
string(REPLACE
    "${RECORD_CMD_MARKER}"
    "#if 0\n${RECORD_CMD_MARKER}"
    VULKAN_CONTEXT_TEXT
    "${VULKAN_CONTEXT_TEXT}")
string(REPLACE
    "${RENDER_LOOP_MARKER}"
    "\n#endif\n${RENDER_LOOP_MARKER}"
    VULKAN_CONTEXT_TEXT
    "${VULKAN_CONTEXT_TEXT}")

set(FILTER_CREATE_MARKER "    if (mode == 3 && fsr1Pipeline == VK_NULL_HANDLE) createFsr1Pipeline();\n")
string(FIND "${VULKAN_CONTEXT_TEXT}" "${FILTER_CREATE_MARKER}" FILTER_CREATE_POS)
if(FILTER_CREATE_POS EQUAL -1)
    message(FATAL_ERROR "VulkanRendererContext.cpp changed: filter creation marker not found")
endif()
string(REPLACE
    "${FILTER_CREATE_MARKER}"
    "    if (mode == 3 && fsr1Pipeline == VK_NULL_HANDLE) createFsr1Pipeline();\n    if (mode == 4 && lanczosPipeline == VK_NULL_HANDLE) createLanczosPipeline();\n"
    VULKAN_CONTEXT_TEXT
    "${VULKAN_CONTEXT_TEXT}")

string(REPLACE
    "auto modeName=[](int m){ return m==1?\"NEAREST\":m==2?\"SGSR\":m==3?\"FSR1\":\"LINEAR\"; };"
    "auto modeName=[](int m){ return m==1?\"NEAREST\":m==2?\"SGSR\":m==3?\"FSR\":m==4?\"LANCZOS2\":m==5?\"COLOR_BOOST\":\"LINEAR\"; };"
    VULKAN_CONTEXT_TEXT
    "${VULKAN_CONTEXT_TEXT}")
string(REPLACE
    "filterMode, filterMode==3?\"FSR1\":filterMode==2?\"SGSR\":filterMode==1?\"NEAREST\":\"LINEAR\");"
    "filterMode, filterMode==5?\"COLOR_BOOST\":filterMode==4?\"LANCZOS2\":filterMode==3?\"FSR\":filterMode==2?\"SGSR\":filterMode==1?\"NEAREST\":\"LINEAR\");"
    VULKAN_CONTEXT_TEXT
    "${VULKAN_CONTEXT_TEXT}")

file(WRITE ${VULKAN_CONTEXT_LEGACY} "${VULKAN_CONTEXT_TEXT}")

include_directories(
        ${CMAKE_CURRENT_BINARY_DIR}
        ${VULKAN_RENDERER_DIR}
)

add_library(vulkan_renderer SHARED
        winlator/renderer/vulkan/vulkan_jni.cpp
        ${VULKAN_CONTEXT_LEGACY}
        winlator/renderer/vulkan/VulkanRendererFilters.cpp
        winlator/renderer/vulkan/VulkanRendererPipelineOpt.cpp
        winlator/renderer/vulkan/VulkanRendererWfg.cpp

        winlator/renderer/vulkan/view_transformation.cpp
        winlator/renderer/vulkan/xform.cpp

        ${CMAKE_CURRENT_BINARY_DIR}/window_vert.h
        ${CMAKE_CURRENT_BINARY_DIR}/window_frag.h
        ${CMAKE_CURRENT_BINARY_DIR}/window_sgsr_frag.h
        ${CMAKE_CURRENT_BINARY_DIR}/window_fsr1_frag.h
        ${CMAKE_CURRENT_BINARY_DIR}/window_lanczos_frag.h
        ${CMAKE_CURRENT_BINARY_DIR}/window_stretch_frag.h
        ${CMAKE_CURRENT_BINARY_DIR}/window_postfx_frag.h
)

set_target_properties(vulkan_renderer PROPERTIES
        CXX_STANDARD 17
        CXX_STANDARD_REQUIRED ON
        CXX_EXTENSIONS OFF
)

target_compile_options(vulkan_renderer PRIVATE
        -Wall -Wextra -fvisibility=hidden
)

target_link_libraries(vulkan_renderer
        log
        android
        vulkan
        adrenotools
        dl
        atomic
        wfg
)
