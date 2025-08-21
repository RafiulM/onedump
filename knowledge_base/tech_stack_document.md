# Tech Stack Document - onedump

Technology Stack Document for onedump  
======================================

This document outlines the major technologies, tools, and services used in the onedump repository. Wherever version information is available (from go.mod, GitHub config files, and Dockerfiles), it is called out. Each entry includes a brief description of its role in the project.  

1. Programming Languages and Versions  
--------------------------------------  
• Go (golang) – version “go 1.21” (as declared in go.mod)  
  • Primary language for all modules.  
  • Provides static typing, fast compilation, cross-compilation support, built-in concurrency (goroutines), and a rich standard library.  

2. Frameworks and Libraries  
-----------------------------  
• Cobra v1.x (github.com/spf13/cobra)  
  • CLI framework that defines commands, flags, help text, and subcommands under the cmd/ directory.  
• YAML parser (gopkg.in/yaml.v3 or v2)  
  • Used in config/ to unmarshal job definitions and storage configurations from YAML files.  
• gocron v1.x (github.com/go-co-op/gocron)  
  • Lightweight job scheduler for recurring tasks (e.g., scheduled dumps or batch sync).  
• AWS SDK for Go v1.x (github.com/aws/aws-sdk-go)  
  • Interfaces with Amazon S3 for upload/download of backups and binlogs.  
• Dropbox SDK v6.x (github.com/dropbox/dropbox-sdk-go-unofficial/dropbox)  
  • Communicates with Dropbox API to store and retrieve files.  
• Google Drive API v3 (google.golang.org/api/drive/v3)  
  • Facilitates file operations on Google Drive.  
• SFTP client (github.com/pkg/sftp v1.x) and SSH (golang.org/x/crypto/ssh v0.x)  
  • Implements secure file transfer over SSH tunnels in synccmd and storage/sftp.  
• go-sql-driver/mysql v1.x (github.com/go-sql-driver/mysql)  
  • Native MySQL driver for direct connections (used by native dumper and binlog querier).  
• pgx or database/sql + lib/pq (github.com/lib/pq)  
  • PostgreSQL driver used by pgdump wrapper.  
• go-sqlmock v1.x (github.com/DATA-DOG/go-sqlmock)  
  • Mocks database interactions in unit tests for dumper, binlog, and jobhandler.  
• testify/assert v1.x (github.com/stretchr/testify)  
  • Simplifies test assertions in many _test.go files.  

3. Database Technologies  
-------------------------  
• MySQL (5.7, 8.0)  
  • Backup/restore via mysqldump; native dumping and binlog querying for point-in-time recovery.  
  • Slow-query log parsing to identify performance bottlenecks.  
  • Time zone table initialization (SQL scripts provided under testutils).  
• PostgreSQL (9.x, 10.x, 11.x+)  
  • Supported via pgdump wrapper for generic database dumps.  

4. Infrastructure and Deployment Tools  
---------------------------------------  
• Docker (Engine v24+, Dockerfile & Dockerfile.goreleaser)  
  • Containerizes onedump CLI for consistent runtime environments.  
• Docker Compose (v2.x, testutils/docker/docker-compose.yml)  
  • Spins up MySQL instances for integration tests (binlog and dump workflows).  
• Goreleaser v1.x (.goreleaser.yml)  
  • Automates building binaries for multiple platforms, packaging, and publishes releases.  
• entrypoint.sh (Bash)  
  • Script that initializes configuration and invokes the CLI within containers.  

5. Development and Build Tools  
-------------------------------  
• Go toolchain (go build, go mod)  
  • Dependency management, module versioning, builds, and cross-compilation.  
• go fmt / goimports  
  • Enforces code formatting and import organization (assumed via CI checks).  
• GolangCI-Lint (optional)  
  • Likely used for static analysis (not committed but common in Go projects).  

6. Testing Frameworks and Tools  
--------------------------------  
• go test (standard testing package)  
  • Runs all unit and integration tests defined under `*_test.go`.  
• go-sqlmock  
  • Enables in-memory mocking of SQL drivers for dumper, syncer, and binlog unit tests.  
• testify/assert  
  • Fluent assertions for test results.  
• Testutils Docker Compose  
  • Real MySQL servers, sample slow logs, and SSH servers to validate end-to-end flows.  
• Custom SFTP server (in testutils)  
  • Lightweight Go implementation to test SFTP sync without external dependencies.  

7. Monitoring and Observability Tools  
--------------------------------------  
• Go’s standard log/slog (Go 1.21 standard library)  
  • Structured logging with levels, timestamps, and context fields.  
• Codecov (codecov.yml)  
  • Aggregates Go test coverage reports from CI runs for visibility.  
• (Optional) Integration with external log aggregators  
  • Given structured logs, teams can wire up to ELK, Splunk, or CloudWatch.  

8. Third-party Services and APIs  
----------------------------------  
• Amazon S3 (via AWS SDK)  
  • Long-term storage of dumps, binlogs, and slow-log archives.  
• Dropbox API  
  • Alternative cloud storage backend.  
• Google Drive API  
  • Another storage option for small or user-managed repositories.  
• Slack Incoming Webhooks (notifier/slack)  
  • Sends job start/stop notifications and alerts to team channels.  
• SFTP over SSH  
  • Secure, on-prem or partner file exchange without cloud vendor lock-in.  

9. Security Tools and Practices  
--------------------------------  
• SSH (golang.org/x/crypto/ssh)  
  • Encrypted tunnels for database and file transfers; key-based auth.  
• TLS/SSL for MySQL connections  
  • Configurable via YAML to enforce encrypted database traffic.  
• Environment-driven secrets (env/env.go)  
  • Avoids hard-coding credentials; supports Docker/Kubernetes secret injection.  
• Minimal IAM permissions (AWS IAM roles/policies)  
  • Users are expected to grant only S3 PutObject/GetObject as needed.  
• Go’s built-in memory safety  
  • No manual memory management, reducing buffer-overflow risks.  

10. DevOps and CI/CD Tools  
---------------------------  
• GitHub Actions (implied by codecov.yml & absence of other CI configs)  
  • Runs unit tests, builds binaries, and pushes coverage reports on PRs.  
• Goreleaser  
  • Integrates with GitHub Actions to build cross-platform releases and publish on GitHub.  
• Codecov  
  • Checks coverage thresholds; fails CI on coverage drop.  
• Docker Hub (or GitHub Container Registry)  
  • Houses official onedump container images for deployments.  
• .gitignore & .goreleaser.yml  
  • Define artifacts to exclude from source control and release pipelines.  

— End of Technology Stack Document —

---
*Generated by Git Search API on 2025-08-21 23:42:08 UTC*
