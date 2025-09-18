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



