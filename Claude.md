# NetDevOps: ملف سياق المشروع

> هذا الملف مكتوب ليقرأه **الفريق** و**Claude**.
> إذا بدك تسأل Claude عن المشروع: أرفق هذا الملف بالمحادثة (أو الصقه)، وبعدين اسأل. وإذا بتستخدم Claude Code بيقرأه تلقائياً من جذر الـ repo.

## 1. شو هو المشروع؟

خدمة (daemon) بتشتغل على السيرفر وبتفتح **لوحة ويب على port محدد** (مثل Portainer بس للشبكات). مهندس الشبكات بيدير منها الراوترات والسويتشات بدل ما يفتح كل جهاز يدوياً.

**القرار المعماري الأساسي:** خدمة وحدة متكاملة نكتب منطقها بأنفسنا بـ Python، مش pipeline من أدوات جاهزة (بدون Jenkins/Ansible/Grafana كأدوات رئيسية). بنستخدم مكتبات (Netmiko, Jinja2...) كأجزاء جوّا الكود.

### الميزات المخططة
Dashboard، Inventory، Discovery، Config Backup + Diff، Deploy + Rollback، Terminal (SSH من المتصفح)، Monitoring، Alerts (Telegram/Email)، Compliance، Users + Audit، Settings.

## 2. التقنيات

| الطبقة | التقنية |
|---|---|
| Backend | Python 3.12، FastAPI، SQLAlchemy 2 |
| قاعدة البيانات | PostgreSQL 16 |
| المهام بالخلفية | Celery + Redis (workers + beat للجدولة) |
| الاتصال بالأجهزة | Netmiko (SSH)، قوالب Jinja2، python-nmap للـ Discovery |
| الواجهة | ملفات ثابتة بـ `web/` (يقدّمها Nginx) |
| النشر | Docker Compose + Nginx |
| الاختبار | pytest، ruff، GitHub Actions |
| المختبر | GNS3 (الكل بنفس الطوبولوجيا) |

## 3. المعمارية

```
 المتصفح ──► Nginx :8080 ──► FastAPI :8000
                              │
          ┌───────────────────┼───────────────┐
      PostgreSQL         Redis + Workers    Scheduler (beat)
                              │
                        SSH ► الراوترات/السويتشات
```

**مبدأ أساسي:** الـ API ما بتتصل بالراوتر مباشرة. بتنشئ **Job** وبتحطه بـ Redis، والـ worker بينفّذه بالخلفية، والواجهة بتتابع حالته من `/api/jobs/{id}`. هيك راوتر بطيء ما بيعلّق الخدمة.

## 4. هيكل المشروع ومين مسؤول عن شو

| المجلد | Track | المسؤولية |
|---|---|---|
| `app/models.py`, `docs/api-spec.yaml`, `app/drivers/base.py`, `app/jobs/schema.py` | **Lead** | "العقود": تعديلها بس عبر PR من الـ Lead |
| `app/api/`, `app/auth/` | **A** | الـ API، تسجيل الدخول، الصلاحيات، Inventory، Audit |
| `app/drivers/`, `app/services/` | **B** | SSH، Backup، Diff، Deploy، Rollback، Compliance |
| `app/jobs/`, `app/monitoring/`, `app/alerts/` | **C** | Celery، Scheduler، Discovery، Monitoring، Alerts |
| `web/` | **D** | واجهة الويب |
| `deploy/`, `lab/`, `.github/`, `docker-compose.yml` | **E** | Docker، Nginx، CI، مختبر GNS3 |

المهام التفصيلية لكل Track والمراحل (M0 إلى M5) موجودة بـ [TASKS.md](TASKS.md).

## 5. العقود (Contracts): أهم قاعدة

نمشي بالتوازي لأنو أربع أشياء ثابتة ومتفق عليها:

1. **DB Schema:** `app/models.py` (User, Device, Backup, Job, Alert, AuditLog)
2. **API Spec:** `docs/api-spec.yaml`
3. **Driver interface:** `app/drivers/base.py` (`connect`, `disconnect`, `get_config`, `push_config`, `get_status`)
4. **Job schema:** `app/jobs/schema.py` (الأنواع: backup/deploy/discovery/check، والحالات: pending/running/success/failed)

**إذا احتجت تغيّر عقد:** افتح Issue أو PR للـ Lead، ولا تعدّله بصمت داخل PR ميزة.

## 6. قواعد الشغل

