#
# Copyright 2018 the original author or authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

ifneq ($(VOLTHA_BUILD),docker)
ifeq ($(VOLTHA_BASE)_set,_set)
$(error To get started, please source the env.sh file)
endif
endif

ifeq ($(TAG),)
TAG := latest
endif

ifeq ($(TARGET_TAG),)
TARGET_TAG := latest
endif

# If no DOCKER_HOST_IP is specified grab a v4 IP address associated with
# the default gateway
ifeq ($(DOCKER_HOST_IP),)
DOCKER_HOST_IP := $(shell ifconfig $$(netstat -rn | grep -E '^(default|0.0.0.0)' | head -1 | awk '{print $$NF}') | grep inet | awk '{print $$2}' | sed -e 's/addr://g')
endif

ifneq ($(http_proxy)$(https_proxy),)
# Include proxies from the environment
DOCKER_PROXY_ARGS = \
       --build-arg http_proxy=$(http_proxy) \
       --build-arg https_proxy=$(https_proxy) \
       --build-arg ftp_proxy=$(ftp_proxy) \
       --build-arg no_proxy=$(no_proxy) \
       --build-arg HTTP_PROXY=$(HTTP_PROXY) \
       --build-arg HTTPS_PROXY=$(HTTPS_PROXY) \
       --build-arg FTP_PROXY=$(FTP_PROXY) \
       --build-arg NO_PROXY=$(NO_PROXY)
endif

DOCKER_BUILD_ARGS = \
	--build-arg TAG=$(TAG) \
	--build-arg REGISTRY=$(REGISTRY) \
	--build-arg REPOSITORY=$(REPOSITORY) \
	$(DOCKER_PROXY_ARGS) $(DOCKER_CACHE_ARG) \
	 --rm --force-rm \
	$(DOCKER_BUILD_EXTRA_ARGS)

DOCKER_BUILD_EXTRA_ARGS = \
	--build-arg LOCAL_PYVOLTHA=${LOCAL_PYVOLTHA} \
	--build-arg LOCAL_PROTOS=${LOCAL_PROTOS}

VENVDIR := venv-$(shell uname -s | tr '[:upper:]' '[:lower:]')

DOCKER_IMAGE_LIST = \
	base \
	adapter-ponsim-olt \
	adapter-ponsim-onu

# The following list was scavanged from the compose / stack files as well as
# from the Dockerfiles. If nothing else it highlights that VOLTHA is not
# using consistent versions for some of the containers.

FETCH_BUILD_IMAGE_LIST = \
       alpine:3.6 \
       centos:7 \
       centurylink/ca-certs:latest \
       grpc/python:latest \
       ubuntu:xenial

FETCH_COMPOSE_IMAGE_LIST = \
        wurstmeister/kafka:latest \
        wurstmeister/zookeeper:latest

# find k8s -type f | xargs grep image: | awk '{print $NF}' | sed -e 's/\"//g' | sed '/:.*$/!s/$/:latest/g' | sort -u | sed -e 's/^/       /g' -e 's/$/ \\/g'
# Manually remove some image from this list as they don't reflect the new 
# naming conventions for the VOLTHA build
FETCH_K8S_IMAGE_LIST = \
       wurstmeister/kafka:1.0.0 \
       zookeeper:3.4.11

FETCH_IMAGE_LIST = $(shell echo $(FETCH_BUILD_IMAGE_LIST) $(FETCH_COMPOSE_IMAGE_LIST) $(FETCH_K8S_IMAGE_LIST) | tr ' ' '\n' | sort -u)

.PHONY: $(DIRS) $(DIRS_CLEAN) $(DIRS_FLAKE8) flake8 base ponsim_olt ponsim_onu protos cli ofagent kafka common start stop tag push pull

# This should to be the first and default target in this Makefile
help:
	@echo "Usage: make [<target>]"
	@echo "where available targets are:"
	@echo
	@echo "build        : Build the Adapters protos and docker images.\n\
               If this is the first time you are building, choose \"make build\" option."
	@echo "clean        : Remove files created by the build and tests"
	@echo "distclean    : Remove venv directory"
	@echo "fetch        : Pre-fetch artifacts for subsequent local builds"
	@echo "help         : Print this help"
	@echo "rebuild-venv : Rebuild local Python virtualenv from scratch"
	@echo "venv         : Build local Python virtualenv if did not exist yet"
	@echo "containers   : Build all the docker containers"
	@echo "base         : Build the base docker container used by all other dockers"
	@echo "adapter_ponsim_olt       : Build the ponsim olt adapter docker container"
	@echo "adapter_ponsim_onu       : Build the ponsim olt adapter docker container"
	@echo "ofagent      : Build the openflow agent docker container"
	@echo "tag          : Tag a set of images"
	@echo "push         : Push the docker images to an external repository"
	@echo "pull         : Pull the docker images from a repository"
	@echo

## New directories can be added here
#DIRS:=

