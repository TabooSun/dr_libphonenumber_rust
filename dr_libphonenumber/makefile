.DEFAULT_GOAL := help
PROJECTNAME=$(shell basename "$(PWD)")
SOURCES=src/lib.rs

OS_NAME=$(shell uname | tr '[:upper:]' '[:lower:]')
PATH := $(ANDROID_NDK_HOME)/toolchains/llvm/prebuilt/$(OS_NAME)-x86_64/bin:$(PATH)

ANDROID_AARCH64_LINKER=$(ANDROID_NDK_HOME)/toolchains/llvm/prebuilt/$(OS_NAME)-x86_64/bin/aarch64-linux-android32-clang
ANDROID_ARMV7_LINKER=$(ANDROID_NDK_HOME)/toolchains/llvm/prebuilt/$(OS_NAME)-x86_64/bin/armv7a-linux-androideabi32-clang
ANDROID_I686_LINKER=$(ANDROID_NDK_HOME)/toolchains/llvm/prebuilt/$(OS_NAME)-x86_64/bin/i686-linux-android32-clang
ANDROID_X86_64_LINKER=$(ANDROID_NDK_HOME)/toolchains/llvm/prebuilt/$(OS_NAME)-x86_64/bin/x86_64-linux-android32-clang

SHELL := /bin/bash

DRLIB_PHONENUMBER_IOS_PLUGIN_PATH=../../ios
DRLIB_PHONENUMBER_MACOS_PLUGIN_PATH=../../macos

DRLIB_PHONENUMBER_DYNAMIC_LIB_DIR_PATH=../../test/artifacts
DRLIB_PHONENUMBER_WINDOWS_ARTIFACTS_PATH=$(DRLIB_PHONENUMBER_DYNAMIC_LIB_DIR_PATH)/x86_64-pc-windows-gnu
DRLIB_PHONENUMBER_MACOS_INTEL_ARTIFACTS_PATH=$(DRLIB_PHONENUMBER_DYNAMIC_LIB_DIR_PATH)/x86_64-apple-darwin
DRLIB_PHONENUMBER_MACOS_APPLE_SILICON_ARTIFACTS_PATH=$(DRLIB_PHONENUMBER_DYNAMIC_LIB_DIR_PATH)/aarch64-apple-darwin
DRLIB_PHONENUMBER_LINUX_ARTIFACTS_PATH=$(DRLIB_PHONENUMBER_DYNAMIC_LIB_DIR_PATH)/x86_64-unknown-linux-gnu

IOS_SIM_FAT_LIBRARY_PATH=target/lipo/ios-sim
MACOS_FAT_LIBRARY_PATH=target/lipo/macos
STATIC_LIBRARY_NAME="libdr_libphonenumber.a"

# ##############################################################################
# # GENERAL
# ##############################################################################

.PHONY: help
help: makefile
	@echo
	@echo " Available actions in "$(PROJECTNAME)":"
	@echo
	@sed -n 's/^##//p' $< | column -t -s ':' |  sed -e 's/^/ /'
	@echo

## init: Install missing dependencies.
.PHONY: init
init:
	rustup target add --toolchain nightly x86_64-pc-windows-gnu
	rustup target add --toolchain nightly aarch64-unknown-linux-gnu
	rustup target add aarch64-apple-ios x86_64-apple-ios aarch64-apple-darwin
	# This is for testing purpose on macOS.
	rustup target add x86_64-apple-darwin
	#rustup target add armv7-apple-ios armv7s-apple-ios i386-apple-ios ## deprecated
	rustup target add aarch64-linux-android armv7-linux-androideabi i686-linux-android x86_64-linux-android
	cargo install cargo-ndk
	@if [ $$(uname) == "Darwin" ] ; then \
  		brew install mingw-w64 ; \
		cargo install cargo-lipo ; \
		rustup target add aarch64-apple-ios-sim --toolchain nightly ; \
		rustup component add rust-src --toolchain nightly-aarch64-apple-darwin ; \
	elif [ $$(uname) == "Linux" ] ; then \
	    rustup component add rust-src --toolchain nightly --target x86_64-unknown-linux-gnu ; \
	    rustup component add rust-src --toolchain nightly --target aarch64-unknown-linux-gnu ; \
	fi
	cargo install cbindgen

## :

# ##############################################################################
# # RECIPES
# ##############################################################################

## all: Compile iOS, Android and bindings targets
all: bindings apple android windows linux

