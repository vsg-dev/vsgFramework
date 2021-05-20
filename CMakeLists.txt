cmake_minimum_required(VERSION 3.5)

project(vsgFramework)

include(ExternalProject)

set(DOWNLOADS_DIR ${PROJECT_BINARY_DIR}/downloads)
set(COMPONENTS_DIR ${PROJECT_BINARY_DIR})
set(BUILD_INSTALL_PREFIX ${PROJECT_BINARY_DIR} CACHE STRING "Install prefix ")
set(COMPONENT_CMAKE_OPTIONS -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCMAKE_INSTALL_PREFIX=${BUILD_INSTALL_PREFIX} -DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS})

# set compilation to use C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Change the default build type to Release
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif(NOT CMAKE_BUILD_TYPE)

install(DIRECTORY ${INSTALL_PREFIX}/bin DESTINATION .)
install(DIRECTORY ${INSTALL_PREFIX}/lib DESTINATION .)
install(DIRECTORY ${INSTALL_PREFIX}/include DESTINATION .)

add_custom_target(clobber
    COMMAND git clean -d -f -x
)

find_package(Threads)

find_library(DL_LIBRARY dl)
if(NOT DL_LIBRARY)
    set(DL_LIBRARY "") # change from NOTFOUND to empty when passed to linker
endif()

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
# glslang
#
OPTION(build_glslang "Build glslang" ON)
if (${build_glslang})
    set(glslang "glslang")
    ExternalProject_Add(glslang
        PREFIX ${COMPONENTS_DIR} CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS}
        GIT_REPOSITORY "https://github.com/KhronosGroup/glslang.git" GIT_TAG "master"
    )
endif()

##############################################################################################################
#
# VulkaSceneGraph
#
OPTION(build_VulkanSceneGraph "Build VulkanSceneGraph" ON)
if (${build_VulkanSceneGraph})
    set(VulkanSceneGraph "VulkanSceneGraph")
    ExternalProject_Add(VulkanSceneGraph
        PREFIX ${COMPONENTS_DIR} CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS}
        GIT_REPOSITORY "https://github.com/vsg-dev/VulkanSceneGraph.git" GIT_TAG "master"
    )
    ExternalProject_Add_StepDependencies(VulkanSceneGraph build ${glslang})
endif()


##############################################################################################################
#
# vsgGIS
#
OPTION(build_vsgGIS "Build vsgGIS" ON)
if (${build_vsgGIS})
    set(vsgGIS "vsgGIS")
    ExternalProject_Add(vsgGIS
        PREFIX ${COMPONENTS_DIR} CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS}
        GIT_REPOSITORY "https://github.com/vsg-dev/vsgGIS.git" GIT_TAG "master"

    )
    ExternalProject_Add_StepDependencies(vsgGIS build ${VulkanSceneGraph})
endif()


##############################################################################################################
#
# assimp
#
OPTION(build_assimp "Build assimp" ON)
if (${build_assimp})
    set(assimp "assimp")
    ExternalProject_Add(assimp
        PREFIX ${COMPONENTS_DIR} CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS}
        GIT_REPOSITORY "https://github.com/assimp/assimp.git" GIT_TAG "master"
    )
endif()


##############################################################################################################
#
# vsgXchange
#
OPTION(build_vsgXchange "Build vsgXchange" ON)
if (${build_vsgXchange})
    set(vsgGIS "vsgGIS")
    ExternalProject_Add(vsgXchange
        PREFIX ${COMPONENTS_DIR}
        GIT_REPOSITORY "https://github.com/vsg-dev/vsgXchange.git"
        GIT_TAG "master"
        CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS}
    )
    ExternalProject_Add_StepDependencies(vsgXchange build ${assimp} ${VulkanSceneGraph} ${vsgGIS})
endif()


##############################################################################################################
#
# vsgImGui
#
OPTION(build_vsgImGui "Build vsgImGui" ON)
if (${build_vsgImGui})
    set(vsgGIS "vsgImGui")
    ExternalProject_Add(vsgImGui
        PREFIX ${COMPONENTS_DIR}
        GIT_REPOSITORY "https://github.com/vsg-dev/vsgImGui.git"
        GIT_TAG "master"
        CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS}
    )
    ExternalProject_Add_StepDependencies(vsgImGui build ${VulkanSceneGraph} ${vsgXchange})
endif()


##############################################################################################################
#
# vsgExamples
#
OPTION(build_vsgExamples "Build vsgExamples" ON)
if (${build_vsgExamples})
    set(vsgExamples "vsgExamples")
    ExternalProject_Add(vsgExamples
        PREFIX ${COMPONENTS_DIR}
        GIT_REPOSITORY "https://github.com/vsg-dev/vsgExamples.git"
        GIT_TAG "master"
        CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS}
    )
    ExternalProject_Add_StepDependencies(vsgExamples build ${VulkanSceneGraph} ${vsgXchange} ${vsgImGui})
endif()
