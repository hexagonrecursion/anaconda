# Makefile.am for anaconda
#
# Copyright (C) 2009  Red Hat, Inc.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU Lesser General Public License as published
# by the Free Software Foundation; either version 2.1 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU Lesser General Public License for more details.
#
# You should have received a copy of the GNU Lesser General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.

include ./branch-config.mk

ACLOCAL_AMFLAGS = -I m4

SUBDIRS = data docs dracut po pyanaconda scripts tests widgets ui utils

EXTRA_DIST = COPYING

# Include the xgettext wrapper so pot-update can be run from the source distribution
# This is needed for make distcheck.
EXTRA_DIST += $(srcdir)/translation-canary/xgettext_werror.sh

MAINTAINERCLEANFILES =	Makefile.in config.guess config.h.in config.sub \
						depcomp install-sh ltmain.sh missing ABOUT-NLS \
						INSTALL aclocal.m4 configure *.pyc \
						py-compile m4/* po/Makefile.in.in po/Rules-quot \
						test-driver

CLEANFILES = *~

dist_noinst_DATA      = $(PACKAGE_NAME).spec

dist_sbin_SCRIPTS     = anaconda.py

install-exec-hook:
	cd $(DESTDIR)$(sbindir) && mv anaconda.py anaconda

uninstall-hook:
	-cd $(DESTDIR)$(sbindir) && rm -f anaconda


# Set environment if 'make -f Makefile.am' is called to avoid autotools
srcdir ?= $(CURDIR)

ARCHIVE_TAG   = $(PACKAGE_NAME)-$(PACKAGE_VERSION)-$(PACKAGE_RELEASE)
# Set this to "true" if you want to have SRPM archive with test version
TEST_BUILD	?= "false"

# LOCALIZATION SETTINGS
L10N_REPOSITORY ?= https://github.com/rhinstaller/anaconda-l10n.git
# Branch used in anaconda-l10n repository. This should be master all the time.
GIT_L10N_BRANCH ?= master

RC_RELEASE ?= $(shell date -u +0.1.%Y%m%d%H%M%S)

PYTHON ?= python3
COVERAGE ?= $(PYTHON) -m coverage
USER_SITE_BASE ?= $(abs_top_builddir)/python-site
USER_SITE_PACKAGES ?= $(shell PYTHONUSERBASE=$(USER_SITE_BASE) $(PYTHON) -m site --user-site)

RESULT_DIR ?= $(abs_top_builddir)/result
BUILD_RESULT_DIR = $(RESULT_DIR)/build
TEST_RESULT_DIR = $(RESULT_DIR)/tests

SRPM_BUILD_DIR = $(BUILD_RESULT_DIR)/00-srpm-build
RPM_BUILD_DIR = $(BUILD_RESULT_DIR)/01-rpm-build
TEST_INST_BUILD_DIR = $(BUILD_RESULT_DIR)/02-test-install

CONTAINER_ENGINE ?= podman
# build/run tweaks for all containers, such as --cap-add
CONTAINER_BUILD_ARGS ?= --no-cache --pull-always
# HACK: bash's builtin `test -r` fails when running on Ubuntu host (GitHub) due to incompatible seccomp profile
CONTAINER_TEST_ARGS ?= $(shell grep -q ID=ubuntu /etc/os-release && echo --security-opt=seccomp=unconfined)
CONTAINER_REGISTRY ?= quay.io
# Add additional args to the existing ones to container engine
CONTAINER_ADD_ARGS ?=

# anaconda-ci container
CI_DOCKERFILE ?= $(srcdir)/dockerfile/anaconda-ci
RPM_DOCKERFILE ?= $(srcdir)/dockerfile/anaconda-rpm
ISO_CREATOR_DOCKERFILE ?= $(srcdir)/dockerfile/anaconda-iso-creator
CI_NAME ?= $(CONTAINER_REGISTRY)/rhinstaller/anaconda-ci
RPM_NAME ?= $(CONTAINER_REGISTRY)/rhinstaller/anaconda-rpm
ISO_CREATOR_NAME ?= $(CONTAINER_REGISTRY)/rhinstaller/anaconda-iso-creator
CI_TAG := $(or $(CI_TAG),$(GIT_BRANCH))
CI_TEST_ARGS ?= --rm -v $(srcdir):/anaconda:z
CI_CMD ?= make ci

SKIP_BRANCHING_CHECK ?= "false"

# If translations are present, run tests on the .po files before tarring them
# up. Use a weird looking loop because shell doesn't have a good way to test
# for a wildcard
dist-hook:
	for p in $(distdir)/po/*.po ; do \
	    if [ -e "$$p" ]; then \
		PYTHONPATH=$(srcdir)/translation-canary $(PYTHON) -m translation_canary.translated \
		    --release $(distdir)/po ; \
	    fi ; \
	    break ; \
	done

tag:
	@git tag -s -a -m "Tag as $(ARCHIVE_TAG)" $(ARCHIVE_TAG)
	@echo "Tagged as $(ARCHIVE_TAG)"

po-pull:
	TEMP_DIR=$$(mktemp --tmpdir -d anaconda-localization-XXXXXXXXXX) && \
	git clone --depth 1 -b $(GIT_L10N_BRANCH) -- $(L10N_REPOSITORY) $$TEMP_DIR && \
	cp $$TEMP_DIR/$(L10N_DIR)/*.po $(srcdir)/po/ && \
	rm -rf $$TEMP_DIR && \
	find $(srcdir)/po -type f -exec sed -ri '/^\"Plural-Forms:/ s/;(\\n")$$/\1/' {} +

pot:
	$(MAKE) -C po $(PACKAGE_NAME).pot-update
	rm $(srcdir)/po/{main,main_js,extra}.pot

pot-update-check: pot
# This check is used by github action for auto update of pot file
# This algorithm will make these steps:
# - clone localization repository
# - copy pot file to this repository
# - check if pot file is changed (ignore the POT-Creation-Date otherwise it's always changed)
	@TEMP_DIR=$$(mktemp --tmpdir -d anaconda-localization-XXXXXXXXXX) || exit 5 ; \
	git clone --depth 1 -b $(GIT_L10N_BRANCH) -- $(L10N_REPOSITORY) $$TEMP_DIR || exit 6 ; \
	cp $(srcdir)/po/anaconda.pot $$TEMP_DIR/$(L10N_DIR)/ || exit 7 ; \
	pushd $$TEMP_DIR/$(L10N_DIR) ; \
	git difftool --trust-exit-code -y -x "diff -u -I '^\"POT-Creation-Date: .*$$'" HEAD ./anaconda.pot &>/dev/null ; \
	RESULT=$$?; \
	popd ; \
	rm -rf $$TEMP_DIR; \
	if [ $$RESULT -eq 0  ] ; then \
		echo "Pot file is up to date."; \
	else \
		echo "Pot file needs update! Please update the pot file ./po/anaconda.pot in $(L10N_REPOSITORY)"; \
	fi ;

# Try to fetch translations, but if that fails just keep going
po-fallback:
	-$(MAKE) po-pull

scratch:
	if [ "$(TEST_BUILD)" == "true" ]; then \
		sed -ri '/AC_INIT/ s/\[[0-9.]+\]/[999999999]/' $(srcdir)/configure.ac; \
	fi
	$(MAKE) po-pull
	$(MAKE) ARCHIVE_TAG=HEAD dist

scratch-bumpver:
	@opts="-S -n $(PACKAGE_NAME) -v $(PACKAGE_VERSION) -r $(PACKAGE_RELEASE) -b $(PACKAGE_BUGREPORT) --newrelease $(RC_RELEASE)" ; \
	if [ ! -z "$(IGNORE)" ]; then \
		opts="$${opts} -i $(IGNORE)" ; \
	fi ; \
	if [ ! -z "$(MAP)" ]; then \
		opts="$${opts} -m $(MAP)" ; \
	fi ; \
	if [ ! -z "$(BZDEBUG)" ]; then \
		opts="$${opts} -d" ; \
	fi ; \
	( cd $(srcdir) && scripts/makebumpver $${opts} ) || exit 1 ; \
	$(MAKE) -C po $(PACKAGE_NAME).pot-update

release:
	$(MAKE) po-pull
	$(MAKE) dist

release-and-tag:
	$(MAKE) dist
	$(MAKE) tag

anaconda-ci-build:
	TEMP=$$(mktemp -t -d anaconda-ci-build.XXXX) && \
	cp $(srcdir)/anaconda.spec.in $(CI_DOCKERFILE)/* $$TEMP/ && \
	echo "Build dir is $$TEMP" && \
	$(CONTAINER_ENGINE) build \
	$(CONTAINER_BUILD_ARGS) \
	$(CONTAINER_ADD_ARGS) \
	--build-arg=git_branch=$(GIT_BRANCH) \
	--build-arg=image=$(BASE_CONTAINER) \
	--build-arg=copr_repo=$(COPR_REPO) \
	-t $(CI_NAME):$(CI_TAG) \
	$$TEMP/ && \
	rm -rf $$TEMP

anaconda-rpm-build:
	TEMP=$$(mktemp -t -d anaconda-rpm-build.XXXX) && \
	cp $(srcdir)/anaconda.spec.in $(RPM_DOCKERFILE)/* $$TEMP/ && \
	echo "Build dir is $$TEMP" && \
	$(CONTAINER_ENGINE) build \
	$(CONTAINER_BUILD_ARGS) \
	$(CONTAINER_ADD_ARGS) \
	--build-arg=git_branch=$(GIT_BRANCH) \
	--build-arg=image=$(BASE_CONTAINER) \
	--build-arg=copr_repo=$(COPR_REPO) \
	-t $(RPM_NAME):$(CI_TAG) \
	$$TEMP/ && \
	rm -rf $$TEMP

anaconda-iso-creator-build:
	$(CONTAINER_ENGINE) build \
	$(CONTAINER_BUILD_ARGS) \
	$(CONTAINER_ADD_ARGS) \
	--build-arg=image=$(BASE_CONTAINER) \
	-t $(ISO_CREATOR_NAME):$(CI_TAG) \
  $(ISO_CREATOR_DOCKERFILE)

# User has to be logged first to be able to push the image.
# See `podman login` for more info.
anaconda-ci-push:
	$(CONTAINER_ENGINE) push \
	$(CONTAINER_ADD_ARGS) \
	$(CI_NAME):$(CI_TAG)

anaconda-rpm-push:
	$(CONTAINER_ENGINE) push \
	$(CONTAINER_ADD_ARGS) \
	$(RPM_NAME):$(CI_TAG)

bumpver: po-pull
	@opts="-n $(PACKAGE_NAME) -v $(PACKAGE_VERSION) -r $(PACKAGE_RELEASE) -b $(PACKAGE_BUGREPORT)" ; \
	if [ ! -z "$(IGNORE)" ]; then \
		opts="$${opts} -i $(IGNORE)" ; \
	fi ; \
	if [ ! -z "$(MAP)" ]; then \
		opts="$${opts} -m $(MAP)" ; \
	fi ; \
	if [ ! -z "$(BZDEBUG)" ]; then \
		opts="$${opts} -d" ; \
	fi ; \
	if [ ! -z "$(SKIP_ACKS)" ]; then \
		opts="$${opts} -s" ; \
	fi ; \
	( cd $(srcdir) && scripts/makebumpver $${opts} ) || exit 1
	-$(MAKE) pot-update-check

# (s)rpm builds

srpm: scratch anaconda.spec
	rpmbuild --define "_sourcedir $(abs_srcdir)" --define "_srcrpmdir $(SRPM_BUILD_DIR)" -bs anaconda.spec

rpms: srpm
	rpmbuild --rebuild --define "_rpmdir $(RPM_BUILD_DIR)" $(SRPM_BUILD_DIR)/anaconda-*.src.rpm
	mv "$(RPM_BUILD_DIR)"/*/*.rpm "$(RPM_BUILD_DIR)" # move rpms from per-architecture subdirectory

