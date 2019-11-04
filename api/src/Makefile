ARCH := $(shell uname -m)

NO_RESET ?= no
NO_BOOT ?= no
WITH_TIZEN ?= no

LIB_NAME:= libmvnc
OUT := $(LIB_NAME).so.0
OBJDIR := obj-$(ARCH)
PREFIX ?= /usr/local
INSTALLDIR ?= ${DESTDIR}${PREFIX}
LIBDIR_NAME ?= lib
INCDIR_NAME ?= include
INCLUDE_LIBUSB := $(shell pkg-config libusb-1.0 --cflags)
LIBS_LIBUSB := $(shell pkg-config libusb-1.0 --libs)
INSTALL_LIBDIR := ${INSTALLDIR}/${LIBDIR_NAME}
INSTALL_INCDIR := ${INSTALLDIR}/${INCDIR_NAME}
INSTALL_PKGCONFIGDIR := ${INSTALLDIR}/${LIBDIR_NAME}/pkgconfig
PYTHON3DIST := $(shell python3 -c "import site; print(site.getsitepackages()[0])")
PYTHON2DIST := $(shell python -c "import site; print(site.getsitepackages()[0])")
LINK_HIGHCLASS ?= no
VERSION_NUMBER=$(shell cat ./version.txt)
SRCS := \
	mvnc_api.c \
	fp16.c


MV_COMMON_BASE=$(realpath ./common)
XLINK_BASE=$(MV_COMMON_BASE)/components/XLink
XLINKCONSOLE_BASE=$(MV_COMMON_BASE)/components/XLinkConsole

XLINK_SRCS=$(XLINK_BASE)/pc/usb_boot.c
XLINK_SRCS+=$(XLINK_BASE)/pc/pcie_host.c
XLINK_SRCS+=$(XLINK_BASE)/shared/XLink.c
XLINK_SRCS+=$(XLINK_BASE)/shared/XLinkDispatcher.c
XLINK_SRCS+=$(XLINKCONSOLE_BASE)/pc/XLinkConsole.c
XLINK_SRCS+=$(XLINK_BASE)/pc/XLinkPlatform.c

XLINK_CFLAGS= -I$(XLINK_BASE)/shared \
			-I$(XLINK_BASE)/pc \
			-I$(XLINKCONSOLE_BASE)/pc \
			-I $(MV_COMMON_BASE)/swCommon/include/ \
			-I $(MV_COMMON_BASE)/shared/include/

CFLAGS+=$(XLINK_CFLAGS) -D__PC__ -DUSE_USB_VSC -DVERSION_NAME="\"$(VERSION_NUMBER)\""
ifeq ($(NO_BOOT), yes)
    CFLAGS+=-DXLINK_NO_BOOT
endif
ifeq ($(NO_RESET), yes)
    CFLAGS+=-DXLINK_NO_RESET
endif
CFLAGS+=-DDEVICE_SHELL_ENABLED

LIBS += -lpthread -ldl

ifndef LIBS_LIBUSB
	INCLUDES += -lusb-1.0
else
	INCLUDES += ${LIBS_LIBUSB}
endif

#Allow compiling without highclass
ifeq ($(LINK_HIGHCLASS),no)
CFLAGS+= -DEXCLUDE_HIGHCLASS
else
LIBS+= -lmvnc_highclass -L$(INSTALLDIR)/${LIBDIR_NAME}/
endif
INCLUDES := \
	-I. \
	-I../include

ifndef INCLUDE_LIBUSB
	INCLUDES += -I$(SYSROOT)/usr/include/libusb-1.0
else
	INCLUDES += ${INCLUDE_LIBUSB}
endif


CFLAGS += -O2 -Wall -pthread -fPIC -MMD -MP
LDFLAGS += -shared

OBJS := $(SRCS:%.c=$(OBJDIR)/%.o)
OBJS += $(XLINK_SRCS:%.c=$(OBJDIR)/%.o)
DEPS := $(OBJS:.o=.d)

.PHONY: get_mvcmd libmvnc_pkgconfig

