# Project Requirements Document - onedump

Project Requirements Document: onedump  
==================================================  

1. Project Overview and Objectives  
----------------------------------  
Purpose  
• onedump is a configuration-driven CLI tool for automating database backup, restore, binlog management, slow-log analysis, and file synchronization.  
• It targets MySQL (primary), PostgreSQL, and multiple storage backends (local, S3, GDrive, Dropbox, SFTP).  

Objectives  
• Provide an extensible, declarative framework for scheduling and executing database administration jobs.  
• Ensure reliable, repeatable backups and point-in-time restores via binlog support.  
• Enable performance diagnostics through slow-log parsing and reporting.  
• Facilitate cross-platform file transfers with integrity checks.  
• Offer pluggable notification channels to report job status.  

2. Functional Requirements  
--------------------------  
2.1 Command-Line Interface  
• Modular CLI built with Cobra.  
• Subcommands:  
  – dump (mysqldump, pgdump, native MySQL dumper)  
  – binlog (query, restore, sync)  
  – sync (SFTP file sync)  
  – download (S3 downloads)  
  – slow (slow-log parsing)  
• Global flags: config file path, log level, dry-run.  

2.2 Configuration Management  
• YAML-based job definitions (sources, destinations, schedules, notifications).  
• Environment variable overrides.  
• Validation of job schemas before execution.  

2.3 Database Dump & Restore  
• mysqldump and pgdump wrappers.  
• Native MySQL dumper for fine-grained control.  
• SSH tunneling for remote database access.  

2.4 Binlog Management (MySQL)  
• Query binlog positions.  
• Restore from binlogs for point-in-time recovery.  
• Sync binlog files to storage backends.  

2.5 Slow-Log Parsing & Analysis  
• Parse MySQL slow-query logs (plain and gzip).  
• Output metrics: query count, latencies, top N slow queries.  

2.6 File Synchronization  
• Checksum-based file sync engine.  
• Support for:  
  – AWS S3  
  – Google Drive  
  – Dropbox  
  – SFTP (via SSH)  
  – Local filesystem  
• Resume and incremental sync.  

2.7 Notifications  
• Pluggable interface.  
• Built-in: console, Slack.  
• Report job start, progress, success/failure details.  

2.8 Scheduling & Orchestration  
• Job scheduler using gocron.  
• Sequential or parallel job execution per configuration.  

2.9 Logging & Error Handling  
• Structured logging via log/slog.  
• Graceful error propagation and retries for transient failures.  

3. Non-functional Requirements  
------------------------------  
3.1 Reliability  
• Guarantee no data loss during backup and sync operations.  
• Automatic retries on recoverable errors (network, transient DB).  

3.2 Performance  
• Handle large dumps (GB-scale) without excessive memory usage (stream exports).  
• Parallel uploads/downloads where supported by backend.  

3.3 Scalability  
• Add new storage backends and notification channels with minimal code changes.  
• Support multiple concurrent jobs.  

3.4 Security  
• Encrypted connections to DB and SFTP (TLS/SSH).  
• Secure storage of credentials (environment variables, encrypted YAML fields).  

3.5 Usability  
• Clear CLI help and examples.  
• Comprehensive documentation (docs/*.md).  

3.6 Maintainability  
• Modular code structure with well‐defined Go interfaces (Dumper, Storage, Notifier).  
• High unit and integration test coverage (>90%).  

3.7 Portability  
• Docker support for containerized deployment.  
• OS-agnostic code; tested on Linux, macOS, Windows clients for CLI.  

4. Technical Requirements  
-------------------------  
4.1 Languages & Frameworks  
• Go 1.20+  
• Cobra (CLI)  
• gocron (scheduler)  
• log/slog (logging)  

4.2 Libraries & SDKs  
• github.com/go-sql-driver/mysql  
• github.com/lib/pq (PostgreSQL)  
• github.com/aws/aws-sdk-go-v2 (S3)  
• google.golang.org/api/drive/v3 (GDrive)  
• github.com/dropbox/dropbox-sdk-go-unofficial/v6/dropbox  
• github.com/pkg/sftp + golang.org/x/crypto/ssh  

4.3 Tools & Utilities  
• Docker & Docker Compose (testutils/mysql, SFTP server)  
• go-sqlmock (unit tests)  
• goreleaser (build & release pipeline)  
• Make or simple shell scripts for build automation  

4.4 Data & Scripts  
• SQL scripts for time zone table population.  
• Test slow log samples.  
• Docker Compose YAML for integration testing.  

5. Dependencies and Prerequisites  
---------------------------------  
5.1 Runtime  
• Go toolchain (1.20+)  
• Docker (for containerized execution)  
• MySQL client or server access credentials  
• PostgreSQL client (if using pgdump)  
• Cloud credentials: AWS Access Key, GDrive OAuth 2.0 token, Dropbox token  
• SSH key or password for SFTP endpoints  

5.2 Build & Test  
• Docker Compose (for integration tests)  
• Access to a Docker-compatible host  
• Permissions to open network ports (SSH, MySQL) on test machines  

5.3 Configuration  
• Valid YAML config file conforming to CONFIG_REF.md  
• Environment variables for sensitive data (DB passwords, API tokens)  

6. Assumptions and Constraints  
------------------------------  
6.1 Assumptions  
• Databases follow standard MySQL/PostgreSQL schemas and accept mysqldump/pgdump.  
• Network connectivity is available between onedump host and target systems.  
• Users will manage retention policies externally (e.g., S3 lifecycle rules).  

6.2 Constraints  
• Native dumper currently supports only MySQL.  
• Binlog operations are MySQL-specific.  
• Time zone initialization applies only to MySQL system tables.  
• SFTP backends require SSH key/password; no GUI fallback.  
• Performance depends on remote network bandwidth.  

7. Success Criteria  
-------------------  
7.1 Functional Validation  
• All CLI commands execute without errors given valid config.  
• Backups complete and can be restored end-to-end (mysqldump/native/pgdump).  
• Binlog restores achieve point-in-time recovery.  
• Slow-log parser outputs correct metrics for all sample logs.  
• File sync operations upload/download and verify checksums across all backends.  
• Notifications delivered on job success/failure (console & Slack).  

7.2 Non-functional Metrics  
• Test coverage ≥ 90% (unit + integration).  
• Memory footprint for 10 GB dumps < 200 MB (streaming).  
• Average upload throughput within 10% of raw network capacity.  
• Mean time to recover from transient errors < 30 s with exponential backoff.  

7.3 Documentation & Usability  
• README.md provides clear quickstart and examples.  
• CONFIG_REF.md covers all job definitions.  
• docs/ contains end-to-end guides for binlog, sync, slow-log features.  

7.4 Deployment & Automation  
• Docker images build reproducibly via goreleaser.  
• CI pipeline passes on pull requests (lint, tests, build).  
• Versioned releases published to GitHub and Docker Hub.  

7.5 Extensibility  
• Adding a new Storage or Notifier plugin achieved by implementing the interface and minimal wiring in cmd/.  
• Community can follow contribution guide in development.md.  

==================================================  
Prepared for Project Managers and Development Teams to guide implementation, ensure alignment, and define clear delivery and quality targets.

---
*Generated by Git Search API on 2025-08-21 23:42:08 UTC*
