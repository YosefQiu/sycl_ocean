cmake_minimum_required(VERSION 3.12 FATAL_ERROR)
project(MOPS VERSION 0.0.1)

set(PRJ_COMPILE_FEATURES)
set(CMAKE_C_COMPILER icx)
set(CMAKE_CXX_COMPILER icpx)



# 1️⃣ **找到 VTK**
FIND_PACKAGE(VTK COMPONENTS
  CommonColor
  CommonCore
  CommonDataModel
  FiltersCore
  FiltersSources
  FiltersGeometry
  IOLegacy
  IOXML
  InteractionStyle
)

if(VTK_FOUND)
    message(STATUS "VTK found at: ${VTK_LIBRARIES}")
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib;${VTK_LIBRARY_DIRS}")
else()
    message(FATAL_ERROR "VTK NOT FOUND!")
endif()

# 2️⃣ **找到 YAML-CPP**
find_package(yaml-cpp REQUIRED)

# 3️⃣ **找到 NDARRAY**
find_package(ndarray REQUIRED)
if(NDARRAY_FOUND)
    message(STATUS "ndarray found at: ${NDARRAY_DIR}")
else()
    message(FATAL_ERROR "ndarray NOT FOUND!")
endif()

find_package(netCDF REQUIRED C CXX)

include_directories(${NETCDF_INCLUDE_DIRS})
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsycl -fno-fast-math")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsycl -fno-fast-math")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsycl")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsycl-targets=nvptx64-nvidia-cuda")

message(STATUS "CMAKE_C_COMPILER = ${CMAKE_C_COMPILER}")
message(STATUS "CMAKE_CXX_COMPILER = ${CMAKE_CXX_COMPILER}")
message(STATUS "CMAKE_CXX_FLAGS =  ${CMAKE_CXX_FLAGS}")

# 4️⃣ **包含头文件路径**
include_directories(
  third_part
  src
  include
  ${ndarray_INCLUDE_DIRS}   
  ${VTK_INCLUDE_DIRS}       
  third/netcdf-cxx4
)

# 5️⃣ **查找源文件**
file(GLOB_RECURSE SOURCE_FILES 
    src/*.c
    src/*.cpp
    src/*.cc
    third_part/netcdf-cxx4/*.cpp)

if(WIN32)
    add_definitions(-D_AMD64_)
endif()

message("Following files are compiled: ${SOURCE_FILES}")

# 6️⃣ **生成可执行文件**
add_executable(${PROJECT_NAME} ${SOURCE_FILES})

# 7️⃣ **链接库**
target_link_libraries(${PROJECT_NAME} PRIVATE
    ${ndarray_LIBRARIES}   # 使用 `ndarray`
    ${VTK_LIBRARIES}       # 使用 `VTK`
    yaml-cpp
    netCDF::netcdf
    netcdf
)

# 8️⃣ **VTK 自动初始化**
vtk_module_autoinit(
    TARGETS ${PROJECT_NAME}
    MODULES ${VTK_LIBRARIES}
)

target_compile_features(${PROJECT_NAME} PRIVATE ${PRJ_COMPILE_FEATURES})

# 9️⃣ **安装目标**
install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Targets
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

#  🔟 **安装头文件**
install(DIRECTORY include/ DESTINATION include)

# 🔟 **生成 CMake 配置文件**
include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
    INSTALL_DESTINATION lib/cmake/${PROJECT_NAME}
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
    DESTINATION lib/cmake/${PROJECT_NAME}
)

# 🔟 **导出 CMake 目标**
export(EXPORT ${PROJECT_NAME}Targets
    FILE ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Targets.cmake
    NAMESPACE ${PROJECT_NAME}::
)
