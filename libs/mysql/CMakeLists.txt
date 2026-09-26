add_library(mysql.hdll
    socket.c
    sha1.c
    my_proto.c
    my_api.c
    mysql.c
)

set_as_hdll(mysql)

target_link_libraries(mysql.hdll
    libhl
)

if(WIN32)
    target_compile_definitions(mysql.hdll PRIVATE _CRT_SECURE_NO_WARNINGS)
    target_link_libraries(mysql.hdll ws2_32)
endif()

install(
    TARGETS
        mysql.hdll
    DESTINATION ${HDLL_DESTINATION}
)