# GUI TESTING
runglade:
	ANACONDA_DATA=$(srcdir)/data \
	ANACONDA_WIDGETS_OVERRIDES=$(srcdir)/widgets/python \
	PYTHONPATH=$(srcdir):$(builddir)/pyanaconda/isys/.libs:$(srcdir)/widgets/python/:$(builddir)/widgets/src/.libs/ \
	LD_LIBRARY_PATH=$(builddir)/widgets/src/.libs \
	UIPATH=$(srcdir)/pyanaconda/ui/gui/ \
	GI_TYPELIB_PATH=$(builddir)/widgets/src/ \
	GLADE_CATALOG_SEARCH_PATH=$(srcdir)/widgets/glade \
	GLADE_MODULE_SEARCH_PATH=$(builddir)/widgets/src/.libs \
	glade ${GLADE_FILE}

ci:
	$(MAKE) run-tests || echo $$? > $(srcdir)/tests/error_occured
	$(MAKE) grab-logs

	@if [ -f $(srcdir)/tests/error_occured ]; then \
		echo "TEST FAILED"; \
		status=$$(cat $(srcdir)/tests/error_occured); \
		rm $(srcdir)/tests/error_occured; \
		exit $$status; \
	fi

# container-ci target will have disabled cockpit Makefiles by default
container-ci:
	$(CONTAINER_ENGINE) run \
	--entrypoint /anaconda/dockerfile/anaconda-ci/run-build-and-arg \
	-e CONFIGURE_ARGS="--disable-webui" \
	$(CONTAINER_TEST_ARGS) \
	$(CI_TEST_ARGS) \
	$(CONTAINER_ADD_ARGS) \
	$(CI_NAME):$(CI_TAG) \
	$(CI_CMD)

