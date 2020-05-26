# -*- mode: Makefile; -*-
# Copyright (c) Microsoft Corporation

BASE_DIR := $(subst /build/linux,,$(PWD))

PF_POSIX := 1
include $(BASE_DIR)/build/linux/config.mak
include $(BASE_DIR)/build/linux/Makefile.common


ifndef ENABLE_DEBUG
$(error "ENABLE_DEBUG is not set.  Please re-run configure")
endif

# Include the version file
include $(BASE_DIR)/build/linux/docker.version

ifndef CONTAINER_BUILDVERSION_STATUS
$(error "Is docker.version missing?  Please re-run configure")
endif


SOURCE_DIR := $(BASE_DIR)/source/code
TEST_DIR := $(BASE_DIR)/test/code

PROVIDER_DIR := $(SOURCE_DIR)/providers
PROVIDER_TEST_DIR := $(TEST_DIR)/providers

INTERMEDIATE_DIR := $(BASE_DIR)/intermediate/$(BUILD_CONFIGURATION)
INTERMEDIATE_TESTFILES := $(INTERMEDIATE_DIR)/testfiles
TARGET_DIR := $(BASE_DIR)/target/$(BUILD_CONFIGURATION)
PROVIDER_LIBRARY := $(INTERMEDIATE_DIR)/libcontainer.so

INSTALLER_TMPDIR := $(INTERMEDIATE_DIR)/installer_tmp


# GO Source dir for custom fluent bit plugin
GO_SOURCE_DIR := $(SOURCE_DIR)/go/src/plugins

ifeq ($(ENABLE_DEBUG),1)
PROV_DEBUG_FLAGS := -g
endif

# Support for installbuilder

STAGING_DIR := $(INTERMEDIATE_DIR)/staging

ifeq ($(ULINUX),1)
  # For consistency, the architecture should be i686 (for x86) and x86_64 (for x64)
  DOCKER_ARCH := $(shell echo $(PF_ARCH) | sed -e 's/x86$$/i686/' -e 's/x64$$/x86_64/')
  OUTPUT_PACKAGE_PREFIX=docker-cimprov-$(CONTAINER_BUILDVERSION_MAJOR).$(CONTAINER_BUILDVERSION_MINOR).$(CONTAINER_BUILDVERSION_PATCH)-$(CONTAINER_BUILDVERSION_BUILDNR).universal.$(DOCKER_ARCH)
else
  PF_DISTRO_LC := $(shell echo $(PF_DISTRO) | tr A-Z a-z)
  OUTPUT_PACKAGE_PREFIX=docker-cimprov-$(CONTAINER_BUILDVERSION_MAJOR).$(CONTAINER_BUILDVERSION_MINOR).$(CONTAINER_BUILDVERSION_PATCH)-$(CONTAINER_BUILDVERSION_BUILDNR).$(PF_DISTRO_LC).$(PF_MAJOR).$(PF_ARCH)
endif

ifeq ("$(wildcard /usr/bin/dpkg-deb)","")
	DPKG_LOCATION="--DPKG_LOCATION=$(BASE_DIR)/build/linux/installer/InstallBuilder/tools/bin/dpkg-deb-$(PF_ARCH)"
else
	DPKG_LOCATION=
endif

#--------------------------------------------------------------------------------
# Build targets

ifeq ($(ULINUX),1)
all : PROVIDER_STATUS KIT_STATUS kit fluentbitplugin rubypluginstests
else
all : PROVIDER_STATUS fluentbitplugin rubypluginstests
endif

clean :
	$(RMDIR) $(BASE_DIR)/build/cppunit_result.* $(BASE_DIR)/installer/intermediate $(BASE_DIR)/intermediate $(BASE_DIR)/target
	-find $(BASE_DIR) -name \*~ -exec rm {} \;

distclean : clean
	$(RM) $(BASE_DIR)/build/config.mak

PROVIDER_STATUS:
	@echo "========================= Performing Building provider"

KIT_STATUS:
	@echo "========================= Performing Building provider tests"


#---------------------------------------------------------------------------------
# fluentbit go plugin build. This is required to send container logs to ODS endpoint
#
fluentbitplugin :
	@echo "========================= Building fluentbit out_oms go plugin for logs"
	$(MKPATH) $(INTERMEDIATE_DIR)
	make -C $(GO_SOURCE_DIR) fbplugin
	$(COPY) $(GO_SOURCE_DIR)/out_oms.so $(INTERMEDIATE_DIR)

rubypluginstests :
	@echo "========================= Installing pre-reqs for running tests"
	sudo apt-add-repository ppa:brightbox/ruby-ng -y
	sudo apt-get update
	sudo apt-get install ruby2.4 rake -y
	sudo gem install minitest
	@echo "========================= Running tests..."
	rake test

#--------------------------------------------------------------------------------
# Build the distribution kit
#
# Build the packages via installbuilder
#
# While the "formal build" only builds ULINUX, we may build something else for DEV purposes.
# Assume we ALWAYS build DPKG, but only build RPM if --enable-ulinux is speified in configure.

kit : CONTAINERLIB_FILENAME = libcontainer.so
kit : fluentbitplugin