- `main` محمي: PR + review واحد + CI أخضر. ممنوع push مباشر
- فروع: `feat/<track>-<اسم-قصير>` (مثال: `feat/drivers-backup-service`)
- PR صغير (أقل من 400 سطر قدر الإمكان)، وكل PR معه اختبارات
- اشتغل ضمن مجلدات الـ Track تبعك. إذا لازم تلمس مجلد غيرك، نسّق مع صاحبه
- **اختبر دايماً على `MockDriver` أول**، لأن الـ CI ما بيقدر يشغّل GNS3
- اختبار على GNS3 يدوي قبل دمج أي PR بيلمس الـ drivers
- `ruff check .` و `pytest` لازم يمرّوا محلياً قبل الـ PR

### قواعد أمان (إلزامية)
- ممنوع أي سر أو كلمة سر بالكود أو الـ repo (استخدم `.env`، وهو بالـ `.gitignore`)
- credentials الأجهزة بتتخزن مشفّرة بـ Fernet (`password_enc`)، ولا تنطبع بالـ logs أبداً
- كلمات سر المستخدمين: argon2
- كل عملية تغيّر شي (إضافة جهاز، deploy، حذف) بتنكتب بـ `AuditLog`
- Deploy لازم يمر بـ **معاينة (dry-run)** قبل التنفيذ، وفيه rollback عند الفشل

## 7. مختبر GNS3

الكل بيستورد نفس المشروع `lab/netdevops-lab.gns3project`.

| الجهاز | IP الإدارة |
|---|---|
| Host (جهازك) | 192.168.100.1 |
| SW1 | 192.168.100.2 |
| R1 / R2 / R3 | 192.168.100.11 / .12 / .13 |

- روابط البيانات: `R1-R2: 10.0.12.0/30`، `R2-R3: 10.0.23.0/30`
- الصور: ابدأ بـ MikroTik CHR أو VyOS (مجانية)، وCisco لاحقاً
- مستخدم المختبر: `lab` / `lab123` **للمختبر فقط**
- الخطوات: [docs/gns3-setup.md](docs/gns3-setup.md)

## 8. تشغيل المشروع

```bash
cp .env.example .env     # عبّي كلمات السر + FERNET_KEY
docker compose up --build
# الواجهة: http://localhost:8080      الـ API: http://localhost:8080/api/health
```

بدون Docker:
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest && uvicorn app.main:app --reload
```

## 9. الحالة الحالية

- [x] M0: هيكل الـ repo، docker-compose، CI، `models.py`، واجهة الـ driver، `MockDriver`، health endpoint
- [ ] `docs/api-spec.yaml`: الـ Lead لازم يكمّل الـ endpoints (حالياً TODO)
- [ ] مختبر GNS3 الموحّد (Track E)
- [ ] M1: Login + Inventory + لوحة أساسية (A, D, E)
- [ ] M2 إلى M5: حسب [TASKS.md](TASKS.md)

> حدّث هذا القسم عند إنهاء كل مرحلة.

---

## 10. تعليمات لـ Claude (عند الإجابة عن أسئلة الفريق)

- جاوب بنفس لغة السائل (غالباً عربي شامي) مع إبقاء المصطلحات التقنية بالإنجليزي
- اشرح المفهوم ببساطة أول شي، وبعدين الكود. فضّل الجداول والرسوم المبسّطة
- التزم بالتقنيات والعقود بهذا الملف. لا تقترح استبدال FastAPI/Celery/Postgres أو إدخال Jenkins/Ansible إلا إذا طُلب
- اقترح الكود ضمن مجلدات الـ Track الخاص بالسائل، ونبّه إذا الحل بيتطلب تغيير عقد (يعني تنسيق مع الـ Lead)
- اكتب الاختبارات ضد `MockDriver`، ولا تفترض وجود راوترات حقيقية بالـ CI
- لا تضع أسرار أو كلمات سر حقيقية بالأمثلة
- إذا السؤال يعتمد على شي مو موجود بهذا الملف (حالة ميزة معيّنة، قرار لسا ما اتخذ)، قل هيك واسأل بدل ما تخمّن

## 11. كيف تسأل Claude (أمثلة)

| الحالة | صيغة السؤال |
|---|---|
| تفهم جزء | "أنا Track B. اشرحلي كيف بيشتغل `BaseDriver` وليش `MockDriver` موجود" |
| تكتب ميزة | "أنا Track B، بدي أكتب `backup_service` يستخدم `BaseDriver` ويخزّن نسخة بجدول `backups` بإصدار جديد، مع اختبارات" |
| مشكلة | "أنا Track E. الـ container ما بيوصل لـ 192.168.100.11 من GNS3. هاد الإعداد تبعي: ..." |
| مراجعة | "راجع هاد الـ PR لـ Track A من ناحية الأمان والتزامه بالعقود: ..." |
| تخطيط | "أنا Track C. شو الترتيب الأنسب لبناء Discovery ثم Monitoring؟" |

**نصيحة:** دايماً اذكر الـ Track تبعك والمجلد اللي بتشتغل فيه، والصق الكود أو رسالة الخطأ كاملة.