all: $(OBJDIR)/$(OUT) get_mvcmd

$(OBJDIR)/$(OUT): $(OBJS)
	if [ ! -e ./version.txt ] ; then echo "missing version.txt file"; exit 1; fi;
	$(CC) $(LDFLAGS) $(OBJS) -o $@ $(LIBS)
	ln -fs $(OBJDIR)/$(OUT) $(LIB_NAME).so
	ln -fs $(OBJDIR)/$(OUT) $(OUT)

$(OBJDIR)/%.o: %.c
	@mkdir -p $(dir $@)
	$(CC) $(CFLAGS) $(INCLUDES) -c $< -o $@

-include $(DEPS)

libmvnc_pkgconfig:
	rm -f libmvnc.pc.tmp
	cp -f libmvnc.pc.in libmvnc.pc.tmp
	sed -i 's|@VERSION@|$(VERSION_NUMBER)|g' libmvnc.pc.tmp
	sed -i 's|@LIB_INSTALL_DIR@|${PREFIX}/${LIBDIR_NAME}|g' libmvnc.pc.tmp
	sed -i 's|@INCLUDE_INSTALL_DIR@|${PREFIX}/${INCDIR_NAME}|g' libmvnc.pc.tmp
	mkdir -p ${INSTALL_PKGCONFIGDIR}
	install -m 644 libmvnc.pc.tmp ${INSTALL_PKGCONFIGDIR}/libmvnc.pc

get_mvcmd:
ifeq ($(strip $(WITH_TIZEN)), no)
	@./get_mvcmd.sh
endif

basicinstall: $(OBJDIR)/$(OUT) get_mvcmd libmvnc_pkgconfig
	mkdir -p ${INSTALL_INCDIR}
	mkdir -p ${INSTALL_INCDIR}/mvnc2
	mkdir -p ${INSTALL_LIBDIR}
	cp $(OBJDIR)/$(OUT) ${INSTALL_LIBDIR}/
	ln -fs $(OUT) ${INSTALL_LIBDIR}/$(LIB_NAME).so
	cp ../include/mvnc.h ${INSTALL_INCDIR}/mvnc2
	cd ${INSTALL_INCDIR} && ln -fs mvnc2/mvnc.h mvnc.h
	mkdir -p ${INSTALL_LIBDIR}/mvnc
	cp mvnc/MvNCAPI-*.mvcmd ${INSTALL_LIBDIR}/mvnc/
	mkdir -p ${DESTDIR}/etc/udev/rules.d/
	cp 97-usbboot.rules ${DESTDIR}/etc/udev/rules.d/

pythoninstall:
	mkdir -p ${DESTDIR}$(PYTHON3DIST)
	mkdir -p ${DESTDIR}$(PYTHON2DIST)
	cp -r ../python/mvnc ${DESTDIR}$(PYTHON3DIST)/
	cp -r ../python/mvnc ${DESTDIR}$(PYTHON2DIST)/

postinstall:
	udevadm control --reload-rules
	udevadm trigger
	ldconfig

install: basicinstall pythoninstall postinstall

uninstall:
	rm -f ${INSTALL_LIBDIR}/$(OUT)
	rm -f ${INSTALL_LIBDIR}/$(LIB_NAME).so
	rm -f ${INSTALL_INCDIR}/mvnc.h
	rm -f ${INSTALL_INCDIR}/mvnc2/mvnc.h
	rm -f ${INSTALL_LIBDIR}/mvnc/MvNCAPI-*.mvcmd
	rm -rf ${DESTDIR}$(PYTHON3DIST)/mvnc
	rm -rf ${DESTDIR}$(PYTHON2DIST)/mvnc
	rm -f ${DESTDIR}/etc/udev/rules.d/97-usbboot.rules

clean:
	rm -f $(OUT)
	rm -f $(OBJS)
	rm -rf $(OBJDIR)
	rm -f $(LIB_NAME).so
	if [ -d "mvnc" ]; then find mvnc/ -type f -delete ; fi
	find -name "libmvnc.pc.tmp" -delete
	find -name "libmvnc.pc" -delete
