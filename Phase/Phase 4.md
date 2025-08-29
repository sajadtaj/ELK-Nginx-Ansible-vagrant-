# فاز ۴ — پایگاه‌داده (PostgreSQL روی نود `db`)

## ۱) هدف فاز

* نصب و پیکربندی **PostgreSQL** روی ماشین `db` (Debian 11/bullseye).
* تنظیمات استاندارد و ایمن برای:

  * **UTF-8** (server/client)،
  * **logging** مناسب برای ELK (ترجیحاً `csvlog` برای سازگاری گسترده)،
  * **شبکه** (‌`listen_addresses`‌ و قواعد دسترسی `pg_hba.conf` برای نودهای مجاز)،
  * **service health** (تست‌های ساده `psql -c 'select 1'` و چک لاگ).
* آماده‌سازی برای **فاز ۵ (Filebeat)** تا لاگ‌های PostgreSQL به Logstash ارسال شوند.
* همه‌چیز **idempotent** و ماژولار با Ansible Role اختصاصی `db`.

### چرا `csvlog`؟

* در PostgreSQL، مقصد لاگ با `log_destination` تنظیم می‌شود؛ گزینه‌ها شامل `stderr`, `csvlog`, `syslog`, (و در نسخه‌های جدیدتر `jsonlog`) است. برای محیط‌های Debian 11 که معمولاً PostgreSQL 13 را دریافت می‌کنند، **`csvlog`** پایدار و استاندارد است؛ فعال‌سازی با **`logging_collector=on`** انجام می‌شود. ([PostgreSQL][1], [postgresqlco.nf][2])

### خط‌مشی کُدگذاری (UTF-8)

* PostgreSQL از تبدیل خودکار کدگذاری بین سرور و کلاینت پشتیبانی می‌کند؛ ما **server\_encoding=UTF8** و **client\_encoding=UTF8** (سطح سشن/کلاینت) را مبنا می‌گیریم تا نمایش فارسی/انگلیسی بدون مشکل باشد. ([PostgreSQL][3])

### دسترسی شبکه و امنیت پایه

* `listen_addresses` کنترل می‌کند سرور روی کدام IPها گوش بدهد؛ کنترل اینکه «چه کسی حق اتصال دارد» با **`pg_hba.conf`** است (سطح آدرس/روش احراز). بنابراین پیشنهاد: `listen_addresses='*'` در شبکه خصوصی Vagrant و محدودسازی دقیق در `pg_hba.conf` فقط به نودهای پروژه (LB/WEB/ELK در صورت نیاز). ([PostgreSQL][4], [Stack Overflow][5])

---

## ۲) معماری این فاز

```mermaid
flowchart LR
  subgraph PrivateNet[192.168.57.0/24]
    LB[lb: Nginx\n.10]
    W1[web1: Nginx:8080\n.11]
    W2[web2: Nginx:8080\n.12]
    DB[(db: PostgreSQL\n.13)]
    ELK[(elk: Elastic Stack\n.14)]
  end

  W1 <-- SQL over 5432 --> DB
  W2 <-- SQL over 5432 --> DB
  DB -- csvlog files --> (فاز۵: Filebeat) --> ELK

  %% در این فاز تمرکز روی DB و لاگ‌نویسی است؛ اتصال اپ اختیاری/بعدی
```

* پورت پیش‌فرض PostgreSQL: **5432**.
* مسیر لاگ در Debian معمولاً زیر `/var/log/postgresql/` مدیریت می‌شود (با `logging_collector=on` و `csvlog`). جزئیات دقیق چرخش/نام‌گذاری با پارامترهای استاندارد لاگ کنترل می‌شود. ([PostgreSQL][6])

---

## ۳) طراحی Role و پارامترها

### تصمیم‌های کلیدی

