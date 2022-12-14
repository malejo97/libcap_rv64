# Building the libcap/{cap.psx} Go packages, and examples.
#
# Note, we use symlinks to construct a go.mod build friendly tree. The
# packages themselves are intended to be (ultimately) found via proxy
# as "kernel.org/pub/linux/libs/security/libcap/cap" and
# "kernel.org/pub/linux/libs/security/libcap/psx". However, to
# validate their use on these paths, we fake such a structure in the
# build tree with symlinks and a vendor directory.

topdir=$(realpath ..)
include $(topdir)/Make.Rules

IMPORTDIR=kernel.org/pub/linux/libs/security/libcap
PKGDIR=pkg/$(GOOSARCH)/$(IMPORTDIR)

DEPS=../libcap/libcap.a ../libcap/libpsx.a
TESTS=compare-cap try-launching psx-signals

all: PSXGOPACKAGE CAPGOPACKAGE web setid gowns captree

$(DEPS):
	$(MAKE) -C ../libcap all

../progs/tcapsh-static:
	$(MAKE) -C ../progs tcapsh-static

vendor/$(IMPORTDIR) vendor/modules.txt:
	mkdir -p "vendor/$(IMPORTDIR)"
	echo "# $(IMPORTDIR)/psx v$(GOMAJOR).$(VERSION).$(MINOR)" > vendor/modules.txt
	echo "$(IMPORTDIR)/psx" >> vendor/modules.txt
	echo "# $(IMPORTDIR)/cap v$(GOMAJOR).$(VERSION).$(MINOR)" >> vendor/modules.txt
	echo "$(IMPORTDIR)/cap" >> vendor/modules.txt

vendor/$(IMPORTDIR)/psx: vendor/modules.txt
	ln -sf $(topdir)/psx vendor/$(IMPORTDIR)
	touch ../psx

vendor/$(IMPORTDIR)/cap: vendor/modules.txt
	ln -sf $(topdir)/cap vendor/$(IMPORTDIR)
	touch ../cap

$(topdir)/libcap/cap_names.h:
	$(MAKE) -C $(topdir)/libcap cap_names.h

good-names.go: $(topdir)/libcap/cap_names.h vendor/$(IMPORTDIR)/cap mknames.go
	CC="$(CC)" $(GO) run -mod=vendor mknames.go --header=$< --textdir=$(topdir)/doc/values | gofmt > $@ || rm -f $@
	diff -u ../cap/names.go $@

