cmake_minimum_required (VERSION 3.7.0) 
project(problem_43)

add_definitions(-DINSTALL=${LIBS_PATH}/date)
add_definitions(-DCURL_STATICLIB)

include_directories(${LIBS_PATH}/date/include/date)
include_directories(${LIBS_PATH}/date/include)
include_directories(${LIBS_PATH}/curl/include)
include_directories(${LIBS_PATH}/curlcpp/include)

add_executable(problem_43 ${LIBS_PATH}/date/src/tz.cpp main.cpp)

target_link_libraries(problem_43 libcurl)
target_link_libraries(problem_43 curlcpp)

if (UNIX)
link_directories(${CURLCPP_LIB_LOCATION})
elseif(WIN32)
target_link_libraries(problem_43 Crypt32.lib ws2_32.lib winmm.lib wldap32.lib)
endif()


set_target_properties (problem_43 PROPERTIES FOLDER "05.Datetime")
