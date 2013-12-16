# сборка приложений среды L4Re и L4Linux с поддержкой неисполнимой (Non-Executable, NX) памяти

## получение компилятора
На данный момент, сборка L4Re для архитектуры ARM с бинарным интерфейсом hard-float невозможна, поэтому необходимо использовать компилятор, собирающий двоичные файлы в формате softfp. Таким образом, большая часть компиляторов, по умолчанию доступная в новых версиях дистрибутивов Linux и сборка компилятора от Linaro несовместимы с L4Re. Кроме того, при вызове компоновщика в процессе сборки L4Re требуется наличие файла crtendS.o и других файлов CRT (библиотеки C Runtime), поэтому необходимо использовать компилятор, собранный под цель(target) "linux", а не под "none" (также называемый bare metal toolchain). Проверена корректная работа системы L4Re при использовании компилятора Sourcery Lite от Mentor.

https://sourcery.mentor.com/GNUToolchain/package11447/public/arm-none-linux-gnueabi/arm-2013.05-24-arm-none-linux-gnueabi-i686-pc-linux-gnu.tar.bz2

## Исправление системы сборки L4Re
По умолчанию среда L4Re не позволяет переопределить префикс компилятора, который используется при сборке. Поэтому, при использовании компилятора Sourcery необходимо внести изменение в программу сборки. В файл **L4Reap/l4/mk/Makeconf** необходимо заменить строку, в которой определена переменная `SYSTEM_TARGET_arm`.
```
./mk/Makeconf:SYSTEM_TARGET_arm     = arm-none-linux-gnueabi-
```

## подготовка каталога для сборки
Перейдите в каталог, в который вы хотите скачать исходный код и в котором будут созданы директории с двоичными файлами fiasco, L4Re и L4Linux.

> **Примечание:** В данном руководстве, такой директорией является `/mnt/disk2/l4_snapshot`.

Получите исходный код из следующего источника:
https://github.com/Ksys-labs/L4Reap
git clone htpps://github.com/Ksys-labs/l4linux

```bash
git clone https://github.com/Ksys-labs/L4Reap.git
git clone htpps://github.com/Ksys-labs/l4linux.git
```

Переключите репозиторий на ветку с реализацией поддержки NX памяти.
```bash
git checkout test-nx
```

Установите переменные среды, указывающие на директории для сброрки.
```bash
export FOC_ROOT="$PWD"
export FOC_PATH="${FOC_ROOT}/L4Reap"
export FOC_BUILDDIR="${FOC_ROOT}/build"
export L4RE_BUILDDIR="${FOC_ROOT}/build_l4re"
export L4LINUX_BUILDDIR="${FOC_ROOT}/build_l4linux"
export MAKE_OPTS="-j10"
```
Подготовьте директории для сборки.

Для микроядра fiasco
```bash
pushd .
	cd "${FOC_PATH}/kernel/fiasco/"
	make BUILDDIR="$FOC_BUILDDIR"
popd
```

Для среды L4Re
```
pushd .
	cd "${FOC_PATH}/l4"
	make B="$L4RE_BUILDDIR"
popd

```

## сборка L4Re
### сборка микроядра fiasco
Перейдите в каталог сборки fiasco, выполните `make config` и установите настройки архитектуры. Можно также скопировать файл `.config` из примера.
```bash
pushd .
	cd "$FOC_BUILDDIR"
	make config
	make "$MAKE_OPTS"
popd

```

Теперь можно приступить к настройке L4Re
```bash
pushd .
	cd "${FOC_PATH}/l4"
	make O="$L4RE_BUILDDIR" config
popd

```
### создание файла конфигурации Makeconf.boot
Каждый раз задавать переменные среды и путь (переменную PATH) неудобно. Вместо этого, можно прописать переменные и аргументы для запуска эмулятора QEMU в файле конфигурации. Он должен находиться в `conf/Makeconf.boot` в каталоге сборки L4Re. Можно использовать файл Makeconf.boot.example из примеров настройки L4Re в качестве шаблона.

```bash
pushd .
	cd "$L4RE_BUILDDIR"
	mkdir conf
	cp ../l4/conf/Makeconf.boot.example conf/Makeconf.boot
popd
```