PSXGOPACKAGE: vendor/$(IMPORTDIR)/psx ../psx/*.go $(DEPS)
	touch $@

CAPGOPACKAGE: vendor/$(IMPORTDIR)/cap ../cap/*.go good-names.go $(PSXGOPACKAGE)
	touch $@

# Compiles something with this package to compare it to libcap. This
# tests more when run under sudotest (see ../progs/quicktest.sh for that).
compare-cap: compare-cap.go CAPGOPACKAGE
	CC="$(CC)" $(CGO_LDFLAGS_ALLOW) CGO_CFLAGS="$(CGO_CFLAGS)" CGO_LDFLAGS="$(CGO_LDFLAGS)" $(GO) build $(GO_BUILD_FLAGS) -mod=vendor $<

web: ../goapps/web/web.go CAPGOPACKAGE
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) $(GO) build $(GO_BUILD_FLAGS) -mod=vendor -o $@ $<
ifeq ($(RAISE_GO_FILECAP),yes)
	$(MAKE) -C ../progs setcap
	$(SUDO) ../progs/setcap cap_setpcap,cap_net_bind_service=p web
	@echo "NOTE: RAISED cap_setpcap,cap_net_bind_service ON web binary"
endif

setid: ../goapps/setid/setid.go CAPGOPACKAGE PSXGOPACKAGE
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) $(GO) build $(GO_BUILD_FLAGS) -mod=vendor -o $@ $<

gowns: ../goapps/gowns/gowns.go CAPGOPACKAGE
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) $(GO) build $(GO_BUILD_FLAGS) -mod=vendor -o $@ $<

captree: ../goapps/captree/captree.go CAPGOPACKAGE
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) $(GO) build $(GO_BUILD_FLAGS) -mod=vendor -o $@ $<

ok: ok.go
	CC="$(CC)" CGO_ENABLED=0 $(GO) build $(GO_BUILD_FLAGS) -mod=vendor $<

try-launching: try-launching.go CAPGOPACKAGE ok
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) $(GO) build $(GO_BUILD_FLAGS) -mod=vendor $<
ifeq ($(CGO_REQUIRED),0)
	CC="$(CC)" CGO_ENABLED="1" $(CGO_LDFLAGS_ALLOW) $(GO) build $(GO_BUILD_FLAGS) -mod=vendor -o $@-cgo $<
endif

psx-signals: psx-signals.go PSXGOPACKAGE
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) CGO_CFLAGS="$(CGO_CFLAGS)" CGO_LDFLAGS="$(CGO_LDFLAGS)" $(GO) build $(GO_BUILD_FLAGS) -mod=vendor $<

ifeq ($(CGO_REQUIRED),0)
psx-signals-cgo: psx-signals.go PSXGOPACKAGE
	CC="$(CC)" CGO_ENABLED="1" $(CGO_LDFLAGS_ALLOW) CGO_CFLAGS="$(CGO_CFLAGS)" CGO_LDFLAGS="$(CGO_LDFLAGS)" $(GO) build $(GO_BUILD_FLAGS) -mod=vendor -o $@ $<
endif

b210613: b210613.go CAPGOPACKAGE
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) CGO_CFLAGS="$(CGO_CFLAGS)" CGO_LDFLAGS="$(CGO_LDFLAGS)" $(GO) build $(GO_BUILD_FLAGS) -mod=vendor $<

test: setid gowns captree $(TESTS)
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) $(GO) test -mod=vendor $(IMPORTDIR)/psx
	CC="$(CC)" CGO_ENABLED="$(CGO_REQUIRED)" $(CGO_LDFLAGS_ALLOW) $(GO) test -mod=vendor $(IMPORTDIR)/cap
	LD_LIBRARY_PATH=../libcap ./compare-cap
	./psx-signals
ifeq ($(CGO_REQUIRED),0)
	$(MAKE) psx-signals-cgo
	./psx-signals-cgo
endif
	./setid --caps=false
	./gowns -- -c "echo gowns runs"
	./captree 0

# Note, the user namespace doesn't require sudo, but I wanted to avoid
# requiring that the hosting kernel supports user namespaces for the
# regular test case.
sudotest: test ../progs/tcapsh-static b210613
	../progs/tcapsh-static --has-b=cap_sys_admin || exit 0 && ./gowns --ns -- -c "echo gowns runs with user namespace"
	./try-launching
ifeq ($(CGO_REQUIRED),0)
	./try-launching-cgo
endif
	$(SUDO) ./try-launching
ifeq ($(CGO_REQUIRED),0)
	$(SUDO) ./try-launching-cgo
endif
	$(SUDO) ../progs/tcapsh-static --cap-uid=$$(id -u) --caps="cap_setpcap=ep" --iab="^cap_setpcap" -- -c ./b210613

# As of libcap-2.55 We stopped installing the cap and psx packages as
# part of the install.  Most distribution's packagers skip the Go
# builds, so it was not well used any way. The new hotness is to just
# use Go modules and download the packages from a tagged release in
# the git repository. For an example of how to do this from scratch:
#
#   https://sites.google.com/site/fullycapable/getting-started-with-go/building-go-programs-that-manipulate-capabilities
#
# For those brave souls that do include the Go build (testing) as part
# of their packaging, we reward them with a copy of the captree
# utility!
install: all
	mkdir -p -m 0755 $(FAKEROOT)$(SBINDIR)
	install -m 0755 captree $(FAKEROOT)$(SBINDIR)

clean:
	rm -f *.o *.so *~ mknames ok good-names.go
	rm -f web setid gowns captree
	rm -f compare-cap try-launching try-launching-cgo
	rm -f $(topdir)/cap/*~ $(topdir)/psx/*~
	rm -f b210613 psx-signals psx-signals-cgo
	rm -fr vendor CAPGOPACKAGE PSXGOPACKAGE go.sum
