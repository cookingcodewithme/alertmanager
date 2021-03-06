# Use `=` instead of `:=` expanding variable lazely, not at beginning. Needed as
# elm files change during execution.
ELM_FILES = $(shell find src -iname *.elm)
DOCKER_IMG := elm-env
DOCKER_RUN_CURRENT_USER := docker run --user=$(shell id -u $(USER)):$(shell id -g $(USER))
DOCKER_CMD := $(DOCKER_RUN_CURRENT_USER) --rm -t -v $(PWD):/app -w /app -e "ELM_HOME=/app" $(DOCKER_IMG)
# macOS requires mktemp template to be at the end of the filename.
TEMPFILE := $(shell mktemp ./elm-XXXXXXXXXX)
# --output flag for elm make must end in .js or .html.
TEMPFILE_JS := "$(TEMPFILE).js"
TEMPOPENAPI := $(shell mktemp -d ./openapi-XXXXXXXXXX)

ifeq ($(NO_DOCKER), true)
  DOCKER_CMD=
endif

all: script.js test

elm-env:
	@(if [ "$(NO_DOCKER)" != "true" ] ; then \
		echo ">> building elm-env docker image"; \
		docker build -t $(DOCKER_IMG) ../. > /dev/null; \
	fi; )

format: elm-env $(ELM_FILES)
	@echo ">> format front-end code"
	@$(DOCKER_CMD) elm-format --yes $(ELM_FILES)

test: src/Data elm-env
	@$(DOCKER_CMD) rm -rf elm-stuff/generated-code
	@$(DOCKER_CMD) elm-format $(ELM_FILES) --validate
	@$(DOCKER_CMD) elm-test

dev-server:
	elm reactor

script.js: src/Data elm-env format $(ELM_FILES)
	@echo ">> building script.js"
	@$(DOCKER_CMD) rm -rf elm-stuff
	@$(DOCKER_CMD) elm make src/Main.elm --optimize --output $(TEMPFILE_JS)
	@$(DOCKER_CMD) uglifyjs $(TEMPFILE_JS) --compress 'pure_funcs="F2,F3,F4,F5,F6,F7,F8,F9,A2,A3,A4,A5,A6,A7,A8,A9",pure_getters,keep_fargs=false,unsafe_comps,unsafe' --mangle --output=$(@)
	@rm -rf $(TEMPFILE_JS)
	@rm -rf $(TEMPFILE)

src/Data: ../../api/v2/openapi.yaml
	-rm -r src/Data
	$(DOCKER_RUN_CURRENT_USER) --rm -v ${PWD}/../..:/local openapitools/openapi-generator-cli:v3.3.4 generate \
		-i /local/api/v2/openapi.yaml \
	  -g elm \
		-o /local/ui/app/$(TEMPOPENAPI)
	# We only want data directory & DateTime package.
	cp -r $(TEMPOPENAPI)/src/Data src/Data
	cp -r $(TEMPOPENAPI)/src/DateTime.elm src/DateTime.elm
	rm -rf $(TEMPOPENAPI)


clean:
	- @rm -rf script.js elm-stuff src/Data src/DateTime.elm openapi-*
	- @if [ ! -z "$(docker images -q $(DOCKER_IMG))" ]; then \
             docker rmi $(DOCKER_IMG); fi
