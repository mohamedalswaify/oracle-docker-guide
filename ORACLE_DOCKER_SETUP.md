# Oracle Database 18c XE on Docker — Windows & macOS

دليل عملي لتشغيل **Oracle Database 18c XE** داخل Docker، ثم الاتصال بقاعدة البيانات من **Oracle SQL Developer** وتثبيت بيانات التدريب **HR Sample Schema**.

> **نفّذ الأوامر مرحلةً مرحلة.** أوامر `powershell` في Windows PowerShell، وأوامر `bash` في Terminal أو داخل الحاوية حسب العنوان، وأوامر `sql` داخل SQL*Plus. لا تنفّذ كل كتل الأوامر دفعة واحدة.
>
> **لجهاز عليه Oracle بالفعل:** انتقل مباشرة إلى [التشغيل اليومي](#7-التشغيل-اليومي-windows--macos). لا تُنشئ حاوية ثانية ولا تعِد تثبيت HR.

## 1. روابط التحميل

| البرنامج أو الملفات | الرابط | ملاحظات |
|---|---|---|
| Docker Desktop — Windows | [الموقع الرسمي](https://docs.docker.com/desktop/setup/install/windows-install/) | يتطلب تفعيل WSL 2 عند استخدام Linux Containers |
| Docker Desktop — macOS | [الموقع الرسمي](https://docs.docker.com/desktop/setup/install/mac-install/) | اختر إصدار **Intel** أو **Apple Silicon** المناسب |
| DBeaver Community — Windows / macOS | [DBeaver Download](https://dbeaver.io/download/) | أداة مجانية للاتصال بـ **Oracle وSQL Server** من برنامج واحد |
| Oracle SQL Developer — Windows / macOS | [Oracle Downloads](https://www.oracle.com/database/sqldeveloper/technologies/download/) | أداة الاتصال وإدارة قاعدة البيانات |
| Oracle Database XE | [صفحة Oracle الرسمية](https://www.oracle.com/database/technologies/appdev/xe.html) | للتثبيت المباشر؛ **غير مطلوب** عند استخدام صورة Docker |
| Oracle XE 18c Docker image | [Docker Hub — gvenzl/oracle-xe](https://hub.docker.com/r/gvenzl/oracle-xe) | صورة **طرف ثالث** وليست صورة Oracle الرسمية؛ الوسم المستخدم `18` |
| Oracle HR Sample Schema v18c | [GitHub](https://github.com/oracle-samples/db-sample-schemas/tree/v18c) · [ZIP مباشر](https://github.com/oracle-samples/db-sample-schemas/archive/refs/tags/v18c.zip) | جداول `EMPLOYEES`, `DEPARTMENTS`, `JOBS` وغيرها |

## 2. تجهيز Windows: WSL 2 ثم Docker Desktop

### 2.1 تثبيت أو تحديث WSL 2 (مرة واحدة)

افتح **PowerShell بصلاحية Administrator**:

```powershell
wsl --version
```

إذا كان WSL **غير مثبت**، نفّذ:

```powershell
wsl --install --no-distribution
```

إذا ظهر خطأ أن الخيار `--no-distribution` غير مدعوم، استخدم `wsl --install` بدلًا منه. وإذا طلب Docker تحديث WSL، نفّذ:

```powershell
wsl --update
wsl --set-default-version 2
wsl --status
```

أعد تشغيل Windows إذا طُلب ذلك. **لا تحتاج تشغيل الأمر `wsl` يدويًا كل يوم**؛ Docker Desktop يدير بيئته تلقائيًا.

### 2.2 تثبيت Docker Desktop

1. نزّل [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/).
2. ثبّته واختر **WSL 2 backend** إذا ظهر الخيار.
3. افتح Docker Desktop وانتظر حتى يصبح Docker Engine جاهزًا.
4. افتح PowerShell ونفّذ:

```powershell
docker --version
docker info
docker ps -a
```

إذا فشل `docker info`، افتح Docker Desktop وانتظر جاهزية المحرك قبل الاستمرار.

## 3. تجهيز macOS: Docker Desktop (بدون WSL)

1. من قائمة Apple اختر **About This Mac** لمعرفة المعالج: **Intel** أو **Apple Silicon**.
2. نزّل [Docker Desktop for Mac](https://docs.docker.com/desktop/setup/install/mac-install/) بالنسخة المناسبة.
3. افتح ملف `.dmg` وانقل Docker إلى **Applications**، ثم شغّله ووافق على الأذونات المطلوبة.
4. من **Terminal**:

```bash
docker --version
docker info
docker ps -a
```

> **تنبيه توافق مهم:** صورة `gvenzl/oracle-xe:18` المستخدمة هنا مخصصة لمعمارية `linux/amd64`. على **Mac Intel** يمكن استخدامها بالطريقة الموضحة أدناه. أما **Mac Apple Silicon (M1/M2/M3/M4...)** فلا نوصي باعتبار محاكاة Oracle 18c x86-64 حلًا مضمونًا؛ قد تفشل أو تكون بطيئة. الحل الأكثر موثوقية هو تشغيل Oracle 18c على جهاز/خادم `amd64` والاتصال به من SQL Developer على الماك، أو اختيار إصدار Oracle بصورة تدعم `arm64` مع التحقق من توافقه.

## 4. إنشاء Oracle 18c XE لأول مرة فقط

**قبل الإنشاء** تأكد أن الحاوية غير موجودة:

```powershell
docker ps -a
```

إذا وجدت `oracle18c`، **لا تنفّذ `docker run`**؛ انتقل إلى التشغيل اليومي.

### 4.1 تنزيل صورة Oracle

في PowerShell على Windows أو Terminal على Mac Intel:

```bash
docker pull --platform linux/amd64 gvenzl/oracle-xe:18
```

### 4.2 إنشاء الحاوية وحفظ البيانات

اختر كلمة مرور قوية مكان `CHANGE_ME_STRONG_PASSWORD` (لبيئة تعليمية فقط)، ثم نفّذ **مرة واحدة**:

```bash
docker run -d --name oracle18c --platform linux/amd64 --restart unless-stopped -p 1522:1521 -e ORACLE_PASSWORD=CHANGE_ME_STRONG_PASSWORD -v oracle18c-data:/opt/oracle/oradata gvenzl/oracle-xe:18
```

> **أمان:** كلمة المرور داخل `docker run` قد تظهر في سجل الطرفية وبيانات إعداد الحاوية؛ لا تستخدم كلمة مرور حقيقية لحساب آخر. استخدم كلمة مرور خاصة بهذه البيئة ولا ترفعها إلى GitHub.

**معنى الإعدادات:**

| الإعداد | القيمة |
|---|---|
| اسم الحاوية | `oracle18c` |
| منفذ الجهاز (Windows/Mac) | `1522` |
| منفذ Oracle داخل الحاوية | `1521` |
| اسم الخدمة | `XEPDB1` |
| تخزين البيانات | Docker volume باسم `oracle18c-data` |

تابع بدء التشغيل:

```bash
docker logs -f oracle18c
```

انتظر حتى تظهر رسالة تفيد بجاهزية قاعدة البيانات. اضغط `Ctrl+C` لإيقاف عرض السجلات فقط، **وليس لإيقاف Oracle**.

## 5. الدخول إلى Oracle من داخل Docker

في PowerShell أو Terminal:

```bash
docker exec -it oracle18c bash
```

أنت الآن داخل الحاوية (`bash-4.4$`). نفّذ:

```bash
sqlplus / as sysdba
```

أنت الآن عند `SQL>`؛ اختر قاعدة البيانات الفرعية:

```sql
ALTER SESSION SET CONTAINER = XEPDB1;
SHOW CON_NAME;
```

يجب أن يظهر `XEPDB1`.

للخروج من SQL*Plus ثم الحاوية:

```sql
EXIT;
```

```bash
exit
```

## 6. تثبيت HR Sample Schema (مرة واحدة على جهاز جديد فقط)

> **لا تُعد تثبيت HR إذا كانت موجودة:** بعض سكربتات التثبيت تحذف مستخدم `HR` وكل بياناته قبل إعادة إنشائه. تحقق أولًا من وجودها.

### 6.1 تنزيل ملفات HR

نزّل [ملف v18c ZIP](https://github.com/oracle-samples/db-sample-schemas/archive/refs/tags/v18c.zip) وفك الضغط. المجلد الذي يحتوي `human_resources/hr_cre.sql` هو المجلد الذي ستنسخه إلى Docker.

**Windows — مثال إذا فككت الضغط في Downloads:**

```powershell
docker cp "C:\Users\YOUR_USER\Downloads\db-sample-schemas-18c\db-sample-schemas-18c" oracle18c:/tmp/db-sample-schemas-18c
```

غيّر `YOUR_USER` والمسار بحسب مكان فك الضغط الفعلي.

**macOS Intel — مثال إذا فككت الضغط في Downloads:**

```bash
docker cp "$HOME/Downloads/db-sample-schemas-18c/db-sample-schemas-18c" oracle18c:/tmp/db-sample-schemas-18c
```

**تحقق من وجود الملفات** في PowerShell أو Terminal:

```bash
docker exec oracle18c ls /tmp/db-sample-schemas-18c/human_resources/hr_cre.sql
```

إذا ظهر `No such file`، راجع اسم المجلد الفعلي بعد فك الضغط قبل الاستمرار.

### 6.2 تحقق أن HR غير موجودة

```bash
docker exec -it oracle18c bash
sqlplus / as sysdba
```

داخل `SQL>`:

```sql
ALTER SESSION SET CONTAINER = XEPDB1;
SELECT username FROM dba_users WHERE username = 'HR';
```

إذا ظهر `HR`، **توقف** ولا تعِد الإنشاء؛ جرّب الاتصال بالمستخدم الموجود أولًا. إذا ظهرت `no rows selected`، أكمل.

### 6.3 إنشاء مستخدم HR

> المثال يستخدم كلمة مرور تدريبية بسيطة `Hr123456` لتجنب مشكلة الرموز الخاصة في سكربت Oracle القديم. **غيّرها إلى كلمة مرور قوية بعد التثبيت، ولا تستخدمها في بيئة إنتاجية.**

وأنت متصل بـ`SYSDBA` داخل `XEPDB1`:

```sql
CREATE USER HR IDENTIFIED BY Hr123456;
ALTER USER HR DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;
ALTER USER HR TEMPORARY TABLESPACE TEMP;
GRANT CREATE SESSION, CREATE VIEW, ALTER SESSION, CREATE SEQUENCE, CREATE SYNONYM, CREATE DATABASE LINK, RESOURCE TO HR;
GRANT EXECUTE ON SYS.DBMS_STATS TO HR;
```

اتصل بمستخدم HR باستخدام **صيغة تطلب كلمة المرور تفاعليًا**:

```sql
CONNECT HR@//localhost:1521/XEPDB1
SHOW USER;
```

أدخل كلمة مرور HR عندما تُطلب. يجب أن يظهر `USER is "HR"`. **إذا فشل الاتصال، لا تشغّل السكربتات التالية.**

### 6.4 إنشاء الجداول وإضافة البيانات

وأنت عند `SQL>` متصل بمستخدم `HR`، نفّذ كل سكربت على حدة، وتوقف إذا ظهر خطأ:

```sql
@/tmp/db-sample-schemas-18c/human_resources/hr_cre.sql
@/tmp/db-sample-schemas-18c/human_resources/hr_popul.sql
@/tmp/db-sample-schemas-18c/human_resources/hr_idx.sql
@/tmp/db-sample-schemas-18c/human_resources/hr_code.sql
@/tmp/db-sample-schemas-18c/human_resources/hr_comnt.sql
@/tmp/db-sample-schemas-18c/human_resources/hr_analz.sql
COMMIT;
```

**تحقق:**

```sql
SELECT COUNT(*) FROM EMPLOYEES;
SELECT table_name FROM user_tables ORDER BY table_name;
```

عدد الموظفين المتوقع في بيانات HR الرسمية هو **107**. الجداول الأساسية: `REGIONS`, `COUNTRIES`, `LOCATIONS`, `DEPARTMENTS`, `JOBS`, `EMPLOYEES`, `JOB_HISTORY`.

> ملاحظة: لا تشغّل `hr_main.sql` بعد إنشاء HR يدويًا؛ يحتوي `DROP USER hr CASCADE`، وقد يحذف ما أنشأته.

## 7. التشغيل اليومي (Windows / macOS)

1. افتح **Docker Desktop** وانتظر جاهزية المحرك.
2. افتح **PowerShell** في Windows أو **Terminal** في macOS.
3. تحقق من حالة الحاوية:

```bash
docker ps -a
```

4. إذا كانت `oracle18c` متوقفة، شغّلها:

```bash
docker start oracle18c
```

5. تحقق من أنها تعمل:

```bash
docker ps
docker logs --tail 30 oracle18c
```

> بسبب `--restart unless-stopped` قد تكون الحاوية بدأت تلقائيًا. إذا كانت تعمل، لا تحتاج إلى `docker start`. لا تحتاج إلى تشغيل WSL يدويًا.

## 8. تنزيل SQL Developer والاتصال بـ HR

1. نزّل [SQL Developer](https://www.oracle.com/database/sqldeveloper/technologies/download/) بالنسخة المناسبة لـWindows أو macOS.
2. افتح التطبيق (قد تحتاج JDK متوافقًا بحسب حزمة التحميل).
3. اختر **New Database Connection** وأدخل:

| الحقل | القيمة |
|---|---|
| Connection Name | `HR-Docker` |
| Username | `HR` |
| Password | كلمة مرور HR التي أنشأتها |
| Connection Type | `Basic` |
| Hostname | `localhost` |
| Port | `1522` |
| Service Name | `XEPDB1` (**ليس SID**) |

4. اضغط **Test** ثم **Connect**.
5. افتح SQL Worksheet ونفّذ:

```sql
SELECT COUNT(*) FROM EMPLOYEES;
SELECT * FROM DEPARTMENTS;
```

إذا كان SQL Developer على **جهاز مختلف** عن Docker، استبدل `localhost` بعنوان IP أو اسم خادم Docker مع التأكد من الشبكة وجدار الحماية.

## 9. الاتصال بـ Oracle وSQL Server باستخدام DBeaver Community

**DBeaver Community** بديل مناسب لو عايز تدير Oracle وSQL Server من نفس التطبيق على **Windows أو macOS**.

1. نزّل البرنامج من [صفحة DBeaver الرسمية](https://dbeaver.io/download/) واختر إصدار Windows أو macOS المناسب لمعالج جهازك.
2. افتح DBeaver واضغط **New Database Connection** (علامة التوصيل الجديدة).
3. اختر **Oracle**، واسمح بتنزيل JDBC Driver عند الطلب (قد يتطلب اتصال إنترنت).
4. أدخل بيانات Oracle Container:

| الحقل | القيمة |
|---|---|
| Host | `localhost` |
| Port | `1522` |
| Database / Service Name | `XEPDB1` (اختر **Service Name** لا SID) |
| Username | `HR` |
| Password | كلمة مرور HR الخاصة بك |

5. اضغط **Test Connection** ثم **Finish**، وبعدها افتح SQL Editor واختبر:

```sql
SELECT COUNT(*) FROM EMPLOYEES;
```

### إضافة SQL Server إلى DBeaver (اختياري)

إذا كان عندك SQL Server Container على نفس الجهاز، اختر **New Database Connection → SQL Server**، واستخدم المنفذ المنشور على الجهاز (غالبًا `1433`، لكن تأكد من `docker ps`). أدخل اسم المستخدم وكلمة المرور الخاصة بـSQL Server، ثم **Test Connection**. قد تحتاج ضبط إعدادات التشفير/الثقة بالشهادة حسب بيئتك.

> **ملاحظة:** Azure Data Studio مخصص أساسًا لـSQL Server، وقد انتهى دعمه من Microsoft. لإدارة Oracle استخدم **DBeaver** أو **Oracle SQL Developer**.

## 10. إيقاف Oracle بدون حذف البيانات

```bash
docker stop oracle18c
```

لتشغيلها مرة أخرى:

```bash
docker start oracle18c
```

**لا تنفّذ** `docker rm oracle18c` أو `docker volume rm oracle18c-data` إذا كنت تريد الاحتفاظ ببياناتك.

## 11. حل المشكلات الشائعة

| المشكلة | الحل |
|---|---|
| `wsl --update` مطلوب | نفّذه من PowerShell كمسؤول، وأعد التشغيل إذا طلب Windows |
| `docker` غير معروف | ثبّت Docker Desktop ثم افتح نافذة طرفية جديدة |
| Cannot connect to Docker daemon | افتح Docker Desktop وانتظر جاهزية Engine |
| اسم `oracle18c` مستخدم | الحاوية موجودة؛ استخدم `docker start oracle18c` بدل `docker run` |
| Port `1521` مستخدم | استخدم الربط `-p 1522:1521` عند **إنشاء حاوية جديدة فقط** |
| SQL Developer: Connection refused | تحقق من `docker ps` وجاهزية Oracle، ثم `localhost:1522/XEPDB1` |
| `ORA-01017` | تحقق من اسم المستخدم وكلمة المرور |
| `ORA-01918` عند حذف HR | المستخدم غير موجود؛ لا يلزم حذفه لإنشاء HR جديد |
| `ORA-00922` عند `CREATE USER` | قد تكون كلمة المرور ذات الرموز الخاصة غير مقتبسة في سكربت قديم |
| `ORA-12154` | تحقق من صيغة الاتصال `//localhost:1521/XEPDB1` **داخل** الحاوية |
| `SP2-0640: Not connected` | الاتصال انقطع؛ أعد الاتصال أولًا قبل تشغيل أي سكربت |
| Mac Apple Silicon | لا تفترض توافق Oracle 18c `amd64`؛ استخدم خادم `amd64` أو صورة Oracle تدعم `arm64` |

---

### ملاحظات GitHub

- هذا الدليل لا يحتوي على كلمة مرور قاعدة بياناتك الفعلية.
- **لا ترفع** ملفات `.env` أو كلمات المرور أو بيانات Docker volumes إلى المستودع.
- مسار مقترح داخل مشروع موجود: `docs/oracle-docker/README.md`.
- إذا كان هذا مستودعًا مستقلًا، استخدم `README.md` في جذر المستودع.

---

## About the Author | عن المؤلف

**Mohamed Al-Swaify (محمد السويفي)**  
**Software Engineer | مهندس برمجيات**  
**Master’s in Computer Science & Artificial Intelligence | ماجستير علوم الحاسب والذكاء الاصطناعي**

*Prepared and maintained by Mohamed Al-Swaify.*