* **Role مجزا: `db`** طبق Best Practice نقش‌ها در Ansible. ([Ansible Docs][7])
* متغیرها در `group_vars/db.yml`:

  * نسخه بسته: از متاپکیج `postgresql` استفاده می‌کنیم (نسخه پیش‌فرض Debian)، یا اگر لازم شد نسخه دقیق را پارامتریک می‌کنیم.
  * `db_listen_addresses` (پیش‌فرض: `"*"`) – فقط داخل شبکه خصوصی Vagrant است.
  * `db_encoding: "UTF8"`, `db_locale: "en_US.UTF-8"` (سازگار با فاز ۱).
  * Logging:

    * `logging_collector: on`
    * `log_destination: "csvlog"`
    * `log_statement: "none"` (قابل ارتقاء در فازهای بعد)
    * `log_line_prefix` استاندارد شامل زمان/سطح/پید و … (برای پارس در Logstash).

      * نمونه: `'%m [%p] %q%u@%d %r '`
  * امنیت اولیه:

    * کاربران/دیتابیس‌های نمونه (اختیاری در این فاز)؛ در صورت تعریف، **رمزها با Ansible Vault** نگه‌داری شوند. ([Ansible Docs][8])

> یادآوری: برخی پارامترهای لاگ فقط با **restart** اعمال می‌شوند (مثل `logging_collector`). ([PostgreSQL][6])

---

## ۴) ساختار فایل‌ها (TREE) و نقش هرکدام

```
ansible/
├─ inventories/
│  └─ vagrant/
│     ├─ hosts.ini
│     └─ group_vars/
│        └─ db.yml                    # متغیرهای نقش DB (شبکه، لاگ، UTF-8، ...)
├─ playbooks/
│  └─ db.yml                          # Playbook اجرای نقش DB روی گروه [dbservers]
└─ roles/
   └─ db/
      ├─ defaults/
      │  └─ main.yml                  # مقادیر پیش‌فرض role (ایمن/مینیمال)
      ├─ vars/
      │  └─ main.yml                  # (در صورت نیاز به ثابت‌ها)
      ├─ tasks/
      │  ├─ install.yml               # نصب PostgreSQL، ماژول‌ها/ابزارها
      │  ├─ config.yml                # ویرایش postgresql.conf و pg_hba.conf
      │  ├─ service.yml               # enable/start/restart + health check
      │  └─ main.yml                  # include_* تسک‌ها
      ├─ templates/
      │  ├─ postgresql.conf.j2        # تنظیمات کلیدی: listen, logging, encoding
      │  ├─ pg_hba.conf.j2            # قواعد دسترسی
      │  └─ 10-logging.sql.j2         # (اختیاری) ست‌کردن پارامترهای session، تست
      ├─ handlers/
      │  └─ main.yml                  # reload/restart PostgreSQL
      └─ files/
         └─ README.txt                # (درصورت نیاز به فایل‌های استاتیک)
```

### نقش فایل‌ها (خلاصه)

* `playbooks/db.yml` اجرای role روی گروه `[dbservers]`.
* `group_vars/db.yml` تنظیمات قابل سفارشی (listen/log/UTF-8).
* `postgresql.conf.j2` شامل `listen_addresses`, `logging_collector`, `log_destination=csvlog`, `log_line_prefix`, `shared_buffers` (حداقلی).
* `pg_hba.conf.j2` دسترسی فقط از IPهای پروژه (web1/web2/elk) با روش `md5`/`scram-sha-256` (بسته به نسخه).
* `service.yml` پس از تغییر تنظیمات، **`nginx -t` معادلش در PostgreSQL** یعنی `pg_isready` و یک `psql -c 'select 1'`.
* `handlers/main.yml` برای `restart` (وقتی `logging_collector` تغییر کرد) و `reload` برای سایر پارامترها. ([PostgreSQL][6])

---

## ۵) دستور ساخت مسیرها و فایل‌ها (ترمینال)

> مطابق روال شما، «در ابتدای هر فاز» اول مسیرها و فایل‌های خالی ایجاد می‌کنیم:

