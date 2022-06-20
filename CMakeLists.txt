project(semaphore)
cmake_minimum_required(VERSION 3.12)

find_program(INKSCAPE inkscape REQUIRED DOC "Path to inkscape")

option(WITH_LIVEKIT_VARIANT "Render Livekit variant of the theme" OFF)

message(STATUS "Check for working Inkscape: ${INKSCAPE}")
execute_process(COMMAND "${INKSCAPE}" --version TIMEOUT 3 RESULT_VARIABLE INKSCAPE_RESULT OUTPUT_VARIABLE INKSCAPE_VERSION_OUTPUT)
if (NOT INKSCAPE_RESULT EQUAL 0)
  message(FATAL_ERROR "Inkscape is not working properly.")
endif()
string(REGEX MATCH "Inkscape ([0-9\.]+)" INKSCAPE_VERSION "${INKSCAPE_VERSION_OUTPUT}")
set(INKSCAPE_VERSION "${CMAKE_MATCH_1}")
if (INKSCAPE_VERSION VERSION_LESS "1.1")
  message(FATAL_ERROR "Inkscape is too old! Needs at least 1.1, found ${INKSCAPE_VERSION}.")
endif()
message(STATUS "Found Inkscape ${INKSCAPE_VERSION}")

file(GLOB SOURCE_FILES "${CMAKE_SOURCE_DIR}/sequences/*/*.svg")

include(GNUInstallDirs)
set(ARTIFACTS)
set(SEMAPHORE_INSTALL_DIR "${CMAKE_INSTALL_DATADIR}/plymouth/themes/semaphore/")
set(LIVEKIT_INSTALL_DIR "${CMAKE_INSTALL_DATADIR}/plymouth/themes/livekit/")

foreach (F ${SOURCE_FILES})
  get_filename_component(FILENAME "${F}" NAME)
  get_filename_component(DIRNAME "${F}" DIRECTORY)
  get_filename_component(DIRNAME "${DIRNAME}" NAME)
  # skip livekit images if not requested
  if ((NOT WITH_LIVEKIT_VARIANT) AND (DIRNAME STREQUAL "livekit"))
    continue()
  endif()
  string(REPLACE ".svg" ".png" OUT "${FILENAME}")
  set(FILESLUG "${DIRNAME}-${OUT}")
  set(OUTPUT_FILE "${CMAKE_CURRENT_BINARY_DIR}/rendered/${FILESLUG}")
  list(APPEND ARTIFACTS "${OUTPUT_FILE}")
  add_custom_command(
    OUTPUT "${OUTPUT_FILE}"
    COMMAND "${INKSCAPE}" "${F}" -w 640 -D -o "${OUTPUT_FILE}"
    DEPENDS "${F}" COMMENT "Rendering ${FILESLUG} ..."
  )
  if (WITH_LIVEKIT_VARIANT)
    if (DIRNAME STREQUAL "livekit")
      install(
        FILES "${OUTPUT_FILE}"
        DESTINATION "${LIVEKIT_INSTALL_DIR}"
        RENAME "startup-${OUT}"
      )
      continue()
    elseif (DIRNAME STREQUAL "startup")
      # does nothing
    else ()
      install(FILES "${OUTPUT_FILE}" DESTINATION "${LIVEKIT_INSTALL_DIR}")
    endif()
  endif()
  install(FILES "${OUTPUT_FILE}" DESTINATION "${SEMAPHORE_INSTALL_DIR}")
endforeach()

add_custom_target(render ALL DEPENDS ${ARTIFACTS})

# install plymouth data
install(
  FILES "${CMAKE_SOURCE_DIR}/data/semaphore.plymouth" "${CMAKE_SOURCE_DIR}/data/semaphore.script"
  DESTINATION "${SEMAPHORE_INSTALL_DIR}"
)
if (WITH_LIVEKIT_VARIANT)
  install(
    FILES "${CMAKE_SOURCE_DIR}/data/livekit.plymouth" "${CMAKE_SOURCE_DIR}/data/livekit.script"
    DESTINATION "${LIVEKIT_INSTALL_DIR}"
  )
endif()
