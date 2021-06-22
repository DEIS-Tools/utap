FILE(GLOB utap_source "*.c" "*.cpp" "*.h")

# message(STATUS "found utap sources: " ${utap_source})
find_package(FLEX 2.6.4 REQUIRED)
find_package(BISON 3.0.4 REQUIRED)

find_program(GPERF gperf)
if(NOT GPERF)
    message(FATAL_ERROR "Could not find gperf executable")
endif()

set(keywords_source "${CMAKE_CURRENT_SOURCE_DIR}/keywords.cc") # in SOURCE_DIR because included by parser.cc
add_custom_command(OUTPUT ${keywords_source}
  DEPENDS keywords.gperf
  COMMAND ${GPERF} -C -E -t -LC++ -c -ZKeywords -k "\\*" "${CMAKE_CURRENT_SOURCE_DIR}/keywords.gperf" "--output-file=${keywords_source}"
)
add_custom_target(keywords_generate DEPENDS ${keywords_source})

set(tags_source "${CMAKE_CURRENT_SOURCE_DIR}/tags.cc") # in SOURCE_DIR because included by xmlreader.cpp
add_custom_command(OUTPUT "${tags_source}"
  DEPENDS tags.gperf
  COMMAND ${GPERF} -C -E -t -LC++ -c -Kstr -ZTags "${CMAKE_CURRENT_SOURCE_DIR}/tags.gperf" "--output-file=${tags_source}"
)
add_custom_target(tags_generate DEPENDS "${tags_source}")

set(lexer_source "${CMAKE_CURRENT_SOURCE_DIR}/lexer.cc") # in SOURCE_DIR due to libparser.h inclusion
add_custom_command(OUTPUT "${lexer_source}"
  DEPENDS keywords_generate tags_generate lexer.ll
  COMMAND ${FLEX_EXECUTABLE} "--outfile=${lexer_source}" -Putap_ "${CMAKE_CURRENT_SOURCE_DIR}/lexer.ll"
)
add_custom_target(lexer_generate DEPENDS "${lexer_source}")

set(parser_source "${CMAKE_CURRENT_SOURCE_DIR}/parser.cc") # in SOURCE_DIR due to libparser.h inclusion
add_custom_command(OUTPUT "${parser_source}"
  DEPENDS lexer_generate parser.yy
  COMMAND ${BISON_EXECUTABLE} -putap_ -bparser "${CMAKE_CURRENT_SOURCE_DIR}/parser.yy" "--output=${parser_source}"
)
add_custom_target(parser_generate DEPENDS "${parser_source}")

if(UNIX)
  set(UTAP_SYSTEM_LIBRARIES dl)
endif(UNIX)

find_package(Boost 1.65 REQUIRED COMPONENTS headers REQUIRED)
message(STATUS "Boost_INCLUDE_DIRS: ${Boost_INCLUDE_DIRS}")
#message(STATUS "Boost libraries: ${Boost_LIBRARIES}")

find_package(libxml2 NAMES LibXml2)
if(libxml2_FOUND)
	message(STATUS "Found libxml2: ${LIBXML2_INCLUDE_DIRS}")
	set(LIBXML2_INCLUDE_DIR ${LIBXML2_INCLUDE_DIRS})
else(libxml2_FOUND)
	set(LIBXML2_INCLUDE_DIR "/usr/include/libxml2")
	set(LIBXML2_LIBRARIES "xml2")
    message(STATUS "Could NOT find libxml2, instead using: ${LIBXML2_INCLUDE_DIR}")
endif(libxml2_FOUND)

set(UTAP_INCLUDES "${CMAKE_CURRENT_SOURCE_DIR} ${LIBXML2_INCLUDE_DIR} ${Boost_INCLUDE_DIRS}")

add_library(utap ${utap_source} ${parser_source})
target_include_directories(utap PRIVATE ${UTAP_INCLUDES})
add_dependencies(utap parser_generate)
target_link_libraries(utap ${LIBXML2_LIBRARIES} ${UTAP_SYSTEM_LIBRARIES})

add_executable(pretty pretty.cpp)
target_include_directories(pretty PRIVATE ${UTAP_INCLUDES})
target_link_libraries(pretty PRIVATE ${libs})

add_executable(syntaxcheck syntaxcheck.cpp)
target_include_directories(syntaxcheck PRIVATE ${UTAP_INCLUDES})
target_link_libraries(syntaxcheck PRIVATE ${libs})

add_executable(taflow taflow.cpp)
target_include_directories(taflow PRIVATE ${UTAP_INCLUDES})
target_link_libraries(taflow PRIVATE ${libs})
