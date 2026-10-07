obj-m += hello.o

KDIR ?= /home/ub22/mxxmstar/buildroot/buildroot-2025.02.15/output/build/linux-custom

all:
	$(MAKE) -C $(KDIR) M=$(CURDIR) ARCH=arm64 \
		CROSS_COMPILE=/home/ub22/mxxmstar/buildroot/buildroot-2025.02.15/output/host/bin/aarch64-buildroot-linux-gnu- \
		modules

clean:
	$(MAKE) -C $(KDIR) M=$(CURDIR) clean