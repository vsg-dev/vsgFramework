cmake_minimum_required(VERSION 3.14)

project(vsgFramework)

include(FetchContent)
set(FETCHCONTENT_QUIET FALSE)

set(DOWNLOADS_DIR ${PROJECT_BINARY_DIR}/downloads)
set(COMPONENTS_DIR ${PROJECT_BINARY_DIR}/components)
set(FETCHCONTENT_BASE_DIR ${COMPONENTS_DIR})

# set compilation to use C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
# Change the default build type to Release
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif(NOT CMAKE_BUILD_TYPE)

#set(CMAKE_PREFIX_PATH ${PROJECT_BINARY_DIR})


find_package(Threads)

find_library(DL_LIBRARY dl)
if(NOT DL_LIBRARY)
    set(DL_LIBRARY "") # change from NOTFOUND to empty when passed to linker
endif()

add_custom_target(clobber
     COMMAND rm -rf ${COMPONENTS_DIR} CMakeCache.txt CMakeFiles cmake_install.cmake Makefile install_manifest.txt bin include lib SPIRV* CMakeDoxyfile.in CMakeDoxygenDefaults.cmake compile_commands.json
)

##############################################################################################################
#
# VulkkanSDK
#
if (VULKAN_SDK)
    set(ENV{VULKAN_SDK} ${VULKAN_SDK})
endif()

find_package(Vulkan)

