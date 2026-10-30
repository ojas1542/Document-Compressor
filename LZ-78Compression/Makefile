CC=clang
CFLAGS=-std=c99 -Wall -Werror -Wextra -Wpedantic
LFLAGS=-lm

# Build native arm64 binaries on Apple Silicon, even from a Rosetta shell
ifeq ($(shell uname -s),Darwin)
ifeq ($(shell sysctl -n hw.optional.arm64 2>/dev/null),1)
CFLAGS += -arch arm64
endif
endif

SOURCES_ENCODE = encode.c bv.c io.c trie.c word.c
SOURCES_DECODE = decode.c bv.c io.c trie.c word.c
OBJECTS_E = $(SOURCES_ENCODE:%.c=%.o)
OBJECTS_D = $(SOURCES_DECODE:%.c=%.o)

.PHONY: all
all : encode decode
encode: $(OBJECTS_E)
	$(CC) $(CFLAGS) -o encode $(OBJECTS_E) $(LFLAGS)
decode	: $(OBJECTS_D) 
	$(CC) $(CFLAGS) -o decode $(OBJECTS_D) $(LFLAGS)
encode.o : encode.c
	$(CC) $(CFLAGS) -c encode.c
decode.o : decode.c
	$(CC) $(CFLAGS) -c decode.c
bv.o : bv.c
	$(CC) $(CFLAGS) -c bv.c
io.o : io.c
	$(CC) $(CFLAGS) -c io.c
trie.o : trie.c
	$(CC) $(CFLAGS) -c trie.c
word.o : word.c
	$(CC) $(CFLAGS) -c word.c
format  :
	clang-format -i *.[ch]
clean	:
	rm -rf encode decode encode.o decode.o io.o trie.o bv.o word.o infer-out
infer	:
	make clean; infer-capture -- make; infer-analyze -- make