ifeq ($(ULINUX),1)

	@echo "========================= Performing Building RPM and DPKG packages"
	$(MKPATH) $(INSTALLER_TMPDIR)
	sudo $(RMDIR) $(STAGING_DIR)
	$(MKPATH) $(INTERMEDIATE_DIR)
	python $(BASE_DIR)/build/linux/installer/InstallBuilder/installbuilder.py \
		--BASE_DIR=$(BASE_DIR) \
		--TARGET_DIR=$(INTERMEDIATE_DIR) \
		--INTERMEDIATE_DIR=$(INSTALLER_TMPDIR) \
		--STAGING_DIR=$(STAGING_DIR) \
		--BUILD_TYPE=$(BUILD_TYPE) \
		--BUILD_CONFIGURATION=$(BUILD_CONFIGURATION) \
		--PFARCH=$(PF_ARCH) \
		--PFDISTRO=$(PF_DISTRO) \
		--PFMAJOR=$(PF_MAJOR) \
		--PFMINOR=$(PF_MINOR) \
		--VERSION=$(CONTAINER_BUILDVERSION_MAJOR).$(CONTAINER_BUILDVERSION_MINOR).$(CONTAINER_BUILDVERSION_PATCH) \
		--RELEASE=$(CONTAINER_BUILDVERSION_BUILDNR) \
		--CONTAINER_BUILD_LIBRARY=$(CONTAINERLIB_FILENAME) \
		--OUTPUTFILE=$(OUTPUT_PACKAGE_PREFIX) \
		--DATAFILE_PATH=$(BASE_DIR)/build/linux/installer/datafiles \
		base_container.data linux.data linux_rpm.data

	sudo $(RMDIR) $(STAGING_DIR)
	$(MKPATH) $(INTERMEDIATE_DIR)
	python $(BASE_DIR)/build/linux/installer/InstallBuilder/installbuilder.py \
		--BASE_DIR=$(BASE_DIR) \
		--TARGET_DIR=$(INTERMEDIATE_DIR) \
		--INTERMEDIATE_DIR=$(INSTALLER_TMPDIR) \
		--STAGING_DIR=$(STAGING_DIR) \
		--BUILD_TYPE=$(BUILD_TYPE) \
		--BUILD_CONFIGURATION=$(BUILD_CONFIGURATION) \
		--PFARCH=$(PF_ARCH) \
		--PFDISTRO=$(PF_DISTRO) \
		--PFMAJOR=$(PF_MAJOR) \
		--PFMINOR=$(PF_MINOR) \
		--VERSION=$(CONTAINER_BUILDVERSION_MAJOR).$(CONTAINER_BUILDVERSION_MINOR).$(CONTAINER_BUILDVERSION_PATCH) \
		--RELEASE=$(CONTAINER_BUILDVERSION_BUILDNR) \
		--CONTAINER_BUILD_LIBRARY=$(CONTAINERLIB_FILENAME) \
		$(DPKG_LOCATION) \
		--OUTPUTFILE=$(OUTPUT_PACKAGE_PREFIX) \
		--DATAFILE_PATH=$(BASE_DIR)/build/linux/installer/datafiles \
		base_container.data linux.data linux_dpkg.data

	# Strip the package extension from the package filename
	sed -re 's/.rpm$$|.deb$$//' $(INTERMEDIATE_DIR)/package_filename > $(INTERMEDIATE_DIR)/package_file.tmp; mv $(INTERMEDIATE_DIR)/package_file.tmp $(INTERMEDIATE_DIR)/package_filename

	# Build the tar file containing both .rpm and .deb packages
	cd $(INTERMEDIATE_DIR); tar cvf $(OUTPUT_PACKAGE_PREFIX).tar $(OUTPUT_PACKAGE_PREFIX).rpm $(OUTPUT_PACKAGE_PREFIX).deb

	bash installer/bundle/create_bundle.sh $(PF)_$(PF_DISTRO) $(INTERMEDIATE_DIR) $(OUTPUT_PACKAGE_PREFIX)
	# Copy the shell bundle to the target directory
	$(MKPATH) $(TARGET_DIR)
	cd $(INTERMEDIATE_DIR); $(COPY) `cat $(INTERMEDIATE_DIR)/package_filename`.sh $(TARGET_DIR)

else

	@echo "========================= Performing Building RPM and DPKG packages"
	sudo $(RMDIR) $(STAGING_DIR)
	$(MKPATH) $(INTERMEDIATE_DIR)
	python $(BASE_DIR)/build/linux/installer/InstallBuilder/installbuilder.py \
		--BASE_DIR=$(BASE_DIR) \
		--TARGET_DIR=$(INTERMEDIATE_DIR) \
		--INTERMEDIATE_DIR=$(INSTALLER_TMPDIR) \
		--STAGING_DIR=$(STAGING_DIR) \
		--BUILD_TYPE=$(BUILD_TYPE) \
		--BUILD_CONFIGURATION=$(BUILD_CONFIGURATION) \
		--PFARCH=$(PF_ARCH) \
		--PFDISTRO=$(PF_DISTRO) \
		--PFMAJOR=$(PF_MAJOR) \
		--PFMINOR=$(PF_MINOR) \
		--VERSION=$(CONTAINER_BUILDVERSION_MAJOR).$(CONTAINER_BUILDVERSION_MINOR).$(CONTAINER_BUILDVERSION_PATCH) \
		--RELEASE=$(CONTAINER_BUILDVERSION_BUILDNR) \
		--CONTAINER_BUILD_LIBRARY=$(CONTAINERLIB_FILENAME) \
		$(DPKG_LOCATION) \
		--OUTPUTFILE=$(OUTPUT_PACKAGE_PREFIX) \
		--DATAFILE_PATH=$(BASE_DIR)/build/linux/installer/datafiles \
		base_container.data linux.data linux_dpkg.data

endif