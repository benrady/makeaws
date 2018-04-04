SHELL := $(shell which bash)
PYTHON_VERSION := $(shell cat .python-version)
PYENV_HOME := $(HOME)/.pyenv
PYENV := $(shell which pyenv || echo $(PYENV_HOME)/bin/pyenv)
PYTHON_HOME := $(PYENV_HOME)/versions/$(PYTHON_VERSION)
PYTHON := $(PYTHON_HOME)/bin/python
AWSCLI := $(PYTHON_HOME)/bin/aws
REGION := us-east-1
BUCKET := makeaws.benrady.com
BUCKET_INFO := .bucket_info
WEBSITE_INFO := .website_info
BUCKET_CONF := conf/s3api.create-bucket.json
WEBSITE_CONF := conf/s3api.put-bucket-website.json

help:
	@grep -E '^[0-9a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

$(PYTHON_HOME): 
	@echo "Installing Python $(PYTHON_VERSION)"
	# If this fails, check your system dependencies
	# https://github.com/pyenv/pyenv/wiki/Common-build-problems
	@$(PYENV) install $(PYTHON_VERSION)
	@$(PYENV) global $(PYTHON_VERSION)
	@$(PYENV) rehash

.env:
	@echo 'export PYENV_ROOT="$(PYENV_HOME)"' >> .env
	@echo 'export PYTHONPATH=.' >> .env
	@echo 'export PATH="$(PYENV_HOME)/bin:$${PATH}"' >> .env
	@$(PYENV) init - >> .env

$(PYENV):
	git clone https://github.com/pyenv/pyenv.git ~/.pyenv

.deps: requirements.txt | $(PYENV) $(PYTHON_HOME) .env
	@$(PYTHON_HOME)/bin/pip install -U pip
	@echo "Installing dependencies"
	@$(PYTHON_HOME)/bin/pip install -r requirements.txt
	@cp requirements.txt .deps

$(BUCKET_INFO): $(BUCKET_CONF)
	@$(AWSCLI) s3api create-bucket \
		--region $(REGION) \
		--bucket $(BUCKET) \
		--cli-input-json file://$(BUCKET_CONF) \
		> $(BUCKET_INFO)

$(WEBSITE_INFO): $(WEBSITE_CONF)
	@$(AWSCLI) s3api put-bucket-website \
		--region $(REGION) \
		--bucket $(BUCKET) \
		--cli-input-json file://$(WEBSITE_CONF) \
		> $(WEBSITE_INFO)

.PHONY: deploy server bucket

deploy: .deps ## Deploy webapp to S3 bucket
	@$(AWSCLI) s3 sync public/ s3://$(BUCKET) \
		--region $(REGION) \
		--acl public-read

bucket: $(BUCKET_INFO) $(WEBSITE_INFO) ## Create a new S3 bucket using the configuration in conf/*
	@echo Website URL: http://$(BUCKET).s3-website-$(REGION).amazonaws.com

server: .deps ## Run a development web server with livereload on port 35729
	@echo "Open http://localhost:35729 in your web browser"
	@$(PYTHON_HOME)/bin/livereload public
