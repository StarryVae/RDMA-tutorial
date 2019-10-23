.PHONY: clean

CFLAGS  := -Wall -g
LD      := gcc
LDLIBS  := ${LDLIBS} -lrdmacm -libverbs -lpthread

APPS    := server client

all: ${APPS}

client: client.o
        ${LD} -o $@ $^ ${LDLIBS}

server: server.o
        ${LD} -o $@ $^ ${LDLIBS}


clean:
        rm -f ${APPS}