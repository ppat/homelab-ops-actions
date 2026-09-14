# The build context is the repository root, so this COPY reaches a directory outside the one
# holding this Dockerfile.
FROM alpine:3.21.3@sha256:a8560b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c AS base
ARG GREETING_SUFFIX
COPY ci/test-data/build-docker-image/shared/greeting.txt /greeting.txt
RUN test -n "${GREETING_SUFFIX}" && echo "${GREETING_SUFFIX}" >> /greeting.txt

FROM base AS selected
RUN echo selected > /stage.txt

# Building this last stage fails, so a green build proves `target` selected the stage above.
FROM base AS last
RUN echo "the target input was ignored" && exit 1
