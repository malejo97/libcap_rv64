
topdir=$(shell pwd)/..
include $(topdir)/Make.Rules
#
# Programs: all of the examples that we will compile
#
PROGS=getpcaps getcap setcap

BUILD=$(PROGS)

all: $(BUILD) capsh

ifeq ($(DYNAMIC),yes)
LDPATH = LD_LIBRARY_PATH=../libcap
DEPS = ../libcap/libcap.so
else
# For this build variant override the LDFLAGS to link statically from
# libraries within the build tree. If you never want this, use
# make DYNAMIC=yes ...
LDFLAGS = --static
DEPS = ../libcap/libcap.a
endif

../libcap/libcap.a:
	$(MAKE) -C ../libcap libcap.a

../libcap/libcap.so:
	$(MAKE) -C ../libcap libcap.so

$(BUILD): %: %.o $(DEPS)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $< $(LIBCAPLIB)

%.o: %.c $(INCS)
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

install: all
	mkdir -p -m 0755 $(FAKEROOT)$(SBINDIR)
	for p in $(PROGS) capsh ; do \
		install -m 0755 $$p $(FAKEROOT)$(SBINDIR) ; \
	done
ifeq ($(RAISE_SETFCAP),yes)
	$(FAKEROOT)$(SBINDIR)/setcap cap_setfcap=i $(FAKEROOT)$(SBINDIR)/setcap
endif

test:
	@echo "no program tests without privilege, try 'make sudotest'"

capshdoc.h.cf: capshdoc.h ./mkcapshdoc.sh
	./mkcapshdoc.sh > $@
	diff -u capshdoc.h $@ || (rm $@ ; exit 1)

capsh: capsh.c capshdoc.h.cf $(DEPS)
	$(CC) $(CFLAGS) $(CPPFLAGS) $(CAPSH_SHELL) $(LDFLAGS) -o $@ $< $(LIBCAPLIB)

# Statically linked with minimal linkage flags to enable running in a
# chroot and in other in-tree testing contexts.
tcapsh-static: capsh.c capshdoc.h.cf $(DEPS)
	$(CC) $(CFLAGS) $(CPPFLAGS) $(CAPSH_SHELL) -o $@ $< $(LIBCAPLIB) --static

uns_test: ../tests/uns_test.c
	$(MAKE) -C ../tests uns_test
	cp ../tests/uns_test .

sudotest: tcapsh-static uns_test capsh setcap getcap getpcaps tcapsh-static
	$(SUDO) $(LDPATH) ./quicktest.sh

clean:
	$(LOCALCLEAN)
	rm -f *.o $(BUILD) privileged ping hack.sh compare-cap uns_test
	rm -f capsh tcapsh* capshdoc.h.cf
