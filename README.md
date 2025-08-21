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

