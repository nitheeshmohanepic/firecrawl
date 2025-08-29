FROM node:22-slim AS base

ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"

RUN corepack enable
COPY . /app
WORKDIR /app

FROM base AS prod-deps
RUN pnpm install --prod --frozen-lockfile

FROM base AS build
RUN pnpm install && pnpm run build

# Install Go
FROM golang:1.24 AS go-base
COPY sharedLibs/go-html-to-md /app/sharedLibs/go-html-to-md

# Install Go dependencies and build parser lib
RUN cd /app/sharedLibs/go-html-to-md && \
    go mod download && \
    go build -o html-to-markdown.so -buildmode=c-shared html-to-markdown.go && \
    chmod +x html-to-markdown.so

# Install Rust
FROM rust:1-slim AS rust-base
COPY sharedLibs/html-transformer /app/sharedLibs/html-transformer
COPY sharedLibs/pdf-parser /app/sharedLibs/pdf-parser
COPY sharedLibs/crawler /app/sharedLibs/crawler

# Install Rust dependencies and build transformer lib
RUN cd /app/sharedLibs/html-transformer && \
    cargo build --release --locked && \
    chmod +x target/release/libhtml_transformer.so

# Install Rust dependencies and build PDF parser lib
RUN cd /app/sharedLibs/pdf-parser && \
    cargo build --release --locked && \
    chmod +x target/release/libpdf_parser.so

# Install Rust dependencies and build crawler lib
RUN cd /app/sharedLibs/crawler && \
    cargo build --release --locked && \
    chmod +x target/release/libcrawler.so

FROM base
COPY --from=build /app/dist /app/dist
COPY --from=prod-deps /app/node_modules /app/node_modules
COPY --from=go-base /app/sharedLibs/go-html-to-md/html-to-markdown.so /app/sharedLibs/go-html-to-md/html-to-markdown.so
COPY --from=rust-base /app/sharedLibs/html-transformer/target/release/libhtml_transformer.so /app/sharedLibs/html-transformer/target/release/libhtml_transformer.so
COPY --from=rust-base /app/sharedLibs/pdf-parser/target/release/libpdf_parser.so /app/sharedLibs/pdf-parser/target/release/libpdf_parser.so
COPY --from=rust-base /app/sharedLibs/crawler/target/release/libcrawler.so /app/sharedLibs/crawler/target/release/libcrawler.so

# Install git and fix entrypoint script
RUN apt-get update && apt-get install -y git procps && rm -rf /var/lib/apt/lists/* && \
    sed -i 's/\r$//' /app/docker-entrypoint.sh && \
    chown -R node:node /app

# Start the server by default, this can be overwritten at runtime
EXPOSE 8080

# Railway expects the container to run as non-root for security
USER node

ENTRYPOINT "/app/docker-entrypoint.sh"