```bash
# از ریشه پروژه (elk-lab/)
mkdir -p ansible/inventories/vagrant/group_vars
mkdir -p ansible/playbooks
mkdir -p ansible/roles/db/{defaults,vars,tasks,templates,handlers,files}

# فایل‌های vars/playbooks
: > ansible/inventories/vagrant/group_vars/db.yml
: > ansible/playbooks/db.yml

# فایل‌های role
: > ansible/roles/db/defaults/main.yml
: > ansible/roles/db/vars/main.yml
: > ansible/roles/db/tasks/main.yml
: > ansible/roles/db/tasks/install.yml
: > ansible/roles/db/tasks/config.yml
: > ansible/roles/db/tasks/service.yml
: > ansible/roles/db/templates/postgresql.conf.j2
: > ansible/roles/db/templates/pg_hba.conf.j2
: > ansible/roles/db/templates/10-logging.sql.j2
: > ansible/roles/db/handlers/main.yml
: > ansible/roles/db/files/README.txt
```

> بعد از تأیید تو، محتوای این فایل‌ها را دقیق و مستند می‌نویسم (هم‌راستا با Debian 11 و PostgreSQL پیش‌فرض) و اجرا را گام‌به‌گام جلو می‌بریم.

---

## ۶) تست و پذیرش (Design of Tests — اجرا پس از تأیید)

1. **نصب و سرویس**

   * `systemctl is-active postgresql && systemctl is-enabled postgresql`
   * `pg_isready` باید `accepting connections` برگرداند.

2. **صحت شبکه و دسترسی**

   * از `web1` یا میزبان: `psql -h 192.168.57.13 -U postgres -d postgres -c 'select 1'` (در صورت تعریف دسترسی/رمز).
   * کنترل قوانین `pg_hba.conf` و `listen_addresses`.

3. **لاگ‌ها (csvlog)**

   * بررسی مسیر لاگ و وجود فایل‌های CSV (rolling با پارامترها مثل `log_rotation_size/time`). ([PostgreSQL][6])
   * تولید چند رویداد (اتصال ناموفق/موفق) و مشاهده خطوط در CSV.

4. **UTF-8**

   * ایجاد جدول/رکورد شامل متن فارسی و خواندن آن با `client_encoding=UTF8`. ([PostgreSQL][3])

---

## ۷) ملاحظات یکپارچه (۵ نقش)

* **DevOps:**

  * Role ماژولار با `defaults` و `group_vars`، هندلرهای تفکیک‌شده (`reload` vs `restart`)، رعایت Best Practices نقش‌ها. ([Ansible Docs][7])
* **Backend Developer:**

  * دسترسی TCP از وب‌سرورها (در صورت نیاز اپ)؛ سیاست حداقلی سطح دسترسی در `pg_hba.conf`.
* **Software Engineer (SOLID/Modularity):**

  * تفکیک install/config/service برای تست‌پذیری و نگهداشت؛ متغیرپذیر بودن log/encoding.
* **Database Manager:**

  * تنظیمات logging استاندارد PostgreSQL (`logging_collector`, `log_destination=csvlog`, `log_line_prefix`)، الگوی چرخش لاگ. ([PostgreSQL][6])
* **Data Engineer:**

  * انتخاب `csvlog` برای ingestion ساده‌تر در Logstash/Beats (فاز ۵/۶)؛ بعداً می‌توان mapping به ECS انجام داد.

---

## ۸) ریسک‌ها و راهکارها

* **عدم سازگاری نسخه/بسته در Debian**: از متاپکیج `postgresql` استفاده می‌کنیم تا نسخه پایدار repo نصب شود؛ نسخه دقیق را با متغیر جدا می‌کنیم تا بعداً بتوانیم pin کنیم.
* **تغییر پارامترهای نیازمند restart**: اعمال با handler `restart` و اعلان شفاف در خروجی Play. (مثل `logging_collector`). ([PostgreSQL][6])
* **دسترسی ناخواسته از خارج شبکه**: `pg_hba.conf` را فقط به IPهای پروژه محدود می‌کنیم؛ اگر نیاز نبود، ارتباط بیرونی را می‌بندیم. ([PostgreSQL][4])