container-shell:
	$(CONTAINER_ENGINE) run -it \
	$(CONTAINER_TEST_ARGS) \
	$(CONTAINER_ADD_ARGS) \
	$(CI_TEST_ARGS) \
	$(CI_NAME):$(CI_TAG)

container-rpm-test:
	$(CONTAINER_ENGINE) run \
	$(CONTAINER_TEST_ARGS) \
	$(CI_TEST_ARGS) \
	-v $(CI_DOCKERFILE)/run-build-and-arg:/run-build-and-arg \
	$(CONTAINER_ADD_ARGS) \
	$(RPM_NAME):$(CI_TAG) \
	sh -exc ' \
	    /run-build-and-arg make run-rpm-tests-only; \
	    dnf install -y /tmp/anaconda/result/build/01-rpm-build/*.rpm'

container-rpms:
	$(CONTAINER_ENGINE) run \
	$(CONTAINER_TEST_ARGS) \
	$(CI_TEST_ARGS) \
	-v $(CI_DOCKERFILE)/run-build-and-arg:/run-build-and-arg \
	$(CONTAINER_ADD_ARGS) \
	$(RPM_NAME):$(CI_TAG) \
	sh -exc ' \
			/run-build-and-arg make rpms && \
			rm -rf ./result && \
			cp -rv /tmp/anaconda/result ./'

