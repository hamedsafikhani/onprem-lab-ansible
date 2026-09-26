# Manual Steps Log

هر کاری که دستی روی سرورها انجام شده، به ترتیب اجرا.
در مرحله‌ی ۷ هر بخش این فایل به یک task یا role در Ansible تبدیل می‌شود.
ستون «معادل Ansible» فعلاً فقط یادداشت است تا بعداً بدانیم هر کار با کدام ماژول انجام می‌شود.

> پسوردها هرگز در این فایل نوشته نمی‌شوند. فقط اسم متغیرشان (مثلاً `DB_PASSWORD`) که در `.secrets.env` است.

---

## مرحله‌ی ۴: سرور `db` (PostgreSQL + Redis)

### ۴-۱. ساختن پسوردها (روی لپ‌تاپ)

```bash
cd ~/devops-lab/todo-infra
echo "DB_PASSWORD=$(openssl rand -hex 16)"    > .secrets.env
echo "REDIS_PASSWORD=$(openssl rand -hex 16)" >> .secrets.env
chmod 600 .secrets.env
echo ".secrets.env" >> .gitignore
```

معادل Ansible: `ansible-vault`

### ۴-۲. نصب بسته‌ها (روی `db`)

```bash
sudo apt update
sudo apt install -y postgresql redis-server
```

معادل Ansible: `ansible.builtin.apt`

### ۴-۳. PostgreSQL: آدرس‌هایی که روی آن گوش می‌دهد

```bash
sudo tee /etc/postgresql/16/main/conf.d/10-todo.conf > /dev/null <<'EOF'
# Managed manually for todo-lab (step 4)
listen_addresses = 'localhost,192.168.60.30'
EOF
```

معادل Ansible: `ansible.builtin.template`

### ۴-۴. PostgreSQL: دسترسی فقط برای app1 و app2

```bash
sudo tee -a /etc/postgresql/16/main/pg_hba.conf > /dev/null <<'EOF'

# todo-lab: only app servers may reach the todo database
host    todo    todo    192.168.60.21/32    scram-sha-256
host    todo    todo    192.168.60.22/32    scram-sha-256
EOF
```

⚠️ idempotent نیست. اجرای دوباره خطوط را تکرار می‌کند.
معادل Ansible: `ansible.builtin.blockinfile` یا `community.postgresql.postgresql_pg_hba`

### ۴-۵. اعمال کانفیگ PostgreSQL

```bash
sudo systemctl restart postgresql
```

`restart` لازم است چون `listen_addresses` با `reload` اعمال نمی‌شود.
معادل Ansible: handler با `ansible.builtin.systemd`

### ۴-۶. PostgreSQL: کاربر و دیتابیس

```bash
sudo -u postgres createuser --pwprompt todo      # DB_PASSWORD
sudo -u postgres createdb --owner=todo todo
```

معادل Ansible: `community.postgresql.postgresql_user` و `community.postgresql.postgresql_db`

### ۴-۷. Redis: فایل کانفیگ جدا

```bash
read -rsp "Redis password: " REDIS_PASS; echo    # REDIS_PASSWORD

sudo tee /etc/redis/todo.conf > /dev/null <<EOF
# Managed manually for todo-lab (step 4)
bind 127.0.0.1 -::1 192.168.60.30
requirepass ${REDIS_PASS}
maxmemory 128mb
maxmemory-policy allkeys-lru
save ""
EOF
unset REDIS_PASS

sudo chown redis:redis /etc/redis/todo.conf
sudo chmod 640 /etc/redis/todo.conf
```

معادل Ansible: `ansible.builtin.template` با `owner`، `group` و `mode`

### ۴-۸. Redis: اضافه کردن include و ری‌استارت

```bash
grep -qxF 'include /etc/redis/todo.conf' /etc/redis/redis.conf \
  || echo 'include /etc/redis/todo.conf' | sudo tee -a /etc/redis/redis.conf > /dev/null

sudo systemctl restart redis-server
```