---

## ۹) منابع رسمی (برای این فاز)

* PostgreSQL — Server Configuration & Logging (Chapter 19 + Logging): ([PostgreSQL][1])
* PostgreSQL — `pg_hba.conf` (فرمت و قواعد): ([PostgreSQL][4])
* PostgreSQL — Character Set/Encoding (UTF-8): ([PostgreSQL][3])
* Ansible — Roles & Best Practices: ([Ansible Docs][7])

---


[1]: https://www.postgresql.org/docs/current/runtime-config.html?utm_source=chatgpt.com "Documentation: 17: Chapter 19. Server Configuration"
[2]: https://postgresqlco.nf/doc/en/param/log_destination/?utm_source=chatgpt.com "PostgreSQL Documentation: log_destination parameter"
[3]: https://www.postgresql.org/docs/current/multibyte.html?utm_source=chatgpt.com "Documentation: 17: 23.3. Character Set Support"
[4]: https://www.postgresql.org/docs/current/auth-pg-hba-conf.html?utm_source=chatgpt.com "Documentation: 17: 20.1. The pg_hba.conf File"
[5]: https://stackoverflow.com/questions/9764126/how-to-configure-postgresql-postgresql-conf-listen-addresses-for-multiple-ip-add?utm_source=chatgpt.com "How to configure postgresql postgresql.conf ..."
[6]: https://www.postgresql.org/docs/current/runtime-config-logging.html?utm_source=chatgpt.com "Documentation: 17: 19.8. Error Reporting and Logging"
[7]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html?utm_source=chatgpt.com "Roles — Ansible Community Documentation"
[8]: https://docs.ansible.com/ansible/2.8/user_guide/playbooks_best_practices.html?utm_source=chatgpt.com "Best Practices - Ansible Documentation"


باشه سجاد. طبق روال فازهای قبلی، «فاز ۴: پایگاه‌داده PostgreSQL روی نود db» را با حفظ یکپارچگی کامل آماده کردم. ابتدا محتوا را قرار می‌دهیم، سپس اجرای Playbook و تست‌ها را انجام می‌دهیم. (در این پاسخ، تمام فایل‌های این فاز **کامل** ارائه می‌شوند.)

---

# فاز ۴ — PostgreSQL روی `db`

## یادآوری ساختار (پیش‌تر ساخته شد)

```
ansible/
├─ inventories/
│  └─ vagrant/
│     └─ group_vars/
│        └─ db.yml
├─ playbooks/
│  └─ db.yml
└─ roles/
   └─ db/
      ├─ defaults/main.yml
      ├─ vars/main.yml
      ├─ tasks/
      │  ├─ main.yml
      │  ├─ install.yml
      │  ├─ config.yml
      │  └─ service.yml
      ├─ templates/
      │  ├─ postgresql.conf.j2
      │  ├─ pg_hba.conf.j2
      │  └─ 10-logging.sql.j2   # اختیاری (اینجا خالی می‌ماند)
      └─ handlers/main.yml
```

> پیش‌فرض: گروه `[dbservers]` در `hosts.ini` موجود است و هاست `db` در آن قرار دارد (همان الگوی inventory که در فازهای قبل داشتیم).

---

## 1) فایل متغیرهای گروه DB

**مسیر:** `ansible/inventories/vagrant/group_vars/db.yml`
**نوع:** جایگزینی کامل (Full)

