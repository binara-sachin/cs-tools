BINARY := server

.PHONY: all run vet test test-db build clean migrate migrate-down seed backfill-meta

all: build

run:
	go run ./cmd/server

vet:
	go vet ./...

# Tests run against a separate "gid_test" database, never the dev database:
# several tests TRUNCATE tables. They read TEST_DATABASE_URL (not
# DATABASE_URL) and internal/testdb refuses any database not ending in _test.
TEST_DB_NAME ?= gid_test
TEST_DB_ADMIN_URL ?= postgres://gid:gid@localhost:5433/postgres?sslmode=disable
TEST_DATABASE_URL ?= postgres://gid:gid@localhost:5433/$(TEST_DB_NAME)?sslmode=disable

# Creates the test database if missing and applies migrations to it.
test-db:
	@psql "$(TEST_DB_ADMIN_URL)" -tAc "SELECT 1 FROM pg_database WHERE datname = '$(TEST_DB_NAME)'" | grep -q 1 \
		|| psql "$(TEST_DB_ADMIN_URL)" -c 'CREATE DATABASE $(TEST_DB_NAME)'
	@migrate -path migrations -database "$(TEST_DATABASE_URL)" up

# -p 1 serializes package test execution: DB-backed tests across packages
# insert raw-SQL fixtures directly into the shared test Postgres (no
# per-run namespacing beyond a hardcoded literal per package), so two
# packages' test binaries running concurrently against the same instance can
# race each other even though no single package's own fixtures collide.
test: vet test-db
	TEST_DATABASE_URL="$(TEST_DATABASE_URL)" go test -race -p 1 ./...

build: test
	go build -o $(BINARY) ./cmd/server

clean:
	rm -f $(BINARY)

# Applies migrations/*.up.sql against DATABASE_URL via golang-migrate.
migrate:
	migrate -path migrations -database "$$DATABASE_URL" up

migrate-down:
	migrate -path migrations -database "$$DATABASE_URL" down 1

# Runs cmd/seed — synthetic fixtures unless GITHUB_TOKEN is set.
seed:
	go run ./cmd/seed

# Runs cmd/backfill-meta — one-time backfill of title/abt_team/opened_by for
# issues rows that predate this feature. Requires GITHUB_TOKEN.
backfill-meta:
	go run ./cmd/backfill-meta