container-rpms-scratch:
	$(MAKE) -f ./Makefile.am CONTAINER_ADD_ARGS="-e TEST_BUILD=true" container-rpms

check-branching:
# checking if branching can be finished and all the pieces are in place
	@echo "===================================="
	@echo "Testing branch configuration"
	@echo "===================================="
	@if [ "$(SKIP_BRANCHING_CHECK)" != "false" ]; then \
		echo "skipping..."; \
	else \
		CURRENT_BRANCH=$$(git rev-parse --abbrev-ref HEAD); \
		if [ "$$CURRENT_BRANCH" != "$(GIT_BRANCH)" ]; then \
			echo "*** Configuration is not adjusted for current branch!"; \
			echo "This check will work only on main development branch fXX-release, rhel-X ..."; \
			echo "current branch:  $$CURRENT_BRANCH"; \
			echo "expected branch: $(GIT_BRANCH)"; \
			exit 1; \
		fi; \
	fi

	@echo "===================================="
	@echo "Testing localization for $(L10N_DIR)"
	@echo "===================================="
	@curl --fail --silent --show-error -L https://raw.githubusercontent.com/rhinstaller/anaconda-l10n/$(GIT_L10N_BRANCH)/$(L10N_DIR)/anaconda.pot >/dev/null ||  \
		(echo "*** Can't download translations from a localization repository"; \
		 echo "repository: $(L10N_REPOSITORY)"; \
		 echo "branch:     $(GIT_L10N_BRANCH)"; \
		 echo "directory:  $(L10N_DIR)"; \
		 exit 2)

	@echo "===================================="
	@echo "Testing pykickstart configuration"
	@echo "===================================="
	@substituted_branch=$$(echo "$(GIT_BRANCH)" | sed 's/^master$$/DEVEL/'); \
	for i in $$(git grep 'from pykickstart.version import DEVEL as VERSION' -- pyanaconda/ dracut/ | awk -F "[: ]" '{print $$1 ";" $$5}'); do \
		i=$${i^^}; \
		if [ "$${i#*;}" != "$$substituted_branch" ]; then \
			echo "$${i%;*} is not substituted properly!"; \
			exit 3;\
		fi; \
	done

	@echo "SUCCESS"