linux:
	cargo build --target x86_64-unknown-linux-gnu --release
	mkdir -p $(DRLIB_PHONENUMBER_LINUX_ARTIFACTS_PATH)
	cp target/x86_64-unknown-linux-gnu/release/libdr_libphonenumber.so $(DRLIB_PHONENUMBER_LINUX_ARTIFACTS_PATH)/libdr_libphonenumber.so

## error: linker `x86_64-w64-mingw32-gcc` not found. Fix: sudo apt install mingw-w64
windows:
	cargo build --target x86_64-pc-windows-gnu --release
	mkdir -p $(DRLIB_PHONENUMBER_WINDOWS_ARTIFACTS_PATH)
	cp target/x86_64-pc-windows-gnu/release/dr_libphonenumber.dll $(DRLIB_PHONENUMBER_WINDOWS_ARTIFACTS_PATH)/libdr_libphonenumber.dll

## ios: Compile the iOS universal library
apple: 
	@if [ $$(uname) == "Darwin" ] ; then \
  		set -x; \
		cargo lipo --release ; \
		cargo build --target=x86_64-apple-darwin --release ; \
		cargo build --target=aarch64-apple-darwin --release ; \
		cargo build --target=x86_64-apple-ios --release ; \
		cargo +nightly build -Zbuild-std --target aarch64-apple-ios-sim --release ; \
		mkdir -p $(IOS_SIM_FAT_LIBRARY_PATH) ; \
		mkdir -p $(MACOS_FAT_LIBRARY_PATH) ; \
		lipo -create target/x86_64-apple-ios/release/$(STATIC_LIBRARY_NAME) target/aarch64-apple-ios-sim/release/$(STATIC_LIBRARY_NAME) -output $(IOS_SIM_FAT_LIBRARY_PATH)/$(STATIC_LIBRARY_NAME) ; \
		lipo -create target/x86_64-apple-darwin/release/$(STATIC_LIBRARY_NAME) target/aarch64-apple-darwin/release/$(STATIC_LIBRARY_NAME) -output $(MACOS_FAT_LIBRARY_PATH)/$(STATIC_LIBRARY_NAME) ; \
		./buildappleartifacts.sh ; \
		@echo "[DONE] Build iOS/macOS xcframeworks" ; \
		mkdir -p $(DRLIB_PHONENUMBER_MACOS_INTEL_ARTIFACTS_PATH) ; \
		mkdir -p $(DRLIB_PHONENUMBER_MACOS_APPLE_SILICON_ARTIFACTS_PATH) ; \
		cp target/x86_64-apple-darwin/release/libdr_libphonenumber.dylib $(DRLIB_PHONENUMBER_MACOS_INTEL_ARTIFACTS_PATH)/libdr_libphonenumber.dylib ; \
		cp target/aarch64-apple-darwin/release/libdr_libphonenumber.dylib $(DRLIB_PHONENUMBER_MACOS_APPLE_SILICON_ARTIFACTS_PATH)/libdr_libphonenumber.dylib ; \
		@echo "[DONE] Moving macOS artifacts" ; \
		else echo "Skipping Apple (iOS, macOS and etc.) compilations on $$(uname)" ; \
	fi

## android: Compile all the android targets using cargo-ndk
## Check https://github.com/bbqsrc/cargo-ndk/issues/22 for the reason of using +nightly with -Zbuild-std
android: 
	cargo +nightly ndk -t arm64-v8a -t armeabi-v7a -t x86 -t x86_64 -o ../../android/src/main/jniLibs build --release -Zbuild-std

.PHONY: ndk-home bindings
ndk-home:
	@if [ ! -d "${ANDROID_NDK_HOME}" ] ; then \
		echo "Error: Please, set the ANDROID_NDK_HOME env variable to point to your NDK folder" ; \
		exit 1 ; \
	fi

bindingsPath = includes/bindings.h

## bindings: Generate the .h file for iOS
bindings:
	cbindgen $(SOURCES) -c cbindgen.toml -o $(bindingsPath)
	@echo "[DONE] $@"
	cp $(bindingsPath) $(DRLIB_PHONENUMBER_IOS_PLUGIN_PATH)/Classes/DrLibPhonenumberBindings.h
	@echo "[DONE] Copy header file to ios plugin"
	cp $(bindingsPath) $(DRLIB_PHONENUMBER_MACOS_PLUGIN_PATH)/Classes/DrLibPhonenumberBindings.h
	@echo "[DONE] Copy header file to macos plugin"
## :

# ##############################################################################
# # OTHER
# ##############################################################################

## clean:
.PHONY: clean
clean:
	rm -rf $(DRLIB_PHONENUMBER_DYNAMIC_LIB_DIR_PATH)
	cargo clean

## test:
.PHONY: test
test:
	cargo test
