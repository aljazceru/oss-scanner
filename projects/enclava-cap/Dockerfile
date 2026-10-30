FROM rust:1.88-bookworm@sha256:af306cfa71d987911a781c37b59d7d67d934f49684058f96cf72079c3626bfe0

RUN apt-get update && apt-get install -y --no-install-recommends \
      clang libclang-dev libcryptsetup-dev libssl-dev pkg-config cmake \
      postgresql postgresql-client python3 \
    && rm -rf /var/lib/apt/lists/*

# Database fixtures create isolated databases and exercise PostgreSQL system catalogs.
RUN pg_ctlcluster 15 main start \
    && runuser -u postgres -- psql -v ON_ERROR_STOP=1 -c "CREATE ROLE test WITH LOGIN SUPERUSER PASSWORD 'test'" \
    && runuser -u postgres -- createdb -O test test \
    && pg_ctlcluster 15 main stop

ENV DATABASE_URL=postgresql://test:test@localhost:5432/test \
    CARGO_BUILD_JOBS=2

WORKDIR /src
COPY . .
RUN cargo fetch --locked
ENV CARGO_NET_OFFLINE=true
RUN cargo build --locked --workspace --all-targets