grab-logs:
# can't be used after tests automatically because make will end when tests fails
# clean result dir
	-rm -rf $(TEST_RESULT_DIR)
	mkdir -p $(TEST_RESULT_DIR)

	-cd $(top_builddir)/tests/ && cp -r --parents ./**/*.log ./*.log* $(TEST_RESULT_DIR)
	-cd $(top_builddir)/tests/ && cp -r --parents ./*.log* $(TEST_RESULT_DIR)

run-tests:
	@mkdir -p $(USER_SITE_PACKAGES)
	@cp $(abs_builddir)/tests/usercustomize.py $(USER_SITE_PACKAGES)
	$(MAKE)
	$(MAKE) TMPDIR=/var/tmp COVERAGE_PROCESS_START=$(abs_builddir)/.coveragerc \
	    TEST_SUITE_LOG=test-suite.log PYTHONUSERBASE=$(USER_SITE_BASE) check
	@tail -n 1 tests/gettext_tests/*.log > tests/gettext_tests/gettext_tests.log
	@rm -rf $(USER_SITE_BASE)
	$(MAKE) coverage-report

tests-pep8:
	$(MAKE)
	$(MAKE) TMPDIR=/var/tmp TEST_SUITE_LOG=test-suite.log TESTS=pep8/runpep8.sh check

tests-pylint:
	$(MAKE)
	$(MAKE) TMPDIR=/var/tmp TEST_SUITE_LOG=test-suite.log TESTS=pylint/runpylint.py check

tests-unit-only:
	@mkdir -p $(USER_SITE_PACKAGES)
	@cp $(abs_builddir)/tests/usercustomize.py $(USER_SITE_PACKAGES)
	$(MAKE)
	$(MAKE) -C $(srcdir) TMPDIR=/var/tmp COVERAGE_PROCESS_START=$(abs_builddir)/.coveragerc \
		TEST_SUITE_LOG=test-suite.log TESTS=unit_tests/unit_tests.sh \
		PYTHONUSERBASE=$(USER_SITE_BASE) check
	@rm -rf $(USER_SITE_BASE)
	$(MAKE) coverage-report

run-rpm-tests-only: rpms
	$(MAKE) TEST_SUITE_LOG=test-suite.log RPM_PATH=$(RPM_BUILD_DIR) \
			ROOT_ANACONDA_PATH=$(abs_srcdir) TESTS=rpm_tests/rpm_tests.sh check

webui-tests: rpms
	$(MAKE) RPM_PATH=$(RPM_BUILD_DIR) --directory ui/webui integration-test

coverage-report:
	$(COVERAGE) combine tests/.coverage.*
	$(COVERAGE) xml -o tests/coverage-report.xml
	$(COVERAGE) report -m --omit "tests/*" > tests/coverage-report.log
	$(COVERAGE) report -m --include "tests/*" > tests/coverage-tests.log
	$(COVERAGE) report -m --include "pyanaconda/dbus*,pyanaconda/modules/*,pyanaconda/core/*" \
								     > tests/coverage-modular.log
	@cat tests/coverage-report.log
