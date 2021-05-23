cmake_minimum_required(VERSION 3.7)

project(vsgFramework)

include(ExternalProject)

# Change the default build type to Release
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif(NOT CMAKE_BUILD_TYPE)

set(DOWNLOADS_DIR ${PROJECT_BINARY_DIR}/downloads)
set(COMPONENTS_DIR ${PROJECT_BINARY_DIR})
set(COMPONENT_INSTALL_PREFIX ${PROJECT_BINARY_DIR} CACHE STRING "Install prefix for components ")
set(COMPONENT_CMAKE_OPTIONS -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCMAKE_INSTALL_PREFIX=${COMPONENT_INSTALL_PREFIX} -DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS})

install(DIRECTORY ${INSTALL_PREFIX}/bin DESTINATION .)
install(DIRECTORY ${INSTALL_PREFIX}/lib DESTINATION .)
install(DIRECTORY ${INSTALL_PREFIX}/include DESTINATION .)

add_custom_target(clobber
    COMMAND git clean -d -f -x
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

# ExternalComponent macro to help set up external projects and their depdendecies
macro(ExternalComponent component_name git_repository git_tag)
    if (build_${component_name})
        set(${component_name} "${component_name}")

        ExternalProject_Add(${component_name}
            PREFIX ${COMPONENTS_DIR} 
            CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS} 
            GIT_PROGRESS ON
            GIT_REPOSITORY ${git_repository} 
            GIT_TAG ${git_tag}
        )
        
        ExternalProject_Add_StepDependencies(${component_name} build "${ARGN}")
    endif()
endmacro()

# glslang
OPTION(build_glslang "Build glslang" ON)
ExternalComponent(glslang "https://github.com/KhronosGroup/glslang.git" "master")

# VulkaSceneGraph
OPTION(build_VulkanSceneGraph "Build VulkanSceneGraph" ON)
ExternalComponent(VulkanSceneGraph "https://github.com/vsg-dev/VulkanSceneGraph.git" "master" ${glslang})

# vsgGIS
OPTION(build_vsgGIS "Build vsgGIS" ON)
ExternalComponent(vsgGIS "https://github.com/vsg-dev/vsgGIS.git" "master" ${VulkanSceneGraph})

# assimp
OPTION(build_assimp "Build assimp" ON)
ExternalComponent(assimp "https://github.com/assimp/assimp.git" "master")

# vsgXchange
OPTION(build_vsgXchange "Build vsgXchange" ON)
ExternalComponent(vsgXchange "https://github.com/vsg-dev/vsgXchange.git" "master" ${assimp} ${VulkanSceneGraph} ${vsgGIS})

# vsgImGui
OPTION(build_vsgImGui "Build vsgImGui" ON)
ExternalComponent(vsgImGui "https://github.com/vsg-dev/vsgImGui.git" "master" ${VulkanSceneGraph} ${vsgXchange})

# vsgExamples
OPTION(build_vsgExamples "Build vsgExamples" ON)
ExternalComponent(vsgExamples "https://github.com/vsg-dev/vsgExamples.git" "master" ${VulkanSceneGraph} ${vsgXchange} ${vsgImGui})
