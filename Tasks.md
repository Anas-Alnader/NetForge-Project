# NetDevOps — توزيع المهام (Task Breakdown)

خدمة daemon بلوحة ويب على port محدد، لإدارة الشبكات. Python (FastAPI) + PostgreSQL + Redis/Celery + Netmiko + Docker Compose + Nginx.

## 0. قواعد العمل المشترك (الكل يقرأها)

**ليش منقدر نمشي بالتوازي؟** لأنو منثبّت "العقود" (Contracts) أول يوم، وكل واحد بيشتغل ضمن مجلده، وبيستخدم mocks مكان الأجزاء اللي لسا ما خلصت.

| العقد | مين بيكتبه | الملف |
|---|---|---|
| API Spec (OpenAPI) | Lead | `docs/api-spec.yaml` |
| DB Schema + Migrations | Lead | `app/models.py`, `alembic/` |
| واجهة الـ Driver | Lead | `app/drivers/base.py` |
| شكل الـ Job (الحالة/النتيجة) | Lead | `app/jobs/schema.py` |

**Git workflow**
- `main` محمي: ممنوع push مباشر، فقط PR + review واحد + CI أخضر
- `develop`: فرع التجميع
- فروع الميزات: `feat/<track>-<اسم-قصير>` مثل `feat/drivers-backup-service`
- Commits صغيرة وواضحة، PR صغير (أقل من 400 سطر قدر الإمكان)
- كل Track إلو مجلداته (CODEOWNERS) لتفادي الـ conflicts

**Mocks:** `MockDriver` (راوتر وهمي بيرجّع إعدادات ثابتة) بيكتبه Track B أول يوم، وهيك Track A, C, D بيشتغلوا بدون راوترات حقيقية.

## 1. الهيكل المقترح للـ repo

```
netdevops/
├── app/
│   ├── main.py
│   ├── api/            # Track A
│   ├── auth/           # Track A
│   ├── models.py       # Lead
│   ├── drivers/        # Track B
│   ├── services/       # Track B (backup, diff, deploy, compliance)
│   ├── jobs/           # Track C (celery tasks, scheduler)
│   ├── monitoring/     # Track C
│   └── alerts/         # Track C
├── web/                # Track D (frontend)
├── deploy/             # Track E (docker, nginx, CI)
├── docs/
├── tests/
├── docker-compose.yml  # Track E
├── CODEOWNERS
└── TASKS.md
```

## 2. المهام حسب الـ Track

### Track 0: Lead (يوم 1، الكل ينتظره)
- [ ] إنشاء الـ repo + هيكل المجلدات + `CODEOWNERS` + حماية `main`
- [ ] كتابة `docs/api-spec.yaml` (endpoints: auth, devices, backups, jobs, alerts, users)
- [ ] تعريف الجداول: User, Device, Backup, Job, Alert, AuditLog
- [ ] كتابة `drivers/base.py` (واجهة: `connect`, `get_config`, `push_config`, `get_status`)
- [ ] تعريف حالات الـ Job: `pending / running / success / failed`

### Track A: Backend Core (API + Auth)
- [ ] تسجيل دخول JWT + تشفير كلمات السر (argon2/bcrypt)
- [ ] أدوار وصلاحيات: admin / engineer / viewer
- [ ] CRUD للأجهزة (Inventory API)
- [ ] تشفير credentials الأجهزة (Fernet) وتخزينها
- [ ] Audit log: تسجيل كل تغيير (مين، شو، متى)
- [ ] endpoints للـ Jobs: إنشاء مهمة، متابعة حالتها `/jobs/{id}`
- [ ] Settings API (port, retention, TLS)
- [ ] اختبارات unit للـ API

### Track B: Network Drivers & Services (قلب المشروع)
- [ ] `MockDriver` (أول شي!)
- [ ] `NetmikoDriver` لـ Cisco IOS، بعدين MikroTik / Juniper
- [ ] Backup service: سحب الإعدادات وتخزينها بإصدارات
- [ ] Diff service: مقارنة نسختين وإظهار الفرق
- [ ] قوالب Jinja2 + معاينة (dry-run) قبل التطبيق
- [ ] Deploy service: تطبيق الإعداد + تحقق بعد التنفيذ
- [ ] Rollback تلقائي عند الفشل
- [ ] Compliance: قواعد فحص (مثال: "SSH v2 مفعّل"، "no telnet")
- [ ] اختبارات ضد MockDriver

### Track C: Workers, Scheduler, Monitoring, Alerts
- [ ] إعداد Celery + Redis + ربط حالات الـ Job
- [ ] Scheduler: backup دوري لكل الأجهزة
- [ ] Discovery: مسح subnet بـ python-nmap وإضافة الأجهزة المكتشفة
- [ ] Monitoring: ping / TCP / SNMP، تخزين النتائج
- [ ] endpoint `/metrics` (Prometheus format، اختياري للمستقبل)
- [ ] Alerts: Telegram + Email عند down / فشل backup / تغيير إعداد غير متوقع
- [ ] منع التكرار (alert dedup) وإعادة المحاولة

### Track D: Frontend (لوحة الويب)
- [ ] هيكل الواجهة + Login + حماية الصفحات
- [ ] Dashboard: حالة الأجهزة، آخر التنبيهات، رسوم بيانية
- [ ] صفحة Inventory (إضافة/تعديل/حذف أجهزة)
- [ ] صفحة Backups + عارض Diff بين نسختين
- [ ] صفحة Deploy: اختيار قالب، معاينة، تأكيد، متابعة النتيجة
- [ ] Terminal: SSH من المتصفح (xterm.js + WebSocket)
- [ ] صفحات Alerts, Users, Settings, Audit
- [ ] دعم عربي/إنجليزي (RTL) وتصميم متجاوب
- [ ] يشتغل أول شي على mock API (من `api-spec.yaml`)