if (NOT VULKAN_FOUND)
    set(VulkanSDKVersion "1.2.176.1" CACHE STRING "Vulkan_SDK version to download")

    if (NOT EXISTS ${DOWNLOADS_DIR}/${VulkanSDKVersion})

        file(MAKE_DIRECTORY downloads)

        if (ANDROID)
            # TODO : Need to implement Andoid VulkanSDK download and unpack
        elseif (WIN32)
            # TODO : Need to implement Windows VulkanSDK download and unpack
        elseif (APPLE)
            # TODO : Need to implement MacOS VulkanSDK download and unpack
            # Possible something like:
            # execute_process(COMMAND wget --no-cookies -O ../vulkansdk-macos-${{env.VulkanSDKVersion}}.dmg https://sdk.lunarg.com/sdk/download/${{env.VulkanSDKVersion}}/mac/vulkansdk-macos-{{env.VulkanSDKVersion}}.dmg)
            # then mount the dmg in some fashion
        else()
            # linux
            execute_process(COMMAND wget --no-cookies -O ${DOWNLOADS_DIR}/vulkansdk-linux-x86_64-${VulkanSDKVersion}.tar.gz https://sdk.lunarg.com/sdk/download/${VulkanSDKVersion}/linux/vulkansdk-linux-x86_64-${VulkanSDKVersion}.tar.gz)
            execute_process(COMMAND tar -zxf ${DOWNLOADS_DIR}/vulkansdk-linux-x86_64-${VulkanSDKVersion}.tar.gz -C ${DOWNLOADS_DIR})
        endif()

    endif()

    if (EXISTS ${DOWNLOADS_DIR}/${VulkanSDKVersion})

        set(VULKAN_SDK ${DOWNLOADS_DIR}/${VulkanSDKVersion}/x86_64)
        set(ENV{VULKAN_SDK} ${VULKAN_SDK})
        set(EXTRA_CMAKE_ARGS_VULKAN_SDK -DVULKAN_SDK=${VULKAN_SDK})

        find_package(Vulkan)

    endif()
endif()

##############################################################################################################
#
# SPIRV-Headers
#
OPTION(BUILD_SPIRV_headers "Build SPIRV headers" OFF)
if (${BUILD_SPIRV_headers})
    FetchContent_Declare(SPIRV_headers
        GIT_REPOSITORY "https://github.com/KhronosGroup/SPIRV-Headers.git"
        GIT_TAG "master"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(SPIRV_headers)
endif()

##############################################################################################################
#
# SPIRV-Tools
#
OPTION(BUILD_SPIRV_tools "Build SPIRV tools" OFF)
if (${BUILD_SPIRV_tools})
    FetchContent_Declare(SPIRV_tools
        GIT_REPOSITORY "https://github.com/KhronosGroup/SPIRV-Tools.git"
        GIT_TAG "master"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(SPIRV_tools)
endif()

##############################################################################################################
#
# glslang
#
OPTION(BUILD_glslang "Build glslang" ON)
if (${BUILD_glslang})
    FetchContent_Declare(glslang
        GIT_REPOSITORY "https://github.com/KhronosGroup/glslang.git"
        GIT_TAG "master"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(glslang)
    set(glslang_DEPENDENCY glslang)
endif()

##############################################################################################################
#
# VulkaSceneGraph
#
OPTION(BUILD_VulkanSceneGraph "Build VulkanSceneGraph" ON)
if (${BUILD_VulkanSceneGraph})
    FetchContent_Declare(vsg
        GIT_REPOSITORY https://github.com/vsg-dev/VulkanSceneGraph.git
        GIT_TAG "master"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(vsg ${glslang_DEPENDENCY})
    set(vsg_DEPENDENCY vsg)
endif()

##############################################################################################################
#
# osg2vsg, only build if OpenSceneGraph is installed
#
find_package(OpenSceneGraph QUIET)
OPTION(BUILD_osg2vsg "Build osg2vsg" ${OpenSceneGraph_FOUND})
if (${BUILD_osg2vsg})
    FetchContent_Declare(osg2vsg
        GIT_REPOSITORY "https://github.com/vsg-dev/osg2vsg.git"
        GIT_TAG "master"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(osg2vsg ${vsg_DEPENDENCY})
    set(osg2vsg_DEPENDENCY osg2vsg)
endif()

##############################################################################################################
#
# assimp
#
OPTION(BUILD_assimp "Build assimp" ON)
if (${BUILD_assimp})
    FetchContent_Declare(assimp
        GIT_REPOSITORY "https://github.com/assimp/assimp.git"
        GIT_TAG "v5.2.5"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(assimp)

    # add definitions to enable assimp locally built lib to be recognized by vsgXchange
    add_library(assimp::assimp ALIAS assimp)
    set(assimp_FOUND TRUE CACHE INTERNAL "assimp found.")
    set(CMAKE_DISABLE_FIND_PACKAGE_assimp TRUE CACHE INTERNAL "Disable find_package(assimp) as it's not necessary.")
    set(assimp_VERSION "5.2.5") # use the same version as pulled in the GIT_TAG above.

    set(assimp_DEPENDENCY assimp)
endif()

##############################################################################################################
#
# vsgXchange
#
OPTION(BUILD_vsgXchange "Build vsgXchange" ON)
if (${BUILD_vsgXchange})
    FetchContent_Declare(vsgxchange
        GIT_REPOSITORY https://github.com/vsg-dev/vsgXchange.git
        GIT_TAG master
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(vsgxchange ${vsg_DEPENDENCY} ${osg2vsg_DEPENDENCY} ${assimp_DEPENDENCY})
    set(vsgXchange_DEPENDENCY vsgXchange)
endif()

##############################################################################################################
#
# vsgImGui
#
OPTION(BUILD_vsgImGui "Build vsgImGui" ON)
if (${BUILD_vsgImGui})
    FetchContent_Declare(vsgimgui
        GIT_REPOSITORY "https://github.com/vsg-dev/vsgImGui.git"
        GIT_TAG "master"
        GIT_SUBMODULES "src/imgui"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(vsgimgui ${vsg_DEPENDENCY} )
endif()


##############################################################################################################
#
# vsgQt
#
OPTION(BUILD_vsgQt "Build vsgQt" ON)
if (${BUILD_vsgQt})
    FetchContent_Declare(vsgqt
        GIT_REPOSITORY "https://github.com/vsg-dev/vsgQt.git"
        GIT_TAG "master"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(vsgqt ${vsg_DEPENDENCY} )
endif()

##############################################################################################################
#
# vsgExamples
#
OPTION(BUILD_vsgExamples "Build vsgExamples" ON)
if (${BUILD_vsgExamples})
    FetchContent_Declare(vsgexamples
        GIT_REPOSITORY "https://github.com/vsg-dev/vsgExamples.git"
        GIT_TAG "master"
        GIT_PROGRESS TRUE
    )
    FetchContent_MakeAvailable(vsgexamples ${vsg_DEPENDENCY} ${vsgXchange_DEPENDENCY} ${vsgImGui_DEPENDENCY})
endif()
