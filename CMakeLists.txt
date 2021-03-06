set(mbusd_DIST_dir ${COMPONENT_DIR}/mbusd)

set(mbusd_SRC_DIR ${mbusd_DIST_dir}/src)

file(READ ${mbusd_SRC_DIR}/tty.c tty_c)
# disable CRTSCTS flags, not supported!
string(REPLACE
    "mod->tios.c_cflag &= ~(CSIZE | CSTOPB | PARENB | PARODD | CRTSCTS);"
    "mod->tios.c_cflag &= ~(CSIZE | CSTOPB | PARENB | PARODD /*| CRTSCTS*/);"
     tty_c "${tty_c}")

# disable rts control, API not supported, move control to user code
string(REGEX REPLACE "void
tty_set_rts\\(int fd\\)
{(.*)}
.*
tty_clr_rts\\(int fd\\)
{(.*)}
#endif"

" /* DISABLED */
void tty_set_rts(int fd){\\n/*\\1 */ \\n}
void tty_clr_rts(int fd){\\n/*\\2 */ \\n}
#endif"
    tty_c "${tty_c}")

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/tty.c "${tty_c}")

file(READ ${mbusd_SRC_DIR}/sock.c sock_c)

string(REPLACE
    "fcntl(server_s, F_SETFD, 1) == -1"
    "0 /* DISABLED */"
     sock_c "${sock_c}")

# setting lwip TX buffer size not implemented
# see http://lwip.100.n7.nabble.com/Setting-TCP-socket-s-send-amp-receive-buffer-size-td10263.html
string(REPLACE
"(setsockopt(server_s, SOL_SOCKET,
                  SO_SNDBUF, (void *)&sock_opt,
\t\t          sizeof(sock_opt)) == -1) ||
      (setsockopt(server_s, SOL_SOCKET,
                  SO_RCVBUF, (void *)&sock_opt,
\t\t          sizeof(sock_opt)) == -1)"
    "(0 /* DISABLED */) ||
      (0 /* DISABLED */)"
    sock_c "${sock_c}")

string(REPLACE
"(setsockopt(sd, SOL_SOCKET,
                  SO_SNDBUF, (void *)&sock_opt,
\t\t          sizeof(sock_opt)) == -1) ||
      (setsockopt(sd, SOL_SOCKET,
                  SO_RCVBUF, (void *)&sock_opt,
\t\t          sizeof(sock_opt)) == -1)"
    "(0 /* DISABLED */) ||
      (0 /* DISABLED */)"
    sock_c "${sock_c}")

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/sock.c "${sock_c}")


file(READ ${mbusd_SRC_DIR}/conn.c conn_c)
# wrap select to my_select()
string(REPLACE
    "select(max_sd + 1, &sdsetrd, &sdsetwr, NULL, &t_out);"
    "my_select(max_sd + 1, &sdsetrd, &sdsetwr, NULL, &t_out, tty.fd);"
    conn_c "${conn_c}")

string(REGEX REPLACE
    "(#include \"state\\.h\")"

    "\\1

extern int my_select(int __nfds, fd_set *__restrict __readfds,
    fd_set *__restrict __writefds, fd_set *__restrict __exceptfds,
    struct timeval *__restrict __timeout, int skip_d);"
    conn_c "${conn_c}")

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/conn.c "${conn_c}")

set(mbusd_SRC
    ${mbusd_SRC_DIR}/cfg.c
    ${CMAKE_CURRENT_BINARY_DIR}/conn.c
    ${mbusd_SRC_DIR}/crc16.c
    ${mbusd_SRC_DIR}/modbus.c
    ${mbusd_SRC_DIR}/queue.c
    ${CMAKE_CURRENT_BINARY_DIR}/sock.c
    ${mbusd_SRC_DIR}/state.c
    ${CMAKE_CURRENT_BINARY_DIR}/tty.c

    esp_log.c
    my_select.c
    mbGateway.c
    sig.c
)

idf_component_register(SRCS ${mbusd_SRC}
        INCLUDE_DIRS ${mbusd_SRC_DIR} ${COMPONENT_DIR}
        REQUIRES log esp32 freertos
)

target_compile_definitions(${COMPONENT_LIB}
    PUBLIC
        -DTRXCTL
        -DLOG
    PRIVATE
        -DDEBUG
)

target_compile_options(${COMPONENT_LIB}
    PRIVATE
        -Wno-error=maybe-uninitialized
)
