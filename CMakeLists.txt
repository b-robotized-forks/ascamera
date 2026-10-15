cmake_minimum_required(VERSION 3.8)
project(ascamera)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic -O3)
endif()

add_compile_options(-std=c++17)

# get gcc -v target
execute_process(COMMAND bash ./scripts/gettarget.sh WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} OUTPUT_VARIABLE gcc_target)
string(REGEX REPLACE "\n$" "" gcc_target "${gcc_target}")
message("default compiler:" ${gcc_target}-g++)

# ARM
# SET(CROSS_COMPILE arm-linux-gnueabihf)
# SET(CROSS_COMPILE aarch64-linux-gnu)

if(CROSS_COMPILE)
if (${CROSS_COMPILE} MATCHES "aarch64")
SET(CMAKE_SYSTEM_PROCESSOR aarch64)
else()
SET(CMAKE_SYSTEM_PROCESSOR arm)
endif()
SET(CMAKE_C_COMPILER ${CROSS_COMPILE}-gcc)
SET(CMAKE_CXX_COMPILER ${CROSS_COMPILE}-g++)
SET(CMAKE_STRIP ${CROSS_COMPILE}-strip)
endif()


# find dependencies
find_package(ament_cmake REQUIRED)
# uncomment the following section in order to fill in
# further dependencies manually.
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
# find_package(pcl_ros REQUIRED)
find_package(pcl_conversions REQUIRED)
find_package(PCL COMPONENTS common REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
# find_package(tf2_geometry_msgs REQUIRED)

include_directories(
  ${catkin_INCLUDE_DIRS}
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  ${CMAKE_CURRENT_SOURCE_DIR}/libs/include
)

if(CROSS_COMPILE)
set(ASCAMERA_SDK_LIB_DIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/lib/${CROSS_COMPILE})
else()
set(ASCAMERA_SDK_LIB_DIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/lib/${gcc_target})
endif()
link_directories(${ASCAMERA_SDK_LIB_DIR})

add_executable(
  ascamera_node 
  src/ascamera_node.cpp 
  src/CameraPublisher.cpp 
  src/CameraSrv.cpp 
  src/Camera.cpp
  src/TfTreeFrameIdInfo.cpp
  )
target_link_libraries(ascamera_node
  rclcpp::rclcpp
  ${std_msgs_TARGETS}
  ${sensor_msgs_TARGETS}
  ${pcl_conversions_TARGETS}
  ${geometry_msgs_TARGETS}
  tf2::tf2
  tf2_ros::tf2_ros
  ${PCL_LIBRARIES}
)

target_include_directories(ascamera_node PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
target_compile_features(ascamera_node PUBLIC c_std_99 cxx_std_17)  # Require C99 and C++17

## Specify libraries to link a library or executable target against
target_link_libraries(ascamera_node
  AngstrongCameraSdk
  zmq
  Filt
  asuvc
  asusb
  turbojpeg
  alg_kunlun
  jpeg
  AngKondyorArith
  ${PCL_LIBRARIES}
)

install(TARGETS ascamera_node
  DESTINATION lib/${PROJECT_NAME})

# Install the vendor SDK libraries next to the node and let the node find them there, so no
# LD_LIBRARY_PATH is needed. They are kept out of lib/ because ament puts lib/ on LD_LIBRARY_PATH
# and the unversioned libjpeg.so/libzmq.so would shadow the system libraries.
set_target_properties(ascamera_node PROPERTIES INSTALL_RPATH "$ORIGIN")
install(DIRECTORY ${ASCAMERA_SDK_LIB_DIR}/
  DESTINATION lib/${PROJECT_NAME}
  FILES_MATCHING PATTERN "*.so*")

install(DIRECTORY launch configurationfiles
  DESTINATION share/${PROJECT_NAME})

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  # the following line skips the linter which checks for copyrights
  # uncomment the line when a copyright and license is not present in all source files
  #set(ament_cmake_copyright_FOUND TRUE)
  # the following line skips cpplint (only works in a git repo)
  # uncomment the line when this package is not in a git repo
  #set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()

