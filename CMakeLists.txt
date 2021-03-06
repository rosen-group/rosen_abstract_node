cmake_minimum_required(VERSION 3.0.2)
project(rosen_abstract_node)

find_library(MENU_LIBRARY menu)

set(DEPENDENCIES
  roscpp
  rospy
  std_msgs
  diagnostic_msgs
  actionlib
  actionlib_msgs
  diagnostic_updater
  actionlib_msgs
)

find_package(catkin REQUIRED
  COMPONENTS
    roslaunch
    message_runtime
    message_generation
    ${DEPENDENCIES}
)

find_package(Curses REQUIRED)

#############################
## Set C++ compile options ##
#############################

add_compile_options(-Wextra)

if(${ROCAT_RELEASE})
    add_compile_options(-g -O2 -fstack-protector-strong -Wformat -Werror=format-security -DNDEBUG -Wdate-time -D_FORTIFY_SOURCE=2 -fPIC)
    set(CMAKE_EXE_LINKER_FLAGS "-pie -fPIC")
endif()

add_compile_options(-std=c++14 -g -fstack-protector-strong -Werror -Wall -Wformat -Wformat-security -Werror=date-time)

##########################
## Execute python setup ##
##########################

catkin_python_setup()

###################################
## Generate messages and actions ##
###################################

add_message_files(
  FILES
  NodeState.msg
  NodeStateInfo.msg
  NodeTransition.msg
)

add_action_files(FILES StateTransition.action)

generate_messages(
  DEPENDENCIES
  actionlib_msgs
  std_msgs
)

####################################################
## Create/define catkin package with dependencies ##
####################################################

catkin_package(
    CATKIN_DEPENDS ${DEPENDENCIES}
    INCLUDE_DIRS include
    LIBRARIES rosen_abstract_node
)

########################################################################################
## Add 'include' subfolder and catkin include directories to search path for includes ##
########################################################################################

include_directories(
  ${PROJECT_SOURCE_DIR}/include
  ${catkin_INCLUDE_DIRS}
)

##########################
## Install python files ##
##########################

catkin_install_python(PROGRAMS scripts/python_dummy_node.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})

############################################################################
## Add the rosen_abstract_node library as target and link it to libraries ##
############################################################################

add_library(rosen_abstract_node
    src/RosenAbstractNode.cpp
    src/StateMachine.cpp
    src/AbstractNodeSm.cpp
    src/TestUtils.cpp)

add_dependencies(rosen_abstract_node ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(rosen_abstract_node ${catkin_LIBRARIES} ${LIBRARY_CONFIGURATION_LINK_THIRDPARTIES} ${ROCAT_PACKAGE_NON_CATKIN_THIRDPARTIES})

######################################################################
## Add the dummy_node executable as target and link it to libraries ##
######################################################################

add_executable(dummy_node
    src/DummyNode.cpp
)
add_dependencies(dummy_node ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(dummy_node ${catkin_LIBRARIES} rosen_abstract_node)

###############################################################################
## Add the execute_transitions executable as target and link it to libraries ##
###############################################################################

add_executable(execute_transitions
    src/ExecuteTransitions.cpp
)
add_dependencies(execute_transitions ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(execute_transitions ${catkin_LIBRARIES} rosen_abstract_node)

##############################################################################
## Add the manual_action_test executable as target and link it to libraries ##
##############################################################################

add_executable(manual_action_test
    src/ManualActionTest.cpp
)
add_dependencies(manual_action_test ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(manual_action_test ${catkin_LIBRARIES} rosen_abstract_node ${MENU_LIBRARY} ${CURSES_LIBRARIES})

##############################
## Install C++ header files ##
##############################

file(GLOB ROCAT_CPP_HEADERS "include/${PROJECT_NAME}/*.h")
install(FILES ${ROCAT_CPP_HEADERS} DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION})

###############################
## Install generated targets ##
###############################

install(TARGETS rosen_abstract_node dummy_node manual_action_test execute_transitions
        ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
        LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
        RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})

##########################
## Install launch files ##
##########################

install(DIRECTORY launch DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})

if(CATKIN_ENABLE_TESTING)

  ############################################
  ## Add CPP Unit tests (GTESTS) with GMOCK ##
  ############################################

  set(GTEST_LIBRARIES gmock)

  file(GLOB test_cpps "test/*.cpp")

  catkin_add_gtest(${PROJECT_NAME}-test ${test_cpps} WORKING_DIRECTORY)
  add_dependencies(${PROJECT_NAME}-test ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
  target_link_libraries(${PROJECT_NAME}-test gmock ${PACKAGE_CONFIGURATION_EXPORTED_LIBRARIES} rosen_abstract_node ${catkin_LIBRARIES})

  #############################
  ## Add ROSLAUNCH filecheck ##
  #############################

  find_package(roslaunch REQUIRED)
  roslaunch_add_file_check(launch)

  ###########################
  ## Add Python unit tests ##
  ###########################

  catkin_add_nosetests(test)

  #############################
  ## Add ROSTEST executables ##
  #############################

  add_executable(diagnostics_rostest
    rostest/DiagnosticsRostest.cpp
  )
  add_dependencies(diagnostics_rostest ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
  target_link_libraries(diagnostics_rostest ${catkin_LIBRARIES} ${PACKAGE_CONFIGURATION_EXPORTED_LIBRARIES} rosen_abstract_node ${GTEST_LIBRARIES} pthread)

  add_executable(transitions_rostest
    rostest/TransitionsRostest.cpp
  )
  add_dependencies(transitions_rostest ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
  target_link_libraries(transitions_rostest ${catkin_LIBRARIES} ${PACKAGE_CONFIGURATION_EXPORTED_LIBRARIES} rosen_abstract_node ${GTEST_LIBRARIES} pthread)

  #######################
  ## Add ROSTEST tests ##
  #######################

  find_package(rostest REQUIRED)
  file(GLOB rostest_testfiles "rostest/*.test")
  foreach(rostest_testfile ${rostest_testfiles})
      add_rostest(${rostest_testfile})
  endforeach()

endif()