## If one directory depends on another directory that
## dependency can be expressed here
##
## For example, if the Tibit directory depended on the eoam
## directory being built first, then that can be expressed here.
##  driver/tibit: eoam

# Parallel Build
$(DIRS):
	@echo "    MK $@"
	$(Q)$(MAKE) -C $@

# Parallel Clean
DIRS_CLEAN = $(addsuffix .clean,$(DIRS))
$(DIRS_CLEAN):
	@echo "    CLEAN $(basename $@)"
	$(Q)$(MAKE) -C $(basename $@) clean

# Parallel Flake8
DIRS_FLAKE8 = $(addsuffix .flake8,$(DIRS))
$(DIRS_FLAKE8):
	@echo "    FLAKE8 $(basename $@)"
	-$(Q)$(MAKE) -C $(basename $@) flake8

build: containers

containers: base adapter_ponsim_olt adapter_ponsim_onu ofagent cli

base:
	mkdir -p local_imports
ifdef LOCAL_PYVOLTHA
	mkdir -p local_imports/pyvoltha/dist
	cp ../../pyvoltha/dist/*.tar.gz local_imports/pyvoltha/dist/
endif
ifdef LOCAL_PROTOS
	mkdir -p local_imports/voltha-protos/dist
	cp ../../voltha-protos/dist/*.tar.gz local_imports/voltha-protos/dist/
endif
	docker build $(DOCKER_BUILD_ARGS) -t ${REGISTRY}${REPOSITORY}voltha-base:${TAG} -f docker/Dockerfile.base .


adapter_ponsim_olt:
	docker build $(DOCKER_BUILD_ARGS) -t ${REGISTRY}${REPOSITORY}voltha-adapter-ponsim-olt:${TAG} -f docker/Dockerfile.adapter_ponsim_olt .

adapter_ponsim_onu:
	docker build $(DOCKER_BUILD_ARGS) -t ${REGISTRY}${REPOSITORY}voltha-adapter-ponsim-onu:${TAG} -f docker/Dockerfile.adapter_ponsim_onu .

ofagent:
	docker build $(DOCKER_BUILD_ARGS) -t ${REGISTRY}${REPOSITORY}voltha-ofagent:${TAG} -f docker/Dockerfile.ofagent .

cli:
	docker build $(DOCKER_BUILD_ARGS) -t ${REGISTRY}${REPOSITORY}voltha-cli:${TAG} -f docker/Dockerfile.cli .

tag: $(patsubst  %,%.tag,$(DOCKER_IMAGE_LIST))

push: tag $(patsubst  %,%.push,$(DOCKER_IMAGE_LIST))

pull: $(patsubst  %,%.pull,$(DOCKER_IMAGE_LIST))

%.tag:
	docker tag ${REGISTRY}${REPOSITORY}voltha-$(subst .tag,,$@):${TAG} ${TARGET_REGISTRY}${TARGET_REPOSITORY}voltha-$(subst .tag,,$@):${TARGET_TAG}

%.push:
	docker push ${TARGET_REGISTRY}${TARGET_REPOSITORY}voltha-$(subst .push,,$@):${TARGET_TAG}

%.pull:
	docker pull ${REGISTRY}${REPOSITORY}voltha-$(subst .pull,,$@):${TAG}

clean:
	find . -name '*.pyc' | xargs rm -f
	rm -rf local_imports

distclean: clean
	rm -rf ${VENVDIR}

fetch:
	@bash -c ' \
		for i in $(FETCH_IMAGE_LIST); do \
			docker pull $$i; \
		done'

purge-venv:
	rm -fr ${VENVDIR}

rebuild-venv: purge-venv venv

ifneq ($(VOLTHA_BUILD),docker)
venv: ${VENVDIR}/.built
else
venv:
endif

${VENVDIR}/.built:
	@ virtualenv ${VENVDIR}
	@ . ${VENVDIR}/bin/activate && \
	    pip install --upgrade pip; \
	    if ! pip install -r requirements.txt; \
	    then \
	        echo "On MAC OS X, if the installation failed with an error \n'<openssl/opensslv.h>': file not found,"; \
	        echo "see the BUILD.md file for a workaround"; \
	    else \
	        uname -s > ${VENVDIR}/.built; \
	    fi

ifdef LOCAL_PYVOLTHA
	mkdir -p local_imports/pyvoltha/dist
	cp ../../pyvoltha/dist/*.tar.gz local_imports/pyvoltha/dist/
	. ${VENVDIR}/bin/activate && \
	    pip install local_imports/pyvoltha/dist/*.tar.gz
endif
ifdef LOCAL_PROTOS
	mkdir -p local_imports/voltha-protos/dist
	cp ../../voltha-protos/dist/*.tar.gz local_imports/voltha-protos/dist/
	. ${VENVDIR}/bin/activate && \
	    pip install local_imports/voltha-protos/dist/*.tar.gz
endif

flake8: $(DIRS_FLAKE8)

# end file
