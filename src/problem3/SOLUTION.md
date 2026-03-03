# PROBLEM 3

## Group 1: Log & File System Issues

### 1. Log Flooding
* **Signs:** `/var/log/nginx/access.log` or `error.log` size reaches tens of gigabytes.
* **Backup & Fix:**
    * **Step 1 (Backup):** Compress the log to an external partition or stream it to a backup server. If space permits:
        `gzip -c /var/log/nginx/access.log > /var/log/nginx/access.log.$(date +%F).gz`
    * **Step 2 (Purge):** `cat /dev/null > /var/log/nginx/access.log` (Clears content while preserving the file handle).
* **Prevention:** Configure strict `logrotate` policies.

### 2. Unlinked Files (Deleted but Still Open)
* **Signs:** `df -h` shows 99% usage, but `du -sh` shows significant free space.
* **Backup & Fix:**
    * **Step 1 (Recovery):** Since the file is logically deleted but still held by Nginx, you can recover the data from the file descriptor if needed:
        `cp /proc/[NGINX_PID]/fd/[FD_NUMBER] /path/to/recovered_backup.log`
    * **Step 2 (Release):** `sudo systemctl reload nginx` (Forces Nginx to release the deleted file handle).
* **Verification:** `sudo lsof +L1` or `sudo lsof | grep deleted`.

### 3. Inode Exhaustion
* **Signs:** `df -h` shows free space, but the system returns "No space left on device" errors.
* **Backup & Fix:**
    * **Step 1 (Backup):** Zipping millions of tiny files locally will fail if disk is at 99%. Use `tar` and pipe it directly to a remote server:
        `tar -czf - /path/to/small_files | ssh user@backup-server "cat > archive.tar.gz"`
    * **Step 2 (Purge):** Once the transfer is verified, delete the source files.
* **Verification:** `df -i`.

---

## Group 2: Nginx Configuration Issues

### 4. "Bottomless" Cache (Missing `max_size`)
* **Signs:** The `proxy_cache_path` directory grows until it fills the entire disk.
* **Backup & Fix:**
    * **Backup:** Backup the Nginx configuration before editing: `cp nginx.conf nginx.conf.$(date +%F).bak`.
    * **Fix:** Add `max_size=10g` to the `proxy_cache_path` directive.
    * **Cleanup:** Clear old cache only after verifying if any specific cached data needs to be preserved for analysis.

### 5. Client Body Buffering (Disk Offloading)
* **Signs:** The `/var/lib/nginx/body` directory fills with large temporary files.
* **Backup & Fix:**
    * **Fix:** Increase `client_body_buffer_size` or mount the temp directory to a RAM Disk (`tmpfs`).
    * **Backup:** If these temp files represent failed large uploads, zip them before purging for developer troubleshooting:
        `zip -r upload_samples.zip /var/lib/nginx/body`

### 6. Debug Mode & Excessive Logging
* **Signs:** Logs contain full Request Bodies, Cookies, or detailed traces.
* **Backup & Fix:**
    * **Action:** Before reverting log level to `warn` or `error`, ensure the current `debug` log is zipped and archived, as it contains critical trace data for the current incident.

---

## Group 3: System & OS Issues

### 7. Systemd Journald & Core Dumps
* **Signs:** `/var/lib/systemd/coredump/` is full.
* **Backup & Fix:**
    * **Backup:** Core dumps are vital for RCA (Root Cause Analysis). Zip them before deletion:
        `zip /backup/coredumps_$(date +%F).zip /var/lib/systemd/coredump/*`
    * **Journal Fix:** Export priority logs before vacuuming:
        `journalctl -u nginx > nginx_priority_logs.txt && gzip nginx_priority_logs.txt`.
    * **Vacuum:** `sudo journalctl --vacuum-size=1G`.

### 8. APT & Snap Cache
* **Signs:** `/var/cache/apt/archives` occupies several GBs.
* **Backup & Fix:**
    * **Action:** For offline/air-gapped servers, backup the `.deb` files to a network drive before running `sudo apt clean`. If the server is online, purging is safe as packages can be re-downloaded.
    * **Snap Fix:** Limit Snap retention: `sudo snap set system refresh.retain=2`.

---

## Long-Term Strategy: ISO/IEC 27001 Compliance

To ensure data integrity and prevent storage exhaustion according to international security standards, we implement a **Tiered Log Management Strategy**.

### 1. Tiered Retention Logic
* **Hot Storage (Days 0-30):** Logs are stored locally on the VM, compressed via `logrotate` (`.gz` format) for immediate incident response and audit (ISO requirement).
* **Cold Storage (Day 31+):** Logs are archived, timestamped, and shipped to Cloud Storage (AWS S3, Azure Blob).

### 2. Automated "Zip-Ship-Purge" (Cron Job)
We use a "Safety-First" automation logic:
1.  **Zip & Hash:** Compress logs and generate a SHA256 checksum (to ensure Non-repudiation for ISO audit).
2.  **Ship:** Upload the archive to Cloud Storage.
3.  **Verify:** The script verifies the Cloud file's existence and size.
4.  **Purge:** Local logs are only deleted or truncated after a `200 OK` confirmation from the Cloud Storage API.

### 3. Cloud Lifecycle Management
* Enable **Lifecycle Rules** on the Cloud Bucket to move logs to **Archive/Glacier Tier** after 90 days. This optimizes costs while maintaining 1-2 years of retention required by compliance policies.
