cmake_minimum_required(VERSION 2.8)
include(parse_arguments.cmake)

set(GENCPP_EXE /home/morten/gencpp/scripts/genmsg_cpp.py)

set(TOP /home/morten/genmsg/test/files)

macro(show VAR)
  message(STATUS ">>>> ${VAR} ${${VAR}}")
endmacro()

macro(generate PKG)
  parse_arguments(PKG "MESSAGES;DEPENDENCIES" "" ${ARGN}) 
  
  message("PKG: ${PKG} MSGS: ${PKG_MESSAGES}  DEPNDS: ${PKG_DEPENDENCIES}")
  set(ALL_OUTPUTS "")
  foreach(MSGPATH ${PKG_MESSAGES}) 

    get_filename_component(SHORTNAME ${MSGPATH} NAME_WE)

    configure_file(${TOP}/${PKG}/msg/${MSGPATH} 
      ${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${MSGPATH}.stamp
      )

    set(GENERATED_NAME ${PKG}/${SHORTNAME}.h)
    list(APPEND GENERATED_ALL ${GENERATED_NAME})

    set(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${PKG}/${SHORTNAME}.h)
    message(STATUS "Output=${OUTPUT}")

     set(IFLAGS "")
     set(DEP_TARGETS "")

     foreach(dep ${PKG_DEPENDENCIES})
       set(IFLAGS -I${dep}:${TOP}/${dep}/msg ${IFLAGS})
       list(APPEND DEP_TARGETS ${dep}_gencpp)
     endforeach()
 
     set(INPUT_FULLPATH ${TOP}/${PKG}/msg/${MSGPATH})
     set(THESEDEPS "")
     execute_process(COMMAND
       ${GENCPP_EXE} ${INPUT_FULLPATH} -d ${IFLAGS}
       OUTPUT_VARIABLE THESEDEPS
       OUTPUT_STRIP_TRAILING_WHITESPACE
       )
     separate_arguments(THESEDEPS UNIX_COMMAND ${THESEDEPS})
     show(THESEDEPS)

     add_custom_command(OUTPUT ${OUTPUT}
       DEPENDS ${MSG} ${THESEDEPS} ${GENCPP_EXE}
       COMMAND ${GENCPP_EXE} ${INPUT_FULLPATH}
       -p ${PKG}
       -o ${CMAKE_CURRENT_BINARY_DIR}/${PKG}
       ${IFLAGS}
       COMMENT "Generating C++ code from ${PKG}/${SHORTNAME}"
       )
     list(APPEND ALL_OUTPUTS ${OUTPUT})

  endforeach()
  show(ALL_OUTPUTS)
  add_custom_target(${PKG}_gencpp
    DEPENDS ${ALL_OUTPUTS}
    )

  show(DEP_TARGETS)
  add_dependencies(${PKG}_gencpp "${DEP_TARGETS}")
endmacro()


add_custom_target(touch
  COMMAND touch ${TOP}/*/msg/*.msg
  COMMENT TOUCHY TOUCHY
  )


generate(std_msgs
  MESSAGES Header.msg
  )
generate(geometry_msgs
  MESSAGES Quaternion.msg Vector3.msg
  DEPENDENCIES std_msgs
  )
generate(sensor_msgs
  MESSAGES Imu.msg
  DEPENDENCIES geometry_msgs std_msgs
  )

# generate(std_msgs 
#   /home/morten/genmsg/test/files/std_msgs/msg/Header.msg)
# 
# generate(geometry_msgs 
#   /home/morten/genmsg/test/files/geometry_msgs/msg/Quaternion.msg
#   std_msgs
#   )
# 
# generate(geometry_msgs 
#   /home/morten/genmsg/test/files/geometry_msgs/msg/Vector3.msg
#   std_msgs
#   )
# 
# generate(sensor_msgs 
#   /home/morten/genmsg/test/files/sensor_msgs/msg/Imu.msg
#   std_msgs
#   geometry_msgs)

include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(
  /opt/ros/electric/stacks/ros_comm/utilities/cpp_common/include/
  /opt/ros/electric/stacks/ros_comm/utilities/rostime/include/
  /opt/ros/electric/stacks/ros_comm/clients/cpp/roscpp_serialization/include/
  /opt/ros/electric/stacks/ros_comm/clients/cpp/roscpp_traits/include/)

# add_custom_target(generate_cpp
#    DEPENDS 
#    sensor_msgs/Imu.h
#    std_msgs/Header.h
#    geometry_msgs/Quaternion.h
#    geometry_msgs/Vector3.h
#  )

add_executable(main main.cpp)
add_executable(main2 main.cpp)

add_dependencies(main sensor_msgs_gencpp)
#add_dependencies(main2 generate_cpp)

# use this to provoke removal of generated headers
set_directory_properties(PROPERTIES
  CLEAN_NO_CUSTOM FALSE
  )

set_source_files_properties(
  std_msgs/Header.h 
  geometry_msgs/Vector3.h
  geometry_msgs/Quaternion.h
  PROPERTIES
  GENERATED TRUE)