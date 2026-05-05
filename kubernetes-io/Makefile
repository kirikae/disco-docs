IMAGE_NAME     ?= k8s-offline-docs
IMAGE_TAG      ?= local
K8S_REPO       := https://github.com/kubernetes/website
K8S_DIR        := k8s-website
PORT           ?= 8080
CONTAINER_NAME ?= k8s-offline-docs

.PHONY: all clone build run stop clean

all: clone build run

clone:
	@if [ -d "$(K8S_DIR)/.git" ]; then \
		echo "==> Updating kubernetes/website..."; \
		git -C $(K8S_DIR) fetch --all; \
		git -C $(K8S_DIR) checkout main; \
		git -C $(K8S_DIR) pull; \
		git -C $(K8S_DIR) submodule update --init --recursive; \
	else \
		echo "==> Cloning kubernetes/website (this will take a while)..."; \
		git clone --recurse-submodules $(K8S_REPO) $(K8S_DIR); \
	fi

build:
	@test -d "$(K8S_DIR)/.git" || (echo "ERROR: Run 'make clone' first" && exit 1)
	@OLD_VERSIONS=$$(git -C $(K8S_DIR) ls-remote --heads origin \
		| grep -o 'refs/heads/release-1\.[0-9]*' \
		| sed 's|refs/heads/||' \
		| sort -uV | tail -n 5 | tr '\n' ' ' | xargs) && \
	COMMIT_SHA=$$(git -C $(K8S_DIR) rev-parse --short HEAD) && \
	echo "==> Building with OLD_VERSIONS=$$OLD_VERSIONS (k8s-website@$$COMMIT_SHA)" && \
	docker build \
		--build-arg OLD_VERSIONS="$$OLD_VERSIONS" \
		-t $(IMAGE_NAME):$(IMAGE_TAG) \
		-t $(IMAGE_NAME):$$COMMIT_SHA \
		.

run:
	@docker rm -f $(CONTAINER_NAME) 2>/dev/null || true
	@docker run -d \
		--name $(CONTAINER_NAME) \
		-p $(PORT):80 \
		$(IMAGE_NAME):$(IMAGE_TAG)
	@echo "==> Serving at http://localhost:$(PORT)"

stop:
	@docker rm -f $(CONTAINER_NAME) 2>/dev/null || true

clean: stop
	@docker rmi -f $(IMAGE_NAME):$(IMAGE_TAG) 2>/dev/null || true
	@rm -rf $(K8S_DIR)
