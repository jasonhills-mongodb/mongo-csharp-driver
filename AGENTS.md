# AGENTS.md - CSharpDriver

## Overview
The C# driver for MongoDB.

## Tech Stack
- .NET library projects producing NuGet packages
- Multi-targeted to various .NET versions from .NET Framework 4.7.2 up
- xUnit + FluentAssertions for testing

## Project Structure
- `src/MongoDB.Bson/` - BSON for MongoDB
- `src/MongoDB.Driver/` - C# driver for MongoDB
- `src/MongoDB.Driver.Encryption/` - Client encryption (CSFLE with KMS).
- `src/MongoDB.Driver.Authentication.AWS/` - AWS IAM authentication
- `tests/MongoDB.Driver.Tests/` - Main C# driver tests
- `tests/MongoDB.Bson.Tests/` - BSON handling tests
- `tests/*/TestHelpers` - Common test utilities
- `tests/*` - Specialized tests; less common
- `tests/MongoDB.Driver.Tests/Specifications/` are JSON-driven tests using a common runner.

## Editing
- Be careful to preserve file BOMs.

## Commands
- Build: `dotnet build CSharpDriver.sln`
- Run all tests: `dotnet test tests/MongoDB.Driver.Tests/MongoDB.Driver.Tests.csproj -f net10.0`
- Run a single test class: `dotnet test tests/MongoDB.Driver.Tests/MongoDB.Driver.Tests.csproj -f net10.0 --filter "FullyQualifiedName~ClassName"`

## Testing
- Tests cannot be run in parallel.
- A MongoDB connection is always available locally, so "integration" tests can be run as well as unit tests. Some test suites also require additional environment variables — if you need to run those tests and the variables are not set, stop and tell the user which variables are needed rather than working around it.

| Feature area | Required environment variables |
|---|---|
| Atlas Search | `ATLAS_SEARCH_TESTS_ENABLED`, `ATLAS_SEARCH_URI` |
| CSFLE / auto-encryption | `CRYPT_SHARED_LIB_PATH` |
| CSFLE with KMS mock servers | `KMS_MOCK_SERVERS_ENABLED` |
| CSFLE with AWS KMS | `CSFLE_AWS_TEMPORARY_CREDS_ENABLED` |
| CSFLE with Azure KMS | `CSFLE_AZURE_KMS_TESTS_ENABLED` |
| CSFLE with GCP KMS | `CSFLE_GCP_KMS_TESTS_ENABLED` |
| AWS authentication | `AWS_TESTS_ENABLED` |
| GSSAPI / Kerberos | `GSSAPI_TESTS_ENABLED`, `AUTH_HOST`, `AUTH_GSSAPI` |
| OIDC authentication | `OIDC_ENV` |
| X.509 authentication | `MONGO_X509_CLIENT_CERTIFICATE_PATH`, `MONGO_X509_CLIENT_CERTIFICATE_PASSWORD` |
| PLAIN authentication | `PLAIN_AUTH_TESTS_ENABLED` |
| SOCKS5 proxy | `SOCKS5_PROXY_SERVERS_ENABLED` |

## Commit and PR Conventions

- The first commit message and the PR message start with a JIRA number: `CSHARP-1234: Description`
- The branch name will usually match the JIRA number: `CSHARP-1234`

## Cursor Cloud specific instructions

The startup update script runs `dotnet restore CSharpDriver.sln`. The .NET 10 SDK (plus 6.0/3.1 runtimes) is pre-installed in `$HOME/.dotnet` and on `PATH` via `~/.bashrc`; standard `dotnet build`/`dotnet test` commands (see `## Commands`) work as-is.

- MongoDB is installed (`mongodb-org` 8.0) but there is no systemd, so `mongod` must be started manually and is NOT auto-started by the update script. Start it as a single-node replica set with test commands enabled (data dir `/data/db` persists in the snapshot, including the replica-set config, so re-running just re-elects the primary):
  ```
  mongod --dbpath /data/db --bind_ip 127.0.0.1 --port 27017 --replSet rs0 --setParameter enableTestCommands=1
  ```
  On a brand-new data dir only, initiate once: `mongosh --eval 'rs.initiate({_id:"rs0",members:[{_id:0,host:"127.0.0.1:27017"}]})'`.
- The replica set + `enableTestCommands=1` matter: many integration tests use `configureFailPoint`/`failCommand` and write-concern-error behavior that a bare standalone `mongod` rejects (`CommandNotFound`). A plain standalone will show spurious failures in suites like `CrudProseTests`.
- Building `MongoDB.Driver.Encryption` downloads and gpg-verifies native `libmongocrypt` binaries from GitHub at build time (needs network + `gpg`/`tar`, both present). The `gpg: This key is not certified with a trusted signature!` warning during build is expected and harmless.
- The default test category is `Integration`, so the standard test commands require a running `mongod`. For a server-free run, filter with `--filter "Category!=Integration"` (see `evergreen/run-unit-tests.sh`).
- There is no separate lint step: C# code analysis runs via the `*.ruleset` files during `dotnet build`. CI `pr.yml` only validates PR title/label metadata.