### Track E: DevOps & Quality
- [ ] Dockerfile للـ api والـ worker
- [ ] `docker-compose.yml`: api, worker, scheduler, postgres, redis, nginx
- [ ] Nginx: reverse proxy + HTTPS + rate limiting + تقديم ملفات الواجهة
- [ ] إدارة الأسرار (`.env.example`، بدون أسرار بالـ repo)
- [ ] GitHub Actions: lint (ruff) + tests + build على كل PR
- [ ] مختبر GNS3 موحّد: مشروع `lab/netdevops-lab.gns3project` + دليل `docs/gns3-setup.md` (تفاصيله بقسم 3)
- [ ] سكربت `lab/bootstrap.py` يفعّل SSH ويضبط IP الإدارة على الراوترات بعد تشغيلها
- [ ] ملف systemd + سكربت نشر على VPS
- [ ] README + دليل التثبيت

## 3. بيئة GNS3 (المختبر الموحّد للفريق)

كل عضو بالفريق بيستورد **نفس المشروع** عشان الكل يختبر على نفس الطوبولوجيا ونفس الـ IPs.

### الطوبولوجيا

```
                    ┌──────────────┐
   جهاز المطوّر ────┤  Cloud node  │  (شبكة الإدارة 192.168.100.0/24)
   (Docker: خدمتك)  └──────┬───────┘
                           │
                      ┌────┴────┐
                      │   SW1   │  (سويتش الإدارة)
                      └─┬──┬──┬─┘
                        │  │  │
                      ┌─┴┐┌┴─┐┌┴─┐
                      │R1││R2││R3│     روابط البيانات:
                      └┬─┘└┬─┘└┬─┘     R1 ── R2 ── R3
                       └───┴───┘
```

### خطة العناوين

| الجهاز | IP الإدارة | الدور |
|---|---|---|
| Host (جهاز المطوّر) | 192.168.100.1 | تشغيل الخدمة |
| SW1 | 192.168.100.2 | سويتش الإدارة |
| R1 | 192.168.100.11 | راوتر 1 |
| R2 | 192.168.100.12 | راوتر 2 |
| R3 | 192.168.100.13 | راوتر 3 |

روابط البيانات: `R1-R2: 10.0.12.0/30` و `R2-R3: 10.0.23.0/30`.

### الصور (Images)

| الصورة | الاستخدام |
|---|---|
| MikroTik CHR | البداية (مجاني وخفيف) |
| VyOS | بديل مجاني، مناسب للأتمتة |
| Cisco IOSv / CSRv | لاحقاً، بصورة مرخّصة من صاحبها |

كل الأجهزة بنفس اسم المستخدم/كلمة السر للمختبر فقط (`lab` / `lab123`)، ولا تُستخدم أبداً خارج المختبر.

### مهام المختبر

- [ ] (E) إنشاء المشروع بـ GNS3 وتصديره إلى `lab/netdevops-lab.gns3project`
- [ ] (E) كتابة `docs/gns3-setup.md`: تثبيت GNS3، استيراد المشروع، ربط Cloud node بواجهة الإدارة
- [ ] (E) `lab/bootstrap.py`: ضبط IP الإدارة وتفعيل SSH على كل راوتر
- [ ] (B) اختبار `NetmikoDriver` على R1 وR2 وR3 (backup، push config، rollback)
- [ ] (C) اختبار Discovery على `192.168.100.0/24` بحيث يلاقي الأجهزة الثلاثة
- [ ] (C) اختبار Monitoring: إيقاف R2 من GNS3 والتأكد أن التنبيه وصل
- [ ] (الكل) التحقق أن الخدمة بالـ Docker توصل لـ `192.168.100.11` (الشبكة والـ routing)

### قواعد

- `MockDriver` يبقى أساس الاختبارات الآلية (الـ CI على GitHub ما بيقدر يشغّل GNS3)
- اختبارات GNS3 يدوية أو بسكربت محلي (`make lab-test`) قبل دمج أي PR يلمس الـ drivers
- ممنوع تعديل `.gns3project` إلا عبر PR يراجعه Track E

## 4. المراحل (Milestones)

| المرحلة | الهدف | Tracks المشاركة | النتيجة |
|---|---|---|---|
| M0 | العقود والهيكل | Lead | الكل يقدر يبدأ |
| M1 | Login + Inventory + لوحة أساسية | A, D, E | لوحة تنفتح على المتصفح وتضيف أجهزة |
| M2 | Backup + Diff | B, C, D | نسخ احتياطية دورية وعارض فروقات |
| M3 | Monitoring + Alerts | C, D | Dashboard حي وتنبيهات Telegram |
| M4 | Deploy + Rollback | B, D | تطبيق إعدادات بأمان |
| M5 | Discovery + Terminal + Compliance | B, C, D | الميزات المتقدمة |

## 5. تعريف "خلص" (Definition of Done)
- [ ] الكود مدموج عبر PR بعد review
- [ ] اختبارات تنجح بالـ CI
- [ ] الـ endpoint/الميزة موثّقة بـ `docs/`
- [ ] ما في أسرار أو كلمات سر بالكود

## 6. توزيع الأشخاص (عدّله حسب فريقك)

| عدد الفريق | التوزيع المقترح |
|---|---|
| 5 أشخاص | كل Track لشخص، والـ Lead بيكون إنت مع Track E |
| 3 أشخاص | (A + E) / (B + C) / (D) |
| 2 | (A + B + E) / (C + D) |