معادل Ansible: `ansible.builtin.lineinfile` و handler

### ۴-۹. بررسی روی `db`

```bash
sudo ss -tlnp | grep -E '5432|6379'                    # فقط 127.0.0.1 و 192.168.60.30
sudo systemctl is-enabled postgresql redis-server      # enabled
```

### ۴-۱۰. نصب کلاینت‌ها برای تست (روی `app1`)

```bash
sudo apt install -y postgresql-client redis-tools
```

### ۴-۱۱. تست‌ها (روی `app1`)

```bash
# مثبت: باید todo و 192.168.60.30 را نشان دهد
psql "postgresql://todo@db/todo" -c "SELECT current_user, inet_server_addr();"

# مثبت: باید PONG بدهد
read -rs REDISCLI_AUTH; export REDISCLI_AUTH
redis-cli -h db ping
redis-cli -h db config get maxmemory
unset REDISCLI_AUTH

# منفی: باید خطای "no pg_hba.conf entry" بدهد
psql "postgresql://todo@db/postgres"
```

### ۴-۱۲. مشاهده از `lb` (روی لپ‌تاپ)

```bash
vagrant ssh lb -c "timeout 2 bash -c '</dev/tcp/db/5432' && echo 'port 5432 is OPEN from lb'"
```

پورت باز است و فقط pg_hba جلوی ورود را می‌گیرد. در مرحله‌ی ۸ فایروال اضافه می‌شود.

### ۴-۱۳. تست ریبوت (روی لپ‌تاپ)

```bash
vagrant reload db
vagrant ssh db -c "sudo ss -tlnp | grep -E '5432|6379'"
```

---

## مرحله‌ی ۵: سرور `app1` (اپلیکیشن)

### ۵-۱. کپی کد (روی لپ‌تاپ)

```bash
cd ~/devops-lab/todo-infra
vagrant ssh-config > .vagrant-ssh-config
grep -qxF ".vagrant-ssh-config" .gitignore || echo ".vagrant-ssh-config" >> .gitignore
scp -F .vagrant-ssh-config -r app app1:/tmp/todo-app
```

معادل Ansible: `ansible.builtin.copy` یا `ansible.posix.synchronize`

### ۵-۲. کاربر سیستمی و پوشه‌ها (روی `app1`)

```bash
sudo useradd --system --no-create-home --home-dir /opt/todo --shell /usr/sbin/nologin todo
sudo mkdir -p /opt/todo
sudo rm -rf /opt/todo/app
sudo cp -r /tmp/todo-app /opt/todo/app
sudo chown -R root:root /opt/todo
rm -rf /tmp/todo-app
```

معادل Ansible: `ansible.builtin.user` و `ansible.builtin.file`

### ۵-۳. virtualenv و وابستگی‌ها

```bash
sudo apt install -y python3-venv
sudo python3 -m venv /opt/todo/venv
sudo /opt/todo/venv/bin/pip install -r /opt/todo/app/requirements.txt
```

معادل Ansible: `ansible.builtin.pip` با `virtualenv`

### ۵-۴. فایل تنظیمات

```bash
sudo install -d -m 750 -o root -g todo /etc/todo

read -rsp "DB password: " DB_PASS; echo
read -rsp "Redis password: " REDIS_PASS; echo
sudo tee /etc/todo/todo.env > /dev/null <<EOF
DATABASE_URL=postgresql://todo:${DB_PASS}@db:5432/todo
REDIS_URL=redis://:${REDIS_PASS}@db:6379/0
LOG_LEVEL=INFO
EOF
unset DB_PASS REDIS_PASS

sudo chown root:todo /etc/todo/todo.env
sudo chmod 640 /etc/todo/todo.env
```

معادل Ansible: `ansible.builtin.template` + vault

### ۵-۵. سرویس systemd