```yaml
# encoding: utf-8
# متغیرهای عمومی نقش DB (Debian 11 / PostgreSQL پیش‌فرض دیسترو)

# شبکه و درگاه
db_listen_addresses: "*"    # فقط در شبکه خصوصی Vagrant مصرف می‌شود
db_port: 5432

# کدگذاری/محلی‌سازی
db_encoding: "UTF8"
db_locale: "en_US.UTF-8"

# Logging (سازگار با ELK)
db_logging_collector: "on"
db_log_destination: "csvlog"                  # پایدار و قابل پردازش
db_log_directory: "log"                       # PGDATA/log (پیش‌فرض collector)
db_log_filename: "postgresql-%Y-%m-%d_%H%M%S.log"
db_log_rotation_age: "1d"
db_log_rotation_size: "100MB"
db_log_line_prefix: "%m [%p] %q%u@%d %r "

# پارامترهای عمومی مینیمال
db_max_connections: 100
db_shared_buffers: "128MB"                    # مقدار محافظه‌کارانه برای VM آموزشی

# دسترسی مجاز از نودهای پروژه (LB/Web/ELK)
db_allowed_hosts:
  - "192.168.57.10"   # lb
  - "192.168.57.11"   # web1
  - "192.168.57.12"   # web2
  - "192.168.57.14"   # elk

# احراز (روش)
db_auth_method: "md5"                         # در صورت نیاز بعداً به scram ارتقا می‌دهیم

# (اختیاری) ایجاد کاربر/دیتابیس نمونه — غیرفعال
db_bootstrap: false
db_app_user: "appuser"
db_app_db: "appdb"
# رمز را فقط در Vault نگه دارید؛ اینجا صرفاً برای الگو:
db_app_password: "CHANGE_ME_IN_VAULT"
```

---

## 2) Playbook فاز DB

**مسیر:** `ansible/playbooks/db.yml`
**نوع:** جایگزینی کامل (Full)

```yaml
# encoding: utf-8
---
- name: Configure PostgreSQL on database nodes
  hosts: dbservers
  become: yes
  vars:
    ansible_remote_tmp: /tmp
  roles:
    - db

```

---

## 3) Defaults نقش DB

**مسیر:** `ansible/roles/db/defaults/main.yml`
**نوع:** جایگزینی کامل (Full)

```yaml
# encoding: utf-8
---
db_listen_addresses: "localhost"
db_port: 5432

db_encoding: "UTF8"
db_locale: "en_US.UTF-8"

db_logging_collector: "on"
db_log_destination: "csvlog"
db_log_directory: "log"
db_log_filename: "postgresql-%Y-%m-%d_%H%M%S.log"
db_log_rotation_age: "1d"
db_log_rotation_size: "100MB"
db_log_line_prefix: "%m [%p] %q%u@%d %r "

db_max_connections: 100
db_shared_buffers: "128MB"

db_allowed_hosts: []
db_auth_method: "md5"

db_bootstrap: false
db_app_user: "appuser"
db_app_db: "appdb"
db_app_password: "CHANGE_ME_IN_VAULT"
```

---

## 4) Vars نقش DB (فعلاً خالی)

**مسیر:** `ansible/roles/db/vars/main.yml`
**نوع:** جایگزینی کامل (Full)

```yaml
# encoding: utf-8
---
# برای مقادیر ثابت/خاص نسخه در صورت نیاز
```

---

## 5) وظایف اصلی نقش

**مسیر:** `ansible/roles/db/tasks/main.yml`
**نوع:** جایگزینی کامل (Full)

```yaml
# encoding: utf-8
---
- import_tasks: install.yml
- import_tasks: config.yml
- import_tasks: service.yml
```

---

## 6) نصب PostgreSQL

**مسیر:** `ansible/roles/db/tasks/install.yml`
**نوع:** جایگزینی کامل (Full)

```yaml
# encoding: utf-8
---
- name: Install PostgreSQL and tools (Debian)
  apt:
    name:
      - postgresql
      - postgresql-contrib
      - postgresql-client
      - python3-psycopg2
    state: present
    update_cache: yes
```

> نکته: برای استفاده از ماژول‌های Ansible مرتبط با PostgreSQL، وجود `python3-psycopg2` مفید است.

---

## 7) پیکربندی

**مسیر:** `ansible/roles/db/tasks/config.yml`
**نوع:** جایگزینی کامل (Full)

