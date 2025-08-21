# App Flow Document - onedump

1. High-Level Architecture Overview  
-------------------------------  
onedump is a CLI-driven, modular Go application that orchestrates database dump/restore, binlog management, slow-log parsing, and file synchronization.  It comprises these layers:  
• CLI Layer (Cobra commands)  
• Config Layer (YAML parsing → `config.Job`)  
• Handler Layer (business logic)  
• Plugin Layers:  
  – Dumper (MySQL/PG dump via exec or native)  
  – Binlog (querier, syncer, restore)  
  – Slow (slow-log parser)  
  – FileSync (checksum + transfer orchestration)  
• Storage Layer (S3, GDrive, Dropbox, SFTP, Local)  
• Notifier Layer (Console, Slack)  
• TestUtils (mock servers, Docker Compose for integration)  

2. Component Interaction Diagram (text)  
---------------------------------------  
  [User CLI]  
      |-- invokes --> [cmd/*.go (Cobra)]  
           |-- loads --> [config.Loader]  
                  |-- builds --> `Job` struct  
           |-- routes --> [handler.JobHandler]  
                  |-- selects --> Dumper / Binlog / Slow / FileSync  
                        |-- uses --> [dumper.Dialer & runner.Exec]  
                        |-- calls --> `mysqldump`/`pg_dump`/native  
                        |-- streams → temporary file / stdout  
                  |-- invokes → [filesync.Checksum]  
                  |-- uploads → [storage.Adapter{S3,GDrive,Dropbox,SFTP,Local}]  
                  |-- records → [jobresult.Result]  
                  |-- notifies → [notifier.Console/Slack]  
  DB Server <– (via SSH tunnel or direct TCP) – dumper.Dialer  
  Storage Backends <– (HTTPS/SFTP) – storage.Adapter  
  Slack API <– (HTTP) – notifier.Slack  

3. Data Flow Description  
------------------------  
1. CLI parse (e.g., `onedump dump --config cfg.yaml --job backup1`).  
2. config.Loader reads YAML → unmarshals into `config.Job`:  
   • Database DSN, dumper type, SSH/Tunnel settings  
   • Storage target, credentials, path prefix  
   • Notifier settings (console, slack webhook)  
3. JobHandler.Dispatch executes job by type:  
   • Dump: calls dumper.Dumper.Dump(ctx, Job)  
   • Binlog Sync: binlog.Syncer.Sync(ctx, Job)  
   • Binlog Restore: binlog.Restorer.Restore(ctx, Job)  
   • Slow: slow.Parser.Parse(ctx, Job)  
   • Download: storage.Adapter.Download(ctx, Job)  
4. Dumper/Dumpershell runs external command via runner.Exec or uses Go native APIs; outputs SQL stream to file or buffer.  
5. FileSync: reads dump file, computes checksum, writes a `.checksum` file.  
6. Storage Adapter uploads both dump and checksum to target backend, using multipart or streaming.  
7. On success/failure, Handler builds `jobresult.Result` containing timestamps, sizes, error messages.  
8. Notifier sends console output and (optionally) posts JSON payload to Slack webhook.  
9. Exit code reflects overall success (0) or failure (>0).  

4. User Journey / Workflow  
--------------------------  
1. Installation & Setup  
   – go install github.com/liweiyi88/onedump@latest  
   – Prepare `config.yaml` per CONFIG_REF.md  
2. Define Jobs in YAML  
   – job.name: “daily-mysql-backup”  
     job.type: dump  
     dumper: mysqldump / native  
     dsn: user:pass@tcp(…)  
     storage:  
       type: s3  
       bucket: my-backups  
       prefix: daily/mysql  
     notifier:  
       slack: webhook_url  
3. Run Backup  
   – `onedump dump --config config.yaml --job daily-mysql-backup`  
   – Observe logs: [handler] → [dumper] → [storage] → [notifier]  
   – Check S3 bucket for `daily/mysql/... .sql` and `.checksum`  
4. Restore from Binlog  
   – `onedump binlog restore --config config.yaml --start-file mysql-bin.000123 --start-pos 4567 --end-file mysql-bin.000125 --end-pos 0 --target-dsn …`  
   – Application pulls binlog files, pipes them into mysql client.  
5. Analyze Slow Logs  
   – `onedump slow --config config.yaml --job slow-report`  
   – Parser ingests local/remote log, outputs summary to STDOUT or file.  
6. Sync to SFTP  
   – `onedump sync --config config.yaml --job sftp-sync`  
   – Handler uses filesync + sftp adapter.  

5. “API” Endpoints and Interactions  
-----------------------------------  
(Here “API” = CLI commands and plugin interfaces)  

A. CLI Commands (via Cobra)  
   1. onedump dump  
      Flags: --config, --job  
      Calls: handler.DumpHandler.Handle()  
   2. onedump binlog syncs3 / syncsftp  
      Flags: --config, --job, optional since/until  
      Calls: binlog.Syncer.Sync()  
   3. onedump binlog restore  
      Flags: --config, --start-file, --start-pos, --end-file, --end-pos, --target-dsn  
      Calls: binlog.Restorer.Restore()  
   4. onedump download  
      Flags: --config, --job, --output  
      Calls: storage.Adapter.Download()  
   5. onedump slow  
      Flags: --config, --job  
      Calls: slow.Parser.Parse()  
   6. onedump sync (sftp)  
      Flags: --config, --job  
      Calls: synccmd/sftp.go → handler  

B. Plugin Interfaces  
   1. type dumper.Dumper interface {  
        Dump(ctx, cfg Job) (io.Reader, error)  
      }  
   2. type storage.Storage interface {  
        Upload(ctx, path string, r io.Reader) error  
        Download(ctx, path, dest string) error  
      }  
   3. type notifier.Notifier interface {  
        Notify(ctx, Result) error  
      }  
   4. type binlog.Querier interface {  
        GetLatestPos(ctx, DSN) (File, Pos, error)  
      }  
   5. type binlog.Syncer interface {  
        Sync(ctx, cfg Job) error  
      }  

6. Configuration Schema Overview  
--------------------------------  
Defined in config/job.go → struct Job:  
• Name          string  
• Type          string [“dump”|“binlog”|“slow”|“sync”|“download”]  
• DumperConfig:  
    – Type           string [mysqldump|pgdump|mysql-native]  
    – DSN            string  
    – SSH?           { Host, Port, User, KeyFile }  
    – ExtraArgs      []string  
• BinlogConfig?  
    – SyncSince      time or GTID  
    – RestoreRange   {StartFile, StartPos, EndFile, EndPos}  
• SlowLogConfig?  
    – Path           string  
    – Filters        {MinTime, DB, Host}  
• StorageConfig:  
    – Type           [s3|gdrive|dropbox|sftp|local]  
    – S3: {Bucket, Region, Prefix, Credentials}  
    – GDrive: {FolderID, CredentialsJSON}  
    – Dropbox: {Token, Path}  
    – SFTP: {Host, Port, User, Pass, KeyFile, RemoteDir}  
    – Local: {Dir}  
• NotifierConfig:  
    – Console: bool  
    – Slack: {WebhookURL, Channel, Username, IconEmoji}  
• RetryPolicy? {MaxRetries, BackoffSec}  

7. External Service Integrations  
--------------------------------  
• MySQL / PostgreSQL Servers  
  – Connection via `database/sql` or exec of client binaries  
  – Optional SSH tunneling: `dumper/dialer/ssh.go`  
• AWS S3  
  – SDK v2, supports multipart, region, credentials from env or config  
• Google Drive  
  – OAuth2 JSON, Drive API v3, token caching  
• Dropbox  
  – HTTP API v2, bearer token  
• SFTP  
  – golang.org/x/crypto/ssh + pkg/sftp  
• Slack  
  – Incoming Webhook JSON POST  
• Local FS  
  – Standard `os` operations  
• Docker Compose (testutils)  
  – For MySQL + binlog retrieval tests  
  – Custom SFTP server (testutils/sftp.go)  
• go-sqlmock  
  – Unit tests for SQL interactions  

8. Error Handling and Fallbacks  
-------------------------------  
• Config Loader  
  – Missing file → fatal exit(1) with “config not found”  
  – Validation errors → lists missing fields, exit(2)  
• Handler  
  – Wraps all errors with context: `fmt.Errorf("dumper failed: %w", err)`  
  – Aggregates multiple errors (e.g., upload + notify)  
  – Returns non-zero exit codes on any failure  
• Dumper  
  – Connection failure → retry once (configurable), then error  
  – Exec failure → capture stderr, wrap as error  
• Binlog Sync/Restore  
  – If single file fails → continue next (sync), then surface summary error  
  – Restore: abort on first error, cleanup temp files  
• FileSync  
  – Checksum mismatch → abort upload, delete corrupted remote file, error  
  – Upload failure → exponential backoff up to MaxRetries, then error  
• Storage  
  – S3: retry on 5xx, respect SDK retry policy  
  – GDrive/Dropbox: handle rate limits (429) with backoff  
  – SFTP: reconnect on SSH EOF/Error, up to N times  
• Notifier  
  – Console always succeeds (writes to stdout)  
  – Slack: on HTTP failure → log error locally, do not block main job (fail-fast optional)  
• Global Fallbacks  
  – On critical failure, Handler may trigger cleanup hook (e.g., delete partial uploads)  
  – Temp files are created in OS temp dir, always removed in defer  
  – Exit code mapping:  
    – 0: all succeeded  
    – 1: config/arg error  
    – 2: execution error (dump/binlog/slow)  
    – 3: storage error  
    – 4: notifier error (non-fatal by default)  

---
*Generated by Git Search API on 2025-08-21 23:42:08 UTC*
