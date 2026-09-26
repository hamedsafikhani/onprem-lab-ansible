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

_(در مرحله‌ی بعد تکمیل می‌شود)_