```bash
sudo tee /etc/systemd/system/todo.service > /dev/null <<'EOF'
[Unit]
Description=Todo API
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
User=todo
Group=todo
WorkingDirectory=/opt/todo/app
EnvironmentFile=/etc/todo/todo.env
ExecStart=/opt/todo/venv/bin/uvicorn main:app --host 192.168.60.21 --port 8000 --workers 2
Restart=always
RestartSec=5

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now todo
```

معادل Ansible: `ansible.builtin.template` + `ansible.builtin.systemd` (با `daemon_reload: true`)

### ۵-۶. بررسی

```bash
systemctl status todo --no-pager
sudo ss -tlnp | grep 8000
journalctl -u todo -n 30 --no-pager
curl -s http://192.168.60.21:8000/health/ready
```

### ۵-۷. تست API (روی لپ‌تاپ)

```bash
curl -s http://192.168.60.21:8000/health/ready
curl -s -X POST http://192.168.60.21:8000/api/todos -H 'Content-Type: application/json' -d '{"title":"first todo"}'
curl -si http://192.168.60.21:8000/api/todos | grep -iE 'x-cache|x-served-by'
curl -si http://192.168.60.21:8000/api/todos | grep -iE 'x-cache|x-served-by'
```

### ۵-۸. تمرین‌های خرابی

```bash
# الف) کرش پروسه (روی app1)
sudo systemctl kill -s SIGKILL todo
sleep 7; systemctl status todo --no-pager | head -5

# ب) خاموش شدن Redis (روی db، بعد از لپ‌تاپ تست کنید)
sudo systemctl stop redis-server
sudo systemctl start redis-server

# ج) خاموش شدن PostgreSQL (روی db)
sudo systemctl stop postgresql
#   روی app1: sudo systemctl restart todo ; journalctl -u todo -f
sudo systemctl start postgresql

# د) ریبوت (روی لپ‌تاپ)
vagrant reload app1
```

### ۵-۹. تکرار روی `app2`

همه‌ی بخش‌های ۵-۱ تا ۵-۶ با این تفاوت‌ها:
- در `scp` به جای `app1` بنویسید `app2`
- در `ExecStart` به جای `192.168.60.21` بنویسید `192.168.60.22`

---

## مرحله‌ی ۶: سرور `lb` (Nginx)

### ۶-۱. کپی فرانت (روی لپ‌تاپ)

```bash
cd ~/devops-lab/todo-infra
vagrant ssh-config > .vagrant-ssh-config
scp -F .vagrant-ssh-config frontend/index.html lb:/tmp/index.html
```

### ۶-۲. نصب Nginx و فایل‌های ثابت (روی `lb`)

```bash
sudo apt update
sudo apt install -y nginx
sudo install -d -m 755 -o root -g root /var/www/todo
sudo install -m 644 -o root -g root /tmp/index.html /var/www/todo/index.html
rm /tmp/index.html
```

معادل Ansible: `ansible.builtin.apt` و `ansible.builtin.copy`

### ۶-۳. فرمت لاگ

```bash
sudo tee /etc/nginx/conf.d/10-log-format.conf > /dev/null <<'EOF'
log_format upstreamlog '$remote_addr [$time_local] "$request" $status '
                       'upstream=$upstream_addr ustatus=$upstream_status '
                       'rt=$request_time urt=$upstream_response_time';
EOF
```

### ۶-۴. سایت todo