```yaml
# encoding: utf-8
---
# 1) کشف مسیرهای کانفیگ استاندارد Debian
- name: Discover postgresql.conf path
  shell: "ls -1 /etc/postgresql/*/main/postgresql.conf | head -n1"
  args: { warn: false }
  register: pg_conf_path
  changed_when: false

- name: Discover pg_hba.conf path
  shell: "ls -1 /etc/postgresql/*/main/pg_hba.conf | head -n1"
  args: { warn: false }
  register: pg_hba_path
  changed_when: false

- name: Fail if config files not found
  fail:
    msg: "Could not find postgresql.conf / pg_hba.conf under /etc/postgresql/*/main/"
  when: pg_conf_path.stdout == "" or pg_hba_path.stdout == ""

# --- استخراج نسخه بدون فیلتر Jinja (با sed) ---
- name: Derive PostgreSQL major version from config path (no Jinja filters)
  shell: |
    conf="{{ pg_conf_path.stdout }}"
    # مثال ورودی: /etc/postgresql/13/main/postgresql.conf  → خروجی: 13
    echo "$conf" | sed -E 's#^/etc/postgresql/([^/]+)/main/postgresql\.conf$#\1#'
  args: { warn: false }
  register: pg_ver_cmd
  changed_when: false
  failed_when: pg_ver_cmd.rc != 0 or (pg_ver_cmd.stdout is not defined) or (pg_ver_cmd.stdout | length == 0)

- name: Set PostgreSQL major version fact
  set_fact:
    pg_version: "{{ pg_ver_cmd.stdout }}"

# --- ساخت مسیر دیتادایر استاندارد Debian بدون فیلتر Jinja ---
- name: Build PostgreSQL data directory path (Debian standard)
  shell: |
    echo "/var/lib/postgresql/{{ pg_version }}/main"
  args: { warn: false }
  register: pg_data_cmd
  changed_when: false
  failed_when: pg_data_cmd.rc != 0 or (pg_data_cmd.stdout is not defined) or (pg_data_cmd.stdout | length == 0)

- name: Set PostgreSQL data directory fact
  set_fact:
    pg_data_dir: "{{ pg_data_cmd.stdout }}"

- name: Ensure data directory exists (safety)
  file:
    path: "{{ pg_data_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: "0700"

# 3) اعمال تمپلیت‌ها
- name: Template postgresql.conf
  template:
    src: postgresql.conf.j2
    dest: "{{ pg_conf_path.stdout }}"
    owner: postgres
    group: postgres
    mode: "0644"
  notify: Restart PostgreSQL

- name: Template pg_hba.conf
  template:
    src: pg_hba.conf.j2
    dest: "{{ pg_hba_path.stdout }}"
    owner: postgres
    group: postgres
    mode: "0640"
  notify: Reload PostgreSQL

# 4) (اختیاری) Bootstrap DB/User نمونه — پیشفرض غیرفعال
- name: Ensure app database and user (optional)
  become: yes
  become_user: postgres
  when: db_bootstrap     # بدون |bool
  block:
    - name: Create app database
      postgresql_db:
        name: "{{ db_app_db }}"
        encoding: "{{ db_encoding }}"
        lc_collate: "{{ db_locale }}"
        lc_ctype: "{{ db_locale }}"
        state: present

    - name: Create app user
      postgresql_user:
        name: "{{ db_app_user }}"
        password: "{{ db_app_password }}"
        state: present

    - name: Grant privileges
      postgresql_privs:
        db: "{{ db_app_db }}"
        roles: "{{ db_app_user }}"
        type: database
        privs: "ALL"
        state: present

```

> توجه: اگر `db_bootstrap: true` شد، رمز را از Vault بخوان.

---

## 8) سرویس و Health Check

**مسیر:** `ansible/roles/db/tasks/service.yml`
**نوع:** جایگزینی کامل (Full)

