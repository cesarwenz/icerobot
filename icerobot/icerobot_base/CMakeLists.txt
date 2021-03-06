cmake_minimum_required(VERSION 2.8.3)
project(icerobot_base)

# Verify Phoenix git submodule
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/Phoenix-Linux-SocketCAN-Example/include)
   message(INFO "Succesfully loaded CTRE Phoenix Library")
else()
  message(FATAL_ERROR
    "Failed to find CTRE Phoenix Library! "
    "Do you need to initialize the submodule?: git submodule update --init --recursive"
  )
endif()

# Set CTRE_ARCH
if (${CMAKE_LIBRARY_ARCHITECTURE} STREQUAL "arm-linux-gnueabihf")
  set(CTRE_ARCH "raspberry")
elseif (${CMAKE_LIBRARY_ARCHITECTURE} STREQUAL "aarch64-linux-gnu")
  set(CTRE_ARCH "jetsontx")
elseif (${CMAKE_LIBRARY_ARCHITECTURE} STREQUAL "x86_64-linux-gnu")
  set(CTRE_ARCH "x86-64")
else()
  message(FATAL_ERROR "Unsupported architecture for CTRE Phoenix")
endif()

## Compile as C++11, supported in ROS Kinetic and newer
add_compile_options(-std=c++17)

## Find catkin macros and libraries
## if COMPONENTS list like find_package(catkin REQUIRED COMPONENTS xyz)
## is used, also find other catkin packages
find_package(catkin REQUIRED
        COMPONENTS
        roscpp
        geometry_msgs
        dynamic_reconfigure
        roslaunch
        message_generation
	controller_manager
	hardware_interface
        )

## System dependencies are found with CMake's conventions
# find_package(Boost REQUIRED COMPONENTS system)
find_package(Threads REQUIRED)

################################################
## Declare ROS messages, services and actions ##
################################################

## To declare and build messages, services or actions from within this
## package, follow these steps:
## * Let MSG_DEP_SET be the set of packages whose message types you use in
##   your messages/services/actions (e.g. std_msgs, actionlib_msgs, ...).
## * In the file package.xml:
##   * add a build_depend tag for "message_generation"
##   * add a build_depend and a exec_depend tag for each package in MSG_DEP_SET
##   * If MSG_DEP_SET isn't empty the following dependency has been pulled in
##     but can be declared for certainty nonetheless:
##     * add a exec_depend tag for "message_runtime"
## * In this file (CMakeLists.txt):
##   * add "message_generation" and every package in MSG_DEP_SET to
##     find_package(catkin REQUIRED COMPONENTS ...)
##   * add "message_runtime" and every package in MSG_DEP_SET to
##     catkin_package(CATKIN_DEPENDS ...)
##   * uncomment the add_*_files sections below as needed
##     and list every .msg/.srv/.action file to be processed
##   * uncomment the generate_messages entry below
##   * add every package in MSG_DEP_SET to generate_messages(DEPENDENCIES ...)

## Generate messages in the 'msg' folder
add_message_files(
   FILES
   MotorControl.msg
   MotorAngle.msg
   MotorStatus.msg
)

## Generate services in the 'srv' folder
add_service_files(
  FILES
  Reset.srv
)

## Generate actions in the 'action' folder
# add_action_files(
#   FILES
#   Action1.action
#   Action2.action
# )

## Generate added messages and services with any dependencies listed here
generate_messages(
   DEPENDENCIES
   std_msgs
)

## add dynamic reconfigure api
generate_dynamic_reconfigure_options(
        ${PROJECT_SOURCE_DIR}/cfg/Talon.cfg
)


# List of Phoenix static libraries
set(PHOENIX_LIBRARIES CTRE_Phoenix
                      CTRE_PhoenixCCI
                      CTRE_PhoenixDiagnostics
                      CTRE_PhoenixPlatform_socketcan
                      CTRE_PhoenixCanutils
                      CTRE_PhoenixCore
)

# Create a target for each Phoenix library
foreach(LIBRARY ${PHOENIX_LIBRARIES})
  add_library(${LIBRARY} STATIC IMPORTED)
  set(LIBRARY_LOCATION ${CMAKE_CURRENT_SOURCE_DIR}/Phoenix-Linux-SocketCAN-Example/lib/${CTRE_ARCH}/lib${LIBRARY}.a)
  SET_TARGET_PROPERTIES(${LIBRARY} PROPERTIES IMPORTED_LOCATION ${LIBRARY_LOCATION})
  install (
    FILES ${LIBRARY_LOCATION}
    DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  )
endforeach()

###################################
## catkin specific configuration ##
###################################
## The catkin_package macro generates cmake config files for your package
## Declare things to be passed to dependent projects
## INCLUDE_DIRS: uncomment this if your package contains header files
## LIBRARIES: libraries you create in this project that dependent projects also need
## CATKIN_DEPENDS: catkin_packages dependent projects also need
## DEPENDS: system dependencies of this project that dependent projects also need
catkin_package(
        INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include ${CMAKE_CURRENT_SOURCE_DIR}/Phoenix-Linux-SocketCAN-Example/include
        LIBRARIES ${PHOENIX_LIBRARIES}
        CATKIN_DEPENDS roscpp geometry_msgs dynamic_reconfigure roslaunch message_runtime controller_manager hardware_interface
)

###########
## Build ##
###########

## Specify additional locations of header files
## Your package locations should be listed before other locations
include_directories(
        ${CMAKE_CURRENT_SOURCE_DIR}/include ${CMAKE_CURRENT_SOURCE_DIR}/Phoenix-Linux-SocketCAN-Example/include
        SYSTEM ${catkin_INCLUDE_DIRS}
)

## Declare a C++ executable
## With catkin_make all packages are built within a single CMake context
## The recommended prefix ensures that target names across packages don't collide
# add_executable(${PROJECT_NAME}_node src/phoenix_node.cpp)
#add_executable(phoenix_node ${PROJECT_SOURCE_DIR}/example.cpp)
add_executable(icerobot_base_control ${PROJECT_SOURCE_DIR}/src/RosControl.cpp)
add_dependencies(icerobot_base_control ${PROJECT_NAME}_gencfg ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})

add_executable(icerobot_base_node
        ${PROJECT_SOURCE_DIR}/src/TalonNode.cpp
        ${PROJECT_SOURCE_DIR}/src/MotorControl.cpp
        )
add_dependencies(icerobot_base_node ${PROJECT_NAME}_gencfg ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})


## Add cmake target dependencies of the executable
## same as for the library above
#add_dependencies(${PROJECT_NAME}_node ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})

## Specify libraries to link a library or executable target against
target_link_libraries(icerobot_base_control
        ${catkin_LIBRARIES}
        Threads::Threads
        ${PHOENIX_LIBRARIES}
        )
target_link_libraries(icerobot_base_node
        ${catkin_LIBRARIES}
        Threads::Threads
        ${PHOENIX_LIBRARIES}
        )

#############
## Install ##
#############

## Install node executables
install(TARGETS icerobot_base_node icerobot_base_control 
        ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
        LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
        RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
        )
## Mark cpp header files for installation
install(DIRECTORY include/
        DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
        FILES_MATCHING PATTERN "*.h"
        PATTERN ".svn" EXCLUDE
        )
## Mark launch files for installation
install(DIRECTORY launch/
        DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/launch
        PATTERN ".svn" EXCLUDE
        )
