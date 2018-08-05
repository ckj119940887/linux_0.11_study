obj-m := hello.o

KDIR:=/lib/modules/`uname -r`/build
PWD:=$(shell pwd)

all:
	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD)

clean:
	rm -rf *.o *.ko *.mod.* *.symvers *.order