```yaml

# encoding: utf-8
---
- name: Ensure PostgreSQL is enabled and started
  service:
    name: postgresql
    state: started
    enabled: yes

- name: pg_isready should accept connections
  command: pg_isready -p {{ db_port }}
  register: pg_ready
  changed_when: false
  failed_when: pg_ready.rc != 0

- name: Simple SELECT 1 as postgres (no-become workaround)
  shell: sudo -u postgres psql -Atq -p {{ db_port }} -c 'SELECT 1;'
  args:
    warn: false
  register: pg_select
  changed_when: false
  failed_when: pg_select.rc != 0 or (pg_select.stdout | trim) != "1"

```

---

## 9) قالب postgresql.conf

**مسیر:** `ansible/roles/db/templates/postgresql.conf.j2`
**نوع:** جایگزینی کامل (Full)

```jinja# encoding: utf-8

# مسیر دیتادایر برای کلستر دیبیان
data_directory   = '{{ pg_data_dir }}'
# مینیمال/شفاف برای محیط آموزشی
listen_addresses = '{{ db_listen_addresses }}'
port             = {{ db_port }}

# Memory
shared_buffers   = '{{ db_shared_buffers }}'
max_connections  = {{ db_max_connections }}

# Locale/Encoding (ایجاد دیتابیس‌ها با UTF-8 انجام می‌شود)
# server_encoding در هسته DB ثابت است؛ دیتابیس‌های جدید با UTF-8 ساخته می‌شوند.
# client_encoding را در کلاینت/سشن ست می‌کنیم.

# Logging
logging_collector  = {{ db_logging_collector }}
log_destination    = '{{ db_log_destination }}'
log_directory      = '{{ db_log_directory }}'
log_filename       = '{{ db_log_filename }}'
log_rotation_age   = '{{ db_log_rotation_age }}'
log_rotation_size  = '{{ db_log_rotation_size }}'
log_line_prefix    = '{{ db_log_line_prefix }}'

# Recommended minimalist settings for dev lab
# (می‌توان در فازهای بعدی بر اساس منابع VM تنظیمات را بهینه‌تر کرد)

```

---

## 10) قالب pg\_hba.conf

**مسیر:** `ansible/roles/db/templates/pg_hba.conf.j2`
**نوع:** جایگزینی کامل (Full)

```jinja
# encoding: utf-8
# pg_hba.conf — کنترل دسترسی
# ترتیب مهم است؛ از خاص به عام.

# اتصال محلی (سوکت) — برای کاربر postgres
local   all             postgres                                peer

# اتصال محلی برای بقیه کاربران/دیتابیس‌ها (در صورت نیاز):
local   all             all                                     peer

# اتصال از شبکه خصوصی Vagrant برای هاست‌های مجاز
{% for host in db_allowed_hosts %}
host    all             all             {{ host }}/32            {{ db_auth_method }}
{% endfor %}

# در صورت نیاز می‌توان کل subnet را مجاز کرد (غیرفعال):
# host  all             all             192.168.57.0/24          {{ db_auth_method }}
```

> امنیت: صرفاً IPهای لیست‌شده دسترسی دارند. اگر کل ساب‌نت را مجاز می‌خواهی، خط مربوطه را فعال کن.

---

## 11) قالب SQL اختیاری (خالی)

**مسیر:** `ansible/roles/db/templates/10-logging.sql.j2`
**نوع:** جایگزینی کامل (Full)

```sql
-- encoding: utf-8
-- این فایل فعلاً خالی است. در صورت نیاز می‌توان پارامترهای سشن/نمونه تستی را اینجا قرار داد.
```

---

## 12) هندلرها

**مسیر:** `ansible/roles/db/handlers/main.yml`
**نوع:** جایگزینی کامل (Full)

```yaml
# encoding: utf-8
---
- name: Reload PostgreSQL
  service:
    name: postgresql
    state: reloaded

- name: Restart PostgreSQL
  service:
    name: postgresql
    state: restarted
```

---

# اجرای فاز ۴

## 1) اجرای Playbook

```bash
ansible-playbook -i ansible/inventories/vagrant/hosts.ini ansible/playbooks/db.yml --limit dbservers -vv
```

