cmake_minimum_required(VERSION 2.8.3)
project(velodyne_driver)



# Set minimum C++ standard to C++11
if (NOT "${CMAKE_CXX_STANDARD_COMPUTED_DEFAULT}")
  message(STATUS "Changing CXX_STANDARD from C++98 to C++11")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif ("${CMAKE_CXX_STANDARD_COMPUTED_DEFAULT}" STREQUAL "98")
  message(STATUS "Changing CXX_STANDARD from C++98 to C++11")
  set(CMAKE_CXX_STANDARD 11)
endif()

if(MSVC)
  set(CMAKE_CXX_STANDARD 11)
endif()

set(${PROJECT_NAME}_CATKIN_DEPS 
    diagnostic_updater
    dynamic_reconfigure
    nodelet
    roscpp
    tf
    velodyne_msgs)

find_package(catkin REQUIRED COMPONENTS ${${PROJECT_NAME}_CATKIN_DEPS} roslint)

# This driver uses Boost threads
find_package(Boost REQUIRED COMPONENTS thread)

include_directories(include ${Boost_INCLUDE_DIR} ${catkin_INCLUDE_DIRS})


# libpcap provides no pkg-config or find_package module:
if (MSVC)

add_definitions("/D_CRT_SECURE_NO_WARNINGS")
add_definitions("/D_WIN32")

add_compile_options("/wd4267")

find_package(PkgConfig)
set(ENV{PKG_CONFIG_PATH} "C:/opt/vcpkg/installed/x64-windows/lib/pkgconfig")
if(PKG_CONFIG_FOUND)
pkg_check_modules(libpcap REQUIRED libpcap)

include_directories(${libpcap_INCLUDE_DIRS})
link_directories(${libpcap_LIBRARY_DIRS})
endif()
else()
set(libpcap_LIBRARIES -lpcap)
endif()


# Generate dynamic_reconfigure server
generate_dynamic_reconfigure_options(cfg/VelodyneNode.cfg)

# objects needed by other ROS packages that depend on this one
catkin_package(CATKIN_DEPENDS ${${PROJECT_NAME}_CATKIN_DEPS}
               INCLUDE_DIRS include
               LIBRARIES velodyne_input)

# compile the driver and input library
add_subdirectory(src/lib)
add_subdirectory(src/driver)

install(DIRECTORY include/${PROJECT_NAME}/
        DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION})
install(FILES nodelet_velodyne.xml
        DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})
install(DIRECTORY launch/
        DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/launch)
install(PROGRAMS src/vdump
        DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})

roslint_cpp()

if (CATKIN_ENABLE_TESTING)

  # these dependencies are only needed for unit testing
  find_package(roslaunch REQUIRED)
  find_package(rostest REQUIRED)

  # Download packet capture (PCAP) files containing test data.
  # Store them in devel-space, so rostest can easily find them.
  catkin_download_test_data(
    ${PROJECT_NAME}_tests_class.pcap
    http://download.ros.org/data/velodyne/class.pcap
    DESTINATION ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_SHARE_DESTINATION}/tests
    MD5 65808d25772101358a3719b451b3d015)
  catkin_download_test_data(
    ${PROJECT_NAME}_tests_32e.pcap
    http://download.ros.org/data/velodyne/32e.pcap
    DESTINATION ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_SHARE_DESTINATION}/tests
    MD5 e41d02aac34f0967c03a5597e1d554a9)
  catkin_download_test_data(
    ${PROJECT_NAME}_tests_vlp16.pcap
    http://download.ros.org/data/velodyne/vlp16.pcap
    DESTINATION ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_SHARE_DESTINATION}/tests
    MD5 f45c2bb1d7ee358274e423ea3b66fd73)
  
  # unit tests
  add_rostest(tests/pcap_node_hertz.test)
  add_rostest(tests/pcap_nodelet_hertz.test)
  add_rostest(tests/pcap_32e_node_hertz.test)
  add_rostest(tests/pcap_32e_nodelet_hertz.test)
  add_rostest(tests/pcap_vlp16_node_hertz.test)
  add_rostest(tests/pcap_vlp16_nodelet_hertz.test)
  
  # parse check all the launch/*.launch files
  roslaunch_add_file_check(launch)

  # unit test
  catkin_add_gtest(time_test tests/timeconversiontest.cpp)
  target_link_libraries(time_test
    ${catkin_LIBRARIES}
    ${Boost_LIBRARIES})
endif (CATKIN_ENABLE_TESTING)
