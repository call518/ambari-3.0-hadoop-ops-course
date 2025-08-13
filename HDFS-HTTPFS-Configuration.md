# HDFS HTTPFS Configuration Guide

This document provides step-by-step instructions for configuring and starting HDFS HTTPFS service on the Ambari cluster deployed using the included docker-compose.yml.

## Overview

HTTPFS (HTTP FileSystem) is a server that provides a REST HTTP gateway supporting all HDFS File System operations (read and write). It acts as a proxy server for HDFS, allowing HTTP clients to interact with HDFS without requiring native HDFS client libraries.

## Prerequisites

- Ambari cluster running with HDFS service installed
- Access to Ambari Web UI (default: http://localhost:8080)
- SSH access to the cluster nodes
- HTTPFS port 14000 exposed (already configured in docker-compose.yml)

## Configuration Steps

### Step 1: Configure Proxy User Settings in Ambari

1. Access the Ambari Web UI at `http://localhost:8080`
2. Navigate to **HDFS** service
3. Click on **Configs** tab
4. Find the **Custom core-site** section
5. Add the following properties to `core-site.xml`:

```xml
<property>
  <name>hadoop.proxyuser.httpfs.hosts</name>
  <value>*</value>
  <description>Hosts from which httpfs user can connect</description>
</property>

<property>
  <name>hadoop.proxyuser.httpfs.groups</name>
  <value>*</value>
  <description>Groups that httpfs user can impersonate</description>
</property>
```

6. Click **Save** to save the configuration changes
7. Add a meaningful comment for the configuration change

### Step 2: Restart HDFS Service

1. In the Ambari Web UI, navigate to the **HDFS** service
2. Click on **Service Actions** dropdown menu
3. Select **Restart All** to restart the HDFS service
4. Confirm the restart operation
5. Wait for the service to restart successfully

### Step 3: Configure and Start HTTPFS Service

Connect to the Ambari cluster container and execute the following commands:

```bash
# Access the cluster container
docker-compose exec -it bigtop-hostname0 /bin/bash
```

#### 3.1 Create Symbolic Link for HDFS Command

Create a symbolic link to ensure proper HDFS command access:

```bash
ln -s /usr/bigtop/3.3.0/usr/lib/hadoop-hdfs/bin/hdfs /usr/bigtop/3.3.0/usr/lib/hadoop/bin/hdfs
```

#### 3.2 Start HTTPFS Service

Start the HTTPFS service using the following command:

```bash
/usr/bigtop/3.3.0/usr/lib/hadoop/sbin/httpfs.sh start
```

### Step 4: Verify HTTPFS Service

Verify that the HTTPFS service is running and listening on port 14000:

```bash
lsof -Pni :14000
```

Expected output should show:
```
COMMAND     PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
java    1175610 root  350u  IPv4 34098303      0t0  TCP *:14000 (LISTEN)
```

## Testing HTTPFS Service

Once HTTPFS is running, you can test it using REST API calls:

### Basic Operations

#### List Root Directory
```bash
curl -i "http://localhost:14000/webhdfs/v1/?op=LISTSTATUS"
```

#### Get Home Directory Status
```bash
curl -i "http://localhost:14000/webhdfs/v1/user?op=LISTSTATUS"
```

#### Create a Directory
```bash
curl -i -X PUT "http://localhost:14000/webhdfs/v1/user/test?op=MKDIRS"
```

### File Upload and Download Tutorial

This section demonstrates how to upload and download files using HTTPFS REST API with practical examples.

#### Creating a Test File

First, create a test file for upload testing:

```bash
# Create a 1GB test file
dd if=/dev/zero of=test-1g.img bs=1M count=1024
```

#### File Upload (PUT) Operations

There are two approaches for uploading files via HTTPFS:

##### Method 1: Two-Step PUT Process

This method involves two separate requests - first to get the redirect location, then to upload the actual data.

**Step 1: Get redirect location**
```bash
curl -X PUT -i -c ~/.httpfsauth "http://localhost:14000/webhdfs/v1/tmp/test.txt?user.name=hdfs&op=CREATE&overwrite=true"
```

Expected response:
```
HTTP/1.1 307 Temporary Redirect
Date: Wed, 13 Aug 2025 22:38:29 GMT
Cache-Control: no-cache
Set-Cookie: hadoop.auth="u=hdfs&p=hdfs&t=simple-dt&e=1755160709078&s=bcctcZbg/kwFSTosXaC+6WDVmouIGayyv4n4StvJcGA="; Path=/; HttpOnly
Location: http://localhost:14000/webhdfs/v1/tmp/test.txt?op=CREATE&data=true&user.name=hdfs&overwrite=true
Content-Type: application/json;charset=utf-8
Content-Length: 0
```

**Step 2: Upload the actual file data**
```bash
curl -X PUT -i -b ~/.httpfsauth -T ./test-1g.img -H "Content-Type:application/octet-stream" "http://localhost:14000/webhdfs/v1/tmp/test.txt?op=CREATE&data=true&user.name=hdfs&overwrite=true"
```

Expected response:
```
HTTP/1.1 100 Continue

HTTP/1.1 201 Created
Date: Wed, 13 Aug 2025 22:50:07 GMT
Location: http://localhost:14000/webhdfs/v1/tmp/test.txt
Content-Type: application/json
Transfer-Encoding: chunked

{"Location":"http://localhost:14000/webhdfs/v1/tmp/test.txt"}
```

##### Method 2: Single-Step PUT Process (with Auto-redirect)

This method handles the redirect automatically in a single command:

```bash
curl -v -L -X PUT -T ./test-1g.img -c ~/.httpfsauth -b ~/.httpfsauth -H "Content-Type:application/octet-stream" "http://localhost:14000/webhdfs/v1/tmp/test-1g.img?user.name=hdfs&op=CREATE&overwrite=true&createparent=true"
```

**Parameters explained:**
- `-v`: Verbose output for debugging
- `-L`: Follow redirects automatically
- `-T ./test-1g.img`: Upload the specified file
- `-c ~/.httpfsauth`: Save cookies to file
- `-b ~/.httpfsauth`: Send cookies from file
- `overwrite=true`: Overwrite existing file if present
- `createparent=true`: Create parent directories if they don't exist

#### File Download (GET) Operation

Download files from HDFS using the OPEN operation:

```bash
curl -i -L "http://localhost:14000/webhdfs/v1/tmp/test-1g.img?user.name=hdfs&op=OPEN" -o ./downloaded-test-1g.img
```

**Parameters explained:**
- `-i`: Include HTTP headers in output
- `-L`: Follow redirects automatically
- `-o ./downloaded-test-1g.img`: Save downloaded content to specified file
- `op=OPEN`: WebHDFS operation to read/download file

#### Verification Commands

After upload/download operations, verify the results:

**Check file exists in HDFS:**
```bash
curl -i "http://localhost:14000/webhdfs/v1/tmp/test-1g.img?user.name=hdfs&op=GETFILESTATUS"
```

**List directory contents:**
```bash
curl -i "http://localhost:14000/webhdfs/v1/tmp?user.name=hdfs&op=LISTSTATUS"
```

**Compare file sizes (local verification):**
```bash
# Check original file size
ls -lh test-1g.img

# Check downloaded file size
ls -lh downloaded-test-1g.img

# Verify file integrity using checksums
md5sum test-1g.img downloaded-test-1g.img
```

#### Common Upload/Download Parameters

| Parameter | Description | Example Values |
|-----------|-------------|----------------|
| `user.name` | User identity for the operation | `hdfs`, `admin`, `user1` |
| `op` | WebHDFS operation type | `CREATE`, `OPEN`, `GETFILESTATUS` |
| `overwrite` | Overwrite existing files | `true`, `false` |
| `createparent` | Create parent directories | `true`, `false` |
| `replication` | HDFS replication factor | `1`, `2`, `3` |
| `blocksize` | HDFS block size in bytes | `134217728` (128MB) |

#### Error Handling

Common error scenarios and solutions:

**Permission Denied (403):**
```bash
# Ensure proper user.name parameter
curl -i "http://localhost:14000/webhdfs/v1/tmp/test.txt?user.name=hdfs&op=CREATE"
```

**File Already Exists (403):**
```bash
# Add overwrite=true parameter
curl -i -X PUT "http://localhost:14000/webhdfs/v1/tmp/test.txt?user.name=hdfs&op=CREATE&overwrite=true"
```

**Parent Directory Not Found (404):**
```bash
# Add createparent=true parameter
curl -i -X PUT "http://localhost:14000/webhdfs/v1/deep/path/test.txt?user.name=hdfs&op=CREATE&createparent=true"
```

## Service Management

### Start HTTPFS
```bash
/usr/bigtop/3.3.0/usr/lib/hadoop/sbin/httpfs.sh start
```

### Stop HTTPFS
```bash
/usr/bigtop/3.3.0/usr/lib/hadoop/sbin/httpfs.sh stop
```

### Check HTTPFS Status
```bash
/usr/bigtop/3.3.0/usr/lib/hadoop/sbin/httpfs.sh status
```

## Configuration Files

Key configuration files for HTTPFS:

- **Core configuration**: `/usr/bigtop/3.3.0/etc/hadoop/conf/core-site.xml`
- **HTTPFS configuration**: `/usr/bigtop/3.3.0/etc/hadoop-httpfs/conf/httpfs-site.xml`
- **HTTPFS logs**: `/usr/bigtop/3.3.0/var/log/hadoop-httpfs/`

## Port Information

| Service | Port | Description |
|---------|------|-------------|
| HTTPFS | 14000 | HTTP REST API endpoint for HDFS operations |
| Ambari Server | 8080 | Ambari Web UI |
| HDFS NameNode | 50070 | NameNode Web UI (mapped to 10001) |

## Troubleshooting

### Common Issues

1. **Port 14000 not accessible**
   - Check if the docker-compose.yml has port mapping: `"14000:14000"`
   - Verify firewall settings
   - Ensure HTTPFS service is running

2. **Permission denied errors**
   - Verify proxy user settings in core-site.xml
   - Ensure HDFS service was restarted after configuration changes

3. **Service fails to start**
   - Check HTTPFS logs in `/usr/bigtop/3.3.0/var/log/hadoop-httpfs/`
   - Verify HDFS service is running and healthy
   - Check for port conflicts

### Log Files

Monitor HTTPFS logs for troubleshooting:
```bash
tail -f /usr/bigtop/3.3.0/var/log/hadoop-httpfs/httpfs.log
```

## Security Considerations

⚠️ **Warning**: The current configuration uses wildcard (*) values for proxy user hosts and groups, which allows access from any host and any group. In production environments, you should:

1. Specify explicit hostnames instead of `*` for `hadoop.proxyuser.httpfs.hosts`
2. Specify explicit groups instead of `*` for `hadoop.proxyuser.httpfs.groups`
3. Enable SSL/TLS for HTTPFS
4. Implement proper authentication mechanisms

## References

- [Apache Hadoop HTTPFS Documentation](https://hadoop.apache.org/docs/stable/hadoop-hdfs-httpfs/index.html)
- [WebHDFS REST API](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html)
- [Ambari User Guide](https://cwiki.apache.org/confluence/display/AMBARI/Ambari+User+Guides)