> اگر گروه/هاست `[dbservers]` را در `hosts.ini` قبلاً اضافه کرده‌ای:
>
> ```
> [dbservers]
> db ansible_host=192.168.57.13 ansible_ssh_private_key_file=.vagrant/machines/db/virtualbox/private_key
>
> [all:vars]
> ansible_user=vagrant
> ansible_python_interpreter=/usr/bin/python3
> ```
>
> مشکلی نخواهی داشت. (الگوی inventory شما از فازهای قبل همین را دنبال می‌کند.)

## 2) تست‌های پس از اجرا

### الف) وضعیت سرویس و Health

```bash
vagrant ssh db -c "sudo systemctl is-active postgresql && systemctl is-enabled postgresql"
vagrant ssh db -c "pg_isready -p 5432"
vagrant ssh db -c "sudo -u postgres psql -p 5432 -c 'SELECT 1;'"
```

### ب) بررسی گوش‌دادن روی پورت و آدرس‌ها

```bash
vagrant ssh db -c "ss -ltnp | grep 5432"
```

### ج) بررسی لاگ‌ها (csvlog)

```bash
# مسیر پیش‌فرض collector: PGDATA/log
# برای یافتن مسیر دقیق دیتادایر، می‌توان از:
vagrant ssh db -c "sudo -u postgres psql -tAc \"SHOW data_directory;\""
vagrant ssh db -c "sudo -u postgres find \$(sudo -u postgres psql -tAc \"SHOW data_directory;\")/log -type f -name '*.log' | tail -n 5"
```

### د) تست دسترسی از web1/web2 (اختیاری)

```bash
vagrant ssh web1 -c "psql -h 192.168.57.13 -U postgres -d postgres -c 'SELECT 1;'"
# (در صورت لازم بودن رمز، روش md5 برقرار است؛ کاربر postgres معمولاً local-only است.
# برای تست شبکه تولیدی، بهتر است یک کاربر/DB نمونه بسازیم و در Vault رمز را نگه داریم.)
```

---

## نکات یکپارچگی (۵ نقش)

* **DevOps:** Role ماژولار، idempotent؛ تمپلیت مجزا برای `postgresql.conf` و `pg_hba.conf`، هندلرهای `reload`/`restart` تفکیک‌شده، health check‌های روشن (`pg_isready`, `SELECT 1`).
* **Backend Developer:** در صورت نیاز اپ، اتصال TCP از `web1/web2` قابل انجام است؛ سیاست دسترسی در `pg_hba.conf` مینیمال و ایمن.
* **Software Engineer (SOLID):** تفکیک `install/config/service` و پارامترپذیری از `group_vars`؛ تست‌پذیر و نگهداشت‌پذیر.
* **Database Manager:** Logging با `csvlog` و الگوی چرخش مشخص؛ UTF-8 برای داده‌های فارسی؛ محدودسازی دسترسی شبکه.
* **Data Engineer:** csvlog برای ingestion ساده در Beats/Logstash در فازهای ELK بعدی؛ `log_line_prefix` مناسب هم‌بستگی رخدادها.
* **Encoding:** UTF-8 در کل مسیر (مستندات/قالب‌ها/پاسخ‌ها).

---

## خطاهای رایج و رفع

* اگر `pg_isready` خطا داد: `journalctl -u postgresql -n 100 --no-pager` را بررسی کن؛ معمولاً به خاطر پیکربندی `listen_addresses` یا اشتباه در مسیر فایل‌هاست.
* اگر اتصال شبکه‌ای رد شد: `pg_hba.conf` و آدرس‌های مجاز (`db_allowed_hosts`) را بررسی و Playbook را دوباره اجرا کن (هندلر Reload اعمال می‌شود).
* اگر لاگ‌ها در `/var/log/postgresql` مدنظر باشد: این فاز با `csvlog` و collector کار می‌کند (مسیر `PGDATA/log`). اگر می‌خواهی مسیر سیستم‌لاگ باشد، در فازهای بعد `syslog` را فعال و پیکربندی می‌کنیم.

---
