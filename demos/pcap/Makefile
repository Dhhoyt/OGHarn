CC_FUZZ       	= afl-clang-fast
CXX_FUZZ      	= afl-clang-fast++
CFLAGS_ASAN 	= -fsanitize=address,undefined
CXXFLAGS_ASAN 	= -fsanitize=address,undefined

LIB_NAME = lib
LIB_REPO = https://github.com/the-tcpdump-group/libpcap.git

DEPS = -I $(PWD)/$(LIB_NAME)/ \
	   -L $(PWD)/$(LIB_NAME)/ \
	   -L $(PWD)/$(LIB_NAME)/systemd/build

DEPS_DYN = -lpcap -ldbus-1 -libverbs -lnl-3 -lnl-genl-3 -lnl-route-3 -lnl-genl-3
DEPS_STC = -l:libpcap.a -ldbus-1 -l:libsystemd.a -libverbs -lnl-3 -lnl-genl-3 -lnl-route-3 -lcap -lnl-genl-3
DEPS_LDD = $(PWD)/$(LIB_NAME)/

#------------------------------------------------------------------------
lib: # build for harness generation. Dynamic linking, asan, and afl instrumentation
	export AFL_USE_ASAN=1
	export AFL_USE_UBSAN=1
	rm -rf $(LIB_NAME)
	git clone $(LIB_REPO) $(LIB_NAME)
	cd $(LIB_NAME) && \
	git checkout e17fe06d6a54abc85fb17998d0cb1742d490382a && \
	./autogen.sh && ./configure \
	CC=$(CC_FUZZ) CXX=$(CC_FUZZ) && \
	make all -j12

lib_plain: # build for indexing with multiplier
	export AFL_USE_ASAN=1
	export AFL_USE_UBSAN=1
	rm -rf $(LIB_NAME)_plain
	git clone $(LIB_REPO) $(LIB_NAME)_plain
	cd $(LIB_NAME)_plain && \
	git checkout e17fe06d6a54abc85fb17998d0cb1742d490382a && \
	./autogen.sh && ./configure \
	CC=$(CC) CXX=$(CXX) && \
	bear -- make all -j12

lib_fuzz: # build for fuzzing. Static linking with afl instrumentation
	rm -rf $(LIB_NAME)_fuzz
	git clone $(LIB_REPO) $(LIB_NAME)_fuzz
	cd $(LIB_NAME)_fuzz && \
	git checkout e17fe06d6a54abc85fb17998d0cb1742d490382a && \
	git clone https://github.com/systemd/systemd.git && \
	cd systemd && \
	meson build --buildtype=release -Dstatic-libsystemd=true && \
	cd build && ninja && cd ../.. && \
	./autogen.sh && ./configure \
	CC=$(CC_FUZZ) CXX=$(CXX_FUZZ) --enable-static && \
	make all -j12

run_mx:  # Use Multiplier to index the library
	mx-index --db $(PWD)/$(LIB_NAME).db --target $(PWD)/$(LIB_NAME)_plain/compile_commands.json --workspace $(PWD)/mx
	
#------------------------------------------------------------------------
all: lib lib_plain lib_fuzz run_mx
	
#------------------------------------------------------------------------
harness: # make command used to make the harness during generation
	$(CC_FUZZ) -o $(OUT)/harness.out $(OUT)/harness.c $(DEPS) $(DEPS_DYN) $(CFLAGS_ASAN)

showmap: # command used to get coverage information about library under test
	LD_LIBRARY_PATH=$(DEPS_LDD) afl-showmap -o $(OUT)/tempfile -- $(OUT)/harness.out $(SEED)

harness_fuzz: # compile a harness specifically for fuzzing. Pass the harness number using make harness_fuzz HARNESS_NUMBER=x
	@ls bin || mkdir bin 
	$(eval LIB_NAME=lib_fuzz) \
	$(CC_FUZZ) -o bin/ogharn$(HARNESS_NUMBER)\_fuzz $(OUT)/final-harnesses/src/harness$(HARNESS_NUMBER):*.c -static $(DEPS) $(DEPS_STC) ;

clean:
	rm -rf bin mx lib.db-*

clean_lib:
	rm -rf $(LIB_NAME)* 