# Gunicorn



## Gunicorn issues

#### `If sock file isn't creating`
1. connection to /run/gunicorn/some_sock.sock failed: [Errno 13] Permission denied

```bash
# if required created directory too
sudo mkdir /run/gunicorn

# user:group
sudo chown ubuntu:www-data /run/gunicorn
sudo chmod 775 /run/gunicorn
```

<hr>

# Gunicorn + Systemd

### 1. Project Structure
```bash
/var/www/myproject/
├── venv/                 # Python virtual environment
├── myproject/            # Django project root
└── manage.py
```


### 2. Gunicorn Systemd Service
Create a new service file:
`/etc/systemd/system/gunicorn_myproject.service`
```bash
[Unit]
Description=gunicorn daemon for MyProject
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/myproject

# Remove old socket before binding a new one
ExecStartPre=/bin/rm -f /run/gunicorn-myproject/gunicorn_myproject.sock

# Keeps the listening socket open, Spawns new workers, Old workers finish in-flight requests & Reopens log files
ExecReload=/bin/kill -HUP $MAINPID

# Start Gunicorn
ExecStart=/var/www/myproject/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --timeout 120 \
          --bind unix:/run/gunicorn-myproject/gunicorn_myproject.sock myproject.wsgi:application

Restart=always

# Ensure runtime directory exists with correct permissions always when server restarts
RuntimeDirectory=gunicorn-myproject
RuntimeDirectoryMode=775

[Install]
WantedBy=multi-user.target
```

### 3. Apply and Test
Reload systemd and restart Gunicorn:
```bash
sudo systemctl daemon-reload
sudo systemctl enable gunicorn_myproject.service
sudo systemctl restart gunicorn_myproject.service
sudo systemctl status gunicorn_myproject.service
```


### 4. If `gunicorn-myproject` directory isn't there in `/run/`
Create a directory for the socket with the right owner/group:
```bash
sudo mkdir -p /run/gunicorn-myproject
sudo chown www-data:www-data /run/gunicorn-myproject
sudo chmod 775 /run/gunicorn-myproject
```

### 5. Verify
Check the socket is created:
```bash
ls -l /run/gunicorn-myproject/
```

Expected output:
```bash
srw-rw---- 1 www-data www-data 0 Sep 18 13:10 gunicorn_myproject.sock
```

### 6. Troubleshooting
- If you see 502 Bad Gateway:
  - Ensure the socket file exists and is owned by www-data:www-data.
  - Run:
    ```bash
    sudo journalctl -u gunicorn_myproject -f
    ```

### 7. Test
- Test the socket using running below command
- `gunicorn --bind unix:/run/gunicorn-myproject/gunicorn_myproject.sock myproject.wsgi:application`
- Open new instance of terminal and run
- `curl --unix-socket unix:/run/gunicorn-myproject/gunicorn_myproject.sock http://localhost/`
- You will see somthing like (if you have nothing in root API endpoint)
```html
<!doctype html>
<html lang="en">
<head>
  <title>Not Found</title>
</head>
<body>
  <h1>Not Found</h1><p>The requested resource was not found on this server.</p>
</body>
</html>
  ```
- BOOM!



### NOTE: Must Check
Adding this line to your unit…

```
ExecReload=/bin/kill -HUP $MAINPID
```

…means that when you run:

```
sudo systemctl reload gunicorn_myproject.service
```

systemd will send `SIGHUP` to Gunicorn’s **master** process. For Gunicorn, `HUP` triggers a **graceful reload**:

* **Keeps the listening socket open** (no drop in connections).
* **Spawns new workers** with the (re)loaded config.
* **Old workers finish in-flight requests** then exit (bounded by `graceful-timeout`).
* **Reopens log files** and applies config changes in `gunicorn.conf.py`.

What it does **not** do:

* It does **not** re-read the systemd unit (that’s `systemctl daemon-reload`).
* It does **not** pick up changed **environment variables** from the unit; for env changes, use a full `restart`.
* If you run Gunicorn with `--preload`, `HUP` **won’t reload application code** (master already has it loaded). In that case, use a full `restart` (or a `USR2` upgrade flow) to get new code.

### When to use reload vs restart for this service

* **Use `reload`** for most code/config deploys (when not using `--preload`) to get near-zero downtime.
* **Use `restart`** if workers are wedged, you changed env vars / virtualenv / dependencies, or you’re using `--preload` and need new code.








