cmake_minimum_required(VERSION 3.5)
project(urg_node2)

# Default to C99
if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

# Default to C++14
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_components REQUIRED)
find_package(rclcpp_lifecycle REQUIRED)
find_package(lifecycle_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(diagnostic_updater REQUIRED)
find_package(laser_proc REQUIRED)

set(URG_LIBRARY_DIR urg_library/current)
set(URG_LIBRARY_C_INCLUDE_DIR ${URG_LIBRARY_DIR}/include/c)
set(URG_LIBRARY_SRC_DIR ${URG_LIBRARY_DIR}/src)

add_library(urg_c STATIC
  ${URG_LIBRARY_SRC_DIR}/urg_sensor.c
  ${URG_LIBRARY_SRC_DIR}/urg_utils.c
  ${URG_LIBRARY_SRC_DIR}/urg_debug.c
  ${URG_LIBRARY_SRC_DIR}/urg_connection.c
  ${URG_LIBRARY_SRC_DIR}/urg_ring_buffer.c
  ${URG_LIBRARY_SRC_DIR}/urg_serial.c
  ${URG_LIBRARY_SRC_DIR}/urg_serial_utils.c
  ${URG_LIBRARY_SRC_DIR}/urg_tcpclient.c
)

target_include_directories(urg_c PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${URG_LIBRARY_C_INCLUDE_DIR}>
  $<INSTALL_INTERFACE:include>
)

if (WIN32)
  target_link_libraries(urg_c Ws2_32.lib Setupapi.lib)
endif()

add_library(${PROJECT_NAME} SHARED src/urg_node2.cpp)

if (WIN32)
  set_target_properties(${PROJECT_NAME} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS true)
endif ()

target_compile_definitions(${PROJECT_NAME} PUBLIC
  $<$<CXX_COMPILER_ID:MSVC>:_USE_MATH_DEFINES NOMINMAX>
)

target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${URG_LIBRARY_C_INCLUDE_DIR}>
  $<INSTALL_INTERFACE:include>
)

ament_target_dependencies(${PROJECT_NAME} rclcpp rclcpp_components rclcpp_lifecycle lifecycle_msgs sensor_msgs diagnostic_updater laser_proc)
rclcpp_components_register_node(${PROJECT_NAME}
  PLUGIN "urg_node2::UrgNode2"
  EXECUTABLE urg_node2_node)
target_link_libraries(urg_node2 urg_c)

file(GLOB URG_HEADER_FILES "${URG_LIBRARY_C_INCLUDE_DIR}/*.h")
install(FILES ${URG_HEADER_FILES}
  DESTINATION include/urg_c
)

install(TARGETS urg_node2 urg_c
  EXPORT ${PROJECT_NAME}
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

install(DIRECTORY launch config
  DESTINATION share/${PROJECT_NAME}
)

if(BUILD_TESTING)
  find_package(ament_cmake_gtest)
  ament_add_gtest(urg_node2_test src/urg_node2.cpp test/urg_node2_test.cpp TIMEOUT 200)
  ament_target_dependencies(urg_node2_test rclcpp rclcpp_components rclcpp_lifecycle lifecycle_msgs sensor_msgs diagnostic_updater laser_proc)
  target_link_libraries(urg_node2_test ${PROJECT_NAME})
endif()

# disable tool tests, because a lot of errors occur in urg_library
#if(BUILD_TESTING)
#  find_package(ament_lint_auto REQUIRED)
#  # the following line skips the linter which checks for copyrights
#  # uncomment the line when a copyright and license is not present in all source files
#  #set(ament_cmake_copyright_FOUND TRUE)
#  # the following line skips cpplint (only works in a git repo)
#  # uncomment the line when this package is not in a git repo
#  #set(ament_cmake_cpplint_FOUND TRUE)
#  ament_lint_auto_find_test_dependencies()
#endif()

ament_export_targets(${PROJECT_NAME})

ament_package()
