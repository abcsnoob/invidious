# Builder stage: Build the application using Crystal
FROM crystallang/crystal:1.12.2-alpine AS builder

# Install dependencies needed for building the app
RUN apk add --no-cache sqlite-static yaml-static

# Set the build argument to control release version
ARG release

# Set the working directory
WORKDIR /invidious

# Copy shard configuration files for Crystal
COPY ./shard.yml ./shard.yml
COPY ./shard.lock ./shard.lock

# Install dependencies via Shards (Crystal's package manager)
RUN shards install --production

# Copy source files and other necessary files
COPY ./src/ ./src/
COPY ./scripts/ ./scripts/
COPY ./assets/ ./assets/
COPY ./videojs-dependencies.yml ./videojs-dependencies.yml

# Run tests to check if the app is working correctly
RUN crystal spec --warnings all \
    --link-flags "-lxml2 -llzma"

# Build the Crystal application based on the release argument
RUN --mount=type=cache,target=/root/.cache/crystal if [[ "${release}" == 1 ]] ; then \
        crystal build ./src/invidious.cr \
        --release \
        --static --warnings all \
        --link-flags "-lxml2 -llzma"; \
    else \
        crystal build ./src/invidious.cr \
        --static --warnings all \
        --link-flags "-lxml2 -llzma"; \
    fi

# Final stage: Create a smaller image for running the application
FROM alpine:3.21

# Install necessary runtime dependencies
RUN apk add --no-cache rsvg-convert ttf-opensans tini tzdata

# Set the working directory
WORKDIR /invidious

# Create a non-root user and group
RUN addgroup -g 1000 -S invidious && \
    adduser -u 1000 -S invidious -G invidious

# Copy configuration files and set the default config
COPY --chown=invidious ./config/config.* ./config/
RUN mv -n config/config.example.yml config/config.yml
RUN sed -i 's/host: \(127.0.0.1\|localhost\)/host: invidious-db/' config/config.yml

# Copy SQL files and localization files
COPY ./config/sql/ ./config/sql/
COPY ./locales/ ./locales/

# Copy assets and built application from the builder stage
COPY --from=builder /invidious/assets ./assets/
COPY --from=builder /invidious/invidious .

# Set proper file permissions for assets, config, and locales
RUN chmod o+rX -R ./assets ./config ./locales

# Expose the port for the application to be accessed
EXPOSE 3000

# Set the user to 'invidious' for security
USER invidious

# Use tini to manage the application process and signals
ENTRYPOINT ["/sbin/tini", "--"]

# Define the command to run the application
CMD [ "/invidious/invidious" ]
