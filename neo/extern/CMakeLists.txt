project(external)

set(src_external)
set(src_external_includes)

file(GLOB src_external_miniz miniz/*.c miniz/*.h)
source_group("Miniz" FILES ${src_external_miniz})
list(APPEND src_external ${src_external_miniz})
list(APPEND src_external_includes ${CMAKE_CURRENT_SOURCE_DIR}/miniz)
#always_optimize_sourcefile(miniz/miniz.c)

add_library(${PROJECT_NAME} STATIC ${src_external})
target_compile_definitions(${PROJECT_NAME} PRIVATE GLEW_STATIC)
target_include_directories(${PROJECT_NAME} PRIVATE ${src_external_includes})
set_target_properties(${PROJECT_NAME} PROPERTIES FOLDER "libs")
