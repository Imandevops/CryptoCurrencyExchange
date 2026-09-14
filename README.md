# CryptoCurrency Exchange

این پروژه یک صرافی ارز دیجیتال با Django هست که با Gunicorn اجراش می‌کنیم و
دیتابیس PostgreSQL هم روی Kubernetes بالا میاد.

## قبل از رفتن روی Production

1. image رو با فایل `dockerfile` می‌سازیم. برنامه داخل کانتینر با user معمولی
   اجرا میشه، نه root. UID/GID این user برابر `10001` هست و Gunicorn روی پورت
   `8000` اجرا میشه.

2. Secret با نام `cryptocurrency-exchange-secrets` رو داخل namespace `crypto`
   می‌سازیم. مقدارهای `DJANGO_SECRET_KEY`، رمز دیتابیس و بقیه اطلاعات حساس رو
   فقط از Secret Manager یا متغیرهای امن CI می‌خونیم؛ داخل Git قرارشون نمی‌دیم.
   فایل `secret.example.yaml` فقط نمونه‌ست.

3. داخل `deployment.yaml` آدرس image نمونه رو با registry و image واقعی خودمون
   عوض می‌کنیم. برای Production بهتره image رو با digest ثابت deploy کنیم.

4. این فایل‌ها رو داخل namespace `crypto` apply می‌کنیم:

   ```bash
   kubectl apply -f configmap.yaml
   kubectl apply -f secret.example.yaml
   kubectl apply -f postgresql.yaml
   kubectl apply -f deployment.yaml
   kubectl apply -f backup.yaml
   ```

## روند CI/CD

Pipeline رو به دو بخش CI و CD تقسیم می‌کنیم؛ image رو مستقیم روی nodeهای
Kubernetes نمی‌سازیم.

1. با هر commit، تست‌ها و بررسی Django رو اجرا می‌کنیم:

   ```bash
   python manage.py check --deploy
   ```

2. image رو می‌سازیم و با Commit SHA tag می‌زنیم و push می‌کنیم:

   ```bash
   IMAGE="registry.example.com/cryptocurrency-exchange:${GIT_COMMIT_SHA}"
   docker build -f dockerfile -t "$IMAGE" .
   docker push "$IMAGE"
   ```

3. داخل `deployment.yaml` image جدید رو با همان Commit SHA یا digest می‌گذاریم.
   از `latest` استفاده نمی‌کنیم.

4. در مرحله Deploy، اول صبر می‌کنیم PostgreSQL آماده بشه، بعد migration رو اجرا
   می‌کنیم، بعد Deployment برنامه رو rollout می‌کنیم و منتظر Ready شدن podها
   می‌مونیم.

5. آخر کار از طریق Service یک تست ساده می‌گیریم تا مطمئن بشیم برنامه بالا اومده.

## Migration دیتابیس

داخل `deployment.yaml` یک Job داریم با نام:

```text
cryptocurrency-exchange-migrate
```

این Job دستور زیر رو اجرا می‌کنه:

```bash
python manage.py migrate --noinput
```

Migration رو بعد از آماده شدن PostgreSQL و قبل از بالا آوردن نسخه جدید برنامه
اجرا می‌کنیم.

Migration رو داخل `CMD` کانتینر وب نمی‌گذاریم؛ چون اگر دو replica هم‌زمان بالا
بیان، ممکنه هر دو بخوان migration رو اجرا کنن.

برای تغییرات مهم دیتابیس این ترتیب رو نگه می‌داریم:

1. اول تغییر سازگار با نسخه قبلی رو به دیتابیس اضافه می‌کنیم.
2. بعد نسخه جدید برنامه رو deploy می‌کنیم.
3. حذف column یا constraint قدیمی رو برای release بعدی می‌گذاریم.
4. قبل از migration مهم، backup می‌گیریم.

## Rollback

اگر نسخه جدید مشکل داشت، برنامه رو به image قبلی برمی‌گردونیم:

```bash
kubectl -n crypto rollout undo deployment/cryptocurrency-exchange
kubectl -n crypto rollout status deployment/cryptocurrency-exchange
```

Rollback برنامه زمانی بدون دردسر انجام میشه که تغییرات دیتابیس با نسخه قبلی
سازگار باشن.

Migration دیتابیس رو خودکار برنمی‌گردونیم. اگر واقعاً لازم شد دیتابیس رو rollback
کنیم، اول نوشتن داخل برنامه رو متوقف می‌کنیم، بعد backup تست‌شده رو restore
می‌کنیم یا migration مشخص رو برمی‌گردونیم:

```bash
python manage.py migrate <app> <migration>
```

## کارهایی که برای Production بهتره انجام بدیم

- imageهای Python، PostgreSQL و برنامه رو با digest ثابت deploy کنیم.
- static fileها رو با Ingress یا NGINX/static server سرو کنیم؛ Gunicorn خودش برای
  سرو static مناسب نیست.
- یک endpoint مثل `/healthz` اضافه کنیم تا readiness و liveness probeها به صفحه
  اصلی برنامه وابسته نباشن.
- Secretها رو داخل Vault، External Secrets، Sealed Secrets یا CI Secret Store
  نگه داریم.
- backup دیتابیس رو روی storage جدا از PVC اصلی نگه داریم و restore رو هر چند وقت
  یک‌بار تست کنیم.
- CPU، Memory، تعداد replicaها، probeها و connectionهای دیتابیس رو با توجه به
  مصرف واقعی Production تنظیم کنیم.