```bash
sudo tee /etc/nginx/sites-available/todo > /dev/null <<'EOF'
upstream todo_api {
    server 192.168.60.21:8000 max_fails=3 fail_timeout=10s;
    server 192.168.60.22:8000 max_fails=3 fail_timeout=10s;
    zone todo_api 64k;
    keepalive 16;
}

server {
    listen 192.168.60.10:80;
    server_name _;

    root /var/www/todo;
    index index.html;

    access_log /var/log/nginx/todo_access.log upstreamlog;
    error_log  /var/log/nginx/todo_error.log;

    location /api/ {
        proxy_pass http://todo_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 2s;
        proxy_read_timeout 10s;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }

    location = /lb-health {
        access_log off;
        default_type text/plain;
        return 200 "ok\n";
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

معادل Ansible: `ansible.builtin.template` (لیست upstream از روی inventory ساخته می‌شود)

### ۶-۵. فعال‌سازی (روی `lb`)

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -sfn /etc/nginx/sites-available/todo /etc/nginx/sites-enabled/todo
sudo nginx -t && sudo systemctl restart nginx
sudo ss -tlnp | grep ':80'                  # فقط 192.168.60.10:80
curl -s http://192.168.60.10/lb-health      # ok
```

⚠️ **اولین بار باید `restart` شود، نه `reload`.** سایت `default` روی `0.0.0.0:80` گوش می‌دهد
و `reload` نمی‌تواند سوکت `192.168.60.10:80` را باز کند (`Address already in use`).
`systemctl reload` این خطا را گزارش نمی‌دهد؛ فقط در `/var/log/nginx/error.log` ثبت می‌شود
و Nginx بی‌صدا با کانفیگ قبلی ادامه می‌دهد.

برای تغییرات بعدی (upstream، location، هدرها) `reload` کافی است:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

عیب‌یابی:

```bash
sudo tail -n 5 /var/log/nginx/error.log
sudo nginx -T 2>/dev/null | grep -E '^# configuration file|listen'
```

معادل Ansible: `ansible.builtin.file` (state=link)، `template` با `validate: nginx -t -c %s`،
و handler جدا برای restart (تغییر listen) و reload (بقیه‌ی تغییرات)

### ۶-۶. IP واقعی کاربر (از لپ‌تاپ، روی `app1` و `app2`)

```bash
cd ~/devops-lab/todo-infra
for h in app1 app2; do
  echo "== $h =="
  vagrant ssh $h -c "sudo sed -i 's|--workers 2\$|--workers 2 --forwarded-allow-ips 192.168.60.10|' /etc/systemd/system/todo.service \
    && sudo systemctl daemon-reload \
    && sudo systemctl restart todo \
    && grep ExecStart /etc/systemd/system/todo.service"
done
```

بررسی (در لاگ باید `192.168.60.1:0` دیده شود، نه `192.168.60.10`):

```bash
for i in 1 2; do curl -s -o /dev/null http://192.168.60.10/api/todos; done
vagrant ssh app1 -c "journalctl -u todo -n 2 --no-pager"
vagrant ssh app2 -c "journalctl -u todo -n 2 --no-pager"
```

معادل Ansible: در template سرویس، `--forwarded-allow-ips {{ lb_ip }}` از اول وجود دارد

### ۶-۷. تست (روی لپ‌تاپ)

```bash
curl -s http://192.168.60.10/lb-health
for i in $(seq 6); do curl -s -o /dev/null -D - http://192.168.60.10/api/todos | grep -i x-served-by; done
```

مرورگر: `http://192.168.60.10`

### ۶-۸. تمرین‌های خرابی

```bash
# الف) خاموش کردن اپلیکیشن روی app2
sudo systemctl stop todo           # روی app2
sudo tail -f /var/log/nginx/todo_access.log   # روی lb
sudo systemctl start todo          # روی app2

# ب) از کار افتادن کامل ماشین (روی لپ‌تاپ)
vagrant suspend app2
for i in $(seq 20); do curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" http://192.168.60.10/api/todos; sleep 1; done
vagrant resume app2

# ج) خاموش شدن دیتابیس
sudo systemctl stop postgresql     # روی db
curl -s -w " %{http_code}\n" http://192.168.60.10/api/todos
sudo systemctl start postgresql    # روی db

# د) ریبوت lb (روی لپ‌تاپ)
vagrant reload lb
curl -s http://192.168.60.10/lb-health
```