При сборке модуля L4Re, может возникнуть необходимость добавить каталог с файлами конфигурации в переменную `MODULE_SEARCH_PATH`.

```make
MODULE_SEARCH_PATH += /mnt/disk2/l4_snapshot/build/:/mnt/disk2/l4_snapshot/L4Reap/l4/pkg/ksys_mem_tests/conf:/mnt/disk2/l4_snapshot/tudos/l4/pkg/io/config
qemu: MODULE_SEARCH_PATH += /mnt/disk2/l4_snapshot/build/:/mnt/disk2/l4_snapshot/L4Reap/l4/pkg/ksys_mem_tests/conf:/mnt/disk2/l4_snapshot/tudos/l4/pkg/io/config
```

Теперь, собрать все пакеты среды L4Re при помощи команды `make` в каталоге сборки. Также, можно собрать только нужные нам пакеты, перечислив их через аргумент `S=`. Иногда система сборки L4Re не может разрешить зависимости между пакетами, и сборка завершается неудачей. В таком случае необходимо перезапустить процесс сборки. Ниже указана команда со списком пакетов для сборки приложения `map_rwx` из пакета `ksys_mem_tests`.

```bash
pushd .
	cd "$L4RE_BUILDDIR"
	make "$MAKE_OPTS" S=libirq,input,libvbus,libio,libio-io,drivers,drivers-frst,bootstrap,libsupc++,libgcc-pure,libgcc,crtn,libkproxy,libsigma0,libsupc++-minimal,uclibc-minimal,lua,ned,ksys_mem_tests,log,libc_backends,cxx,cxx_libc_io,l4re,l4re_c,l4re_kernel,l4re_vfs,l4sys,l4util,ldscripts,ldso,libloader,loader,uclibc,moe
popd
```

### создание образа u-boot для загрузки на устройстве (Pandaboard)
```bash
	make MODULES_LIST=/mnt/disk2/l4_snapshot/L4Reap/l4/pkg/ksys_mem_tests/conf/modules.list uimage E=map_rwx

	# copy the uimage to the device or the TFTP server or whatever
	cp images/bootstrap.uimage /srv/tftp/panda/uImage
```

## сборка L4Linux
Необходимо дополнительно собрать следующие пакеты среды L4Re:

> io,libstdc++-v3,libvcpu,shmc

Чтобы собрать L4Linux, сначала подготовим каталог для сборки.
```bash
pushd .
	cd "$FOC_ROOT/l4linux"
	make O="$L4LINUX_BUILDDIR" arm_defconfig
popd
```

Непосредственно процесс сборки. Можно задать опции конфигурации ядра при помощи компанды `make menuconfig`. Также, необходимо указать в опциях конфигурации или добавлением строки в файл `.config` путь к каталогу сборки микроядра fiasco.
```
pushd .
	cd "$L4LINUX_BUILDDIR"
	make menuconfig

	echo CONFIG_L4_OBJ_TREE=\\"$L4_BUILDDIR\\" >> $L4BUI/l4reap/.config

	make O=$PWD/../build_l4re L4ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi- -j10
	mkdir -p "$L4RE_BUILDDIR/bin/arm_armv7a"
	cp vmlinuz.arm "$L4RE_BUILDDIR/bin/arm_armv7a/l4f/vmlinuz.arm" 
popd
```

### сборка начального RAMdisk
Можно собрать RAMdisk (виртуальный диск с образом корневой файловой системы для L4Linux) при помощи утилит типа buildroot или использовать готовый образ.
Чтобы получить готовый образ:

```bash
pushd .
	cd "$L4RE_BUILDDIR"
	wget http://genode.org/files/release-11.11/l4lx/initrd-arm.gz -O bin/arm_armv7a/ramdisk-arm.rd
popd
```

Может возникнуть необходимость отредактировать содержимое загрузочного диска. Для этого можно использовать утилиту `cpio`.

Извлечение содержимого виртуального диска.
```bash
cpio -i /path/to/ramdisk.img
```

Cоздание нового образа.
```
find . | cpio -o -H newc > /path/to/new_ramdisk.img

```

## компиляция тестовых утилит PaX для L4Linux
```
	tar xvf pax_tests_l4linux.tar.gz
	cd pax_tests_l4linux
	make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi-
```

Теперь двоичные файлы доступны в каталоге pax_tests_l4linux.
