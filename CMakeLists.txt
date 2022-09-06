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
set(COMPONENT_CMAKE_OPTIONS
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX=${COMPONENT_INSTALL_PREFIX}
    -DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS}
    -DCMAKE_PREFIX_PATH=${COMPONENT_INSTALL_PREFIX}
)

install(DIRECTORY ${INSTALL_PREFIX}/bin DESTINATION .)
install(DIRECTORY ${INSTALL_PREFIX}/lib DESTINATION .)
install(DIRECTORY ${INSTALL_PREFIX}/include DESTINATION .)

add_custom_target(clean-all)
add_custom_target(clobber
    COMMAND git -C ${CMAKE_SOURCE_DIR} clean -d -f -x
)

add_custom_target(clobber-all)

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
# Usage:
#   ExternalComponent(component_name git_repository_url git_tag [optional_component_dependencies...])
macro(ExternalComponent component_name in_git_repository in_git_tag)
    # migrate deprecated setting

    set(${component_name}_repository ${in_git_repository} CACHE STRING "${component_name} repository" )
    set(${component_name}_tag ${in_git_tag} CACHE STRING "${component_name} tag" )

    if(build_${component_name})
        set(BUILD_${component_name} ON)
        unset(build_${component_name})
    endif()
    if (BUILD_${component_name})
        set(${component_name} "${component_name}")

        ExternalProject_Add(${component_name}
            PREFIX ${COMPONENTS_DIR} 
            CMAKE_ARGS ${COMPONENT_CMAKE_OPTIONS} ${${component_name}_CMAKE_OPTIONS}
            GIT_PROGRESS ON
            GIT_REPOSITORY ${${component_name}_repository}
            GIT_TAG ${${component_name}_git_tag}
        )
        
        ExternalProject_Add_StepDependencies(${component_name} build "${ARGN}")

        add_custom_target(clean-${component_name}
            COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}/src/${component_name}-build
                                     --target clean
        )
        add_dependencies(clean-all clean-${component_name})

        add_custom_target(clobber-${component_name}
            COMMAND git -C ${CMAKE_BINARY_DIR}/src/${component_name} clean -d -f -x
        )
        add_dependencies(clobber-all clobber-${component_name})
    endif()
endmacro()

OPTION(BUILD_SHARED_LIBS "Build shared libraries" OFF)

# glslang
OPTION(BUILD_glslang "Build glslang" ON)
ExternalComponent(glslang "https://github.com/KhronosGroup/glslang.git" "master")

OPTION(BUILD_SPIRV_headers "Build SPIRV headers" ON)
ExternalComponent(SPIRV_headers "https://github.com/KhronosGroup/SPIRV-Headers.git" "master")

OPTION(BUILD_SPIRV_tools "Build SPIRV tools" ON)
set(SPIRV_tools_CMAKE_OPTIONS -DSPIRV-Headers_SOURCE_DIR=${COMPONENTS_DIR})
ExternalComponent(SPIRV_tools "https://github.com/KhronosGroup/SPIRV-Tools.git" "master" ${SPIRV_headers})

# OpenSceneGraph
OPTION(BUILD_OpenSceneGraph "Build OpenSceneGraph" OFF)
set(OpenSceneGraph_CMAKE_OPTIONS -DLIB_POSTFIX=64 -DDYNAMIC_OPENSCENEGRAPH=${BUILD_SHARED_LIBS})
ExternalComponent(OpenSceneGraph "https://github.com/openscenegraph/OpenSceneGraph.git" "OpenSceneGraph-3.6")

# VulkaSceneGraph
OPTION(BUILD_VulkanSceneGraph "Build VulkanSceneGraph" ON)
ExternalComponent(VulkanSceneGraph "https://github.com/vsg-dev/VulkanSceneGraph.git" "master" ${glslang} ${SPIRV_tools})

# assimp
OPTION(BUILD_assimp "Build assimp" ON)
ExternalComponent(assimp "https://github.com/assimp/assimp.git" "master")

# osg2vsg
OPTION(BUILD_osg2vsg "Build osg2vsg" ON)
ExternalComponent(osg2vsg "https://github.com/vsg-dev/osg2vsg.git" "master" ${VulkanSceneGraph} ${assimp} ${OpenSceneGraph})

# vsgXchange
OPTION(BUILD_vsgXchange "Build vsgXchange" ON)
ExternalComponent(vsgXchange "https://github.com/vsg-dev/vsgXchange.git" "master" ${VulkanSceneGraph} ${assimp} ${OpenSceneGraph} ${osg2vsg} )

# vsgImGui
OPTION(BUILD_vsgImGui "Build vsgImGui" ON)
ExternalComponent(vsgImGui "https://github.com/vsg-dev/vsgImGui.git" "master" ${VulkanSceneGraph})

# vsgExamples
OPTION(BUILD_vsgExamples "Build vsgExamples" ON)
ExternalComponent(vsgExamples "https://github.com/vsg-dev/vsgExamples.git" "master" ${VulkanSceneGraph} ${assimp} ${osg2vsg} ${vsgXchange} ${vsgImGui})

# vsgQt
OPTION(BUILD_vsgQt "Build vsgQt" ON)
ExternalComponent(vsgQt "https://github.com/vsg-dev/vsgQt.git" "master" ${VulkanSceneGraph} ${vsgXchange})

# vsgUnity
OPTION(BUILD_vsgUnity "Build vsgUnity" OFF)
ExternalComponent(vsgUnity "https://github.com/vsg-dev/vsgUnity.git" "master" ${VulkanSceneGraph})

# MyFirstVsgApplication
OPTION(BUILD_MyFirstVsgApplication "Build MyFirstVsgApplication" OFF)
ExternalComponent(MyFirstVsgApplication "https://github.com/vsg-dev/MyFirstVsgApplication.git" "master" ${VulkanSceneGraph})
