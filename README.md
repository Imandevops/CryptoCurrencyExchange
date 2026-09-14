# CryptoCurrency Exchange

اپلیکیشن صرافی ارز دیجیتال بر پایهٔ Django 4.1 که با Gunicorn اجرا می‌شود و
manifestهای Kubernetes برای PostgreSQL  دارد.

## پیش‌نیازهای Production

1. image را با `dockerfile` نسخه‌دار بسازید. container با کاربر non-root دارای
   UID/GID برابر `10001` اجرا می‌شود و Gunicorn روی پورت `8000` گوش می‌دهد.
2. Secret با نام `cryptocurrency-exchange-secrets` را در namespace `crypto` از
   Secret Manager یا متغیرهای امن CI بسازید. مقدار واقعی `DJANGO_SECRET_KEY` و
   رمزهای دیتابیس را هرگز commit نکنید. فایل `secret.example.yaml` فقط template است.
3. نام image نمونه در `deployment.yaml` را با image واقعی registry خود جایگزین کنید.
   در Production، image باید تا جای ممکن با digest غیرقابل‌تغییر pin شود.
4. فایل‌های `configmap.yaml`، `secret.example.yaml` (پس از جایگذاری امن مقدارها)،
   `postgresql.yaml`، `deployment.yaml` و `backup.yaml` را در namespace `crypto`
   اعمال کنید. namespace باید از قبل وجود داشته باشد.

## طراحی CI/CD

Pipeline باید به دو بخش CI و CD تقسیم شود؛ image بدون نسخه را مستقیم روی nodeهای
Kubernetes نسازید.

1. با هر commit، تست‌ها و دستور `python manage.py check --deploy` را با تنظیمات
   امن CI اجرا کنید.
2. image را بسازید و با tag تغییرناپذیر مانند زیر push کنید:
   `REGISTRY/cryptocurrency-exchange:${GIT_COMMIT_SHA}`. برای release می‌توان یک
   tag قابل‌خواندن SemVer هم ساخت، اما Deployment باید از Commit SHA یا digest
   استفاده کند، نه `latest`.
3. مقدار image در `deployment.yaml` را در یک commit بررسی‌شده GitOps به SHA یا
   digest دقیق تغییر دهید. مرحلهٔ CD ابتدا برای آماده شدن PostgreSQL صبر می‌کند،
   سپس Job مهاجرت را اجرا و منتظر اتمام آن می‌ماند؛ در ادامه Deployment اپلیکیشن
   rollout می‌شود و Ready بودن replicaها بررسی می‌گردد.
4. پس از rollout، یک smoke test از طریق Service اجرا کنید. Readiness فعلی مسیر
   `/` را بررسی می‌کند که به Django و مسیر query دیتابیس می‌رسد.

نمونهٔ build و push؛ ورود به registry باید با Secretهای CI انجام شود:

```bash
IMAGE="registry.example.com/cryptocurrency-exchange:${GIT_COMMIT_SHA}"
docker build -f dockerfile -t "$IMAGE" .
docker push "$IMAGE"
```

## اجرای Migration دیتابیس

در `deployment.yaml` یک Kubernetes Job با نام
`cryptocurrency-exchange-migrate` وجود دارد که دستور
`python manage.py migrate --noinput` را اجرا می‌کند. این Job را برای هر release،
بعد از Ready شدن PostgreSQL و **قبل از** rollout کردن Deployment وب اجرا کنید.
Migration را در `CMD` کانتینر وب قرار ندهید؛ زیرا دو replica ممکن است هم‌زمان
برای اجرای migration تلاش کنند.

برای release امن از الگوی expand/contract Django استفاده کنید: ابتدا تغییر schema
سازگار با نسخهٔ قبلی را اضافه کنید، بعد کد سازگار اپلیکیشن را deploy کنید، و حذف
column یا constraint قدیمی را به یک release بعدی منتقل کنید. پیش از هر migration
تغییردهندهٔ schema، یک backup تأییدشده از دیتابیس بگیرید.

## Rollback

برای بازگرداندن اپلیکیشن، image Deployment را به SHA یا digest سالم قبلی برگردانید
و وضعیت rollout را بررسی کنید:

```bash
kubectl -n crypto rollout undo deployment/cryptocurrency-exchange
kubectl -n crypto rollout status deployment/cryptocurrency-exchange
```

Rollback اپلیکیشن فقط زمانی ایمن است که schema دیتابیس با نسخهٔ قبلی سازگار باشد.
Migrationهای Django را خودکار reverse نکنید. اگر rollback دیتابیس ضروری شد، ابتدا
نوشتن در اپلیکیشن را متوقف کنید، سپس یک backup تست‌شده را restore کنید یا دستور
بررسی‌شدهٔ `python manage.py migrate <app> <migration>` را اجرا نمایید؛ پیش از
بازگرداندن traffic نیز سرویس را اعتبارسنجی کنید.

## تغییرات ضروری برای Production

- برای Python، PostgreSQL و image اپلیکیشن پس از تست، از digest registry استفاده کنید.
- static fileها را با Ingress/static server سرو کنید یا middleware مناسب Django اضافه
  کنید؛ Gunicorn به تنهایی static assetها را سرو نمی‌کند.
- یک endpoint سبک و مستقل از authentication مانند `/healthz` اضافه و آن را برای
  liveness/readiness استفاده کنید؛ endpoint dashboard برای health check مناسب نیست.
- Secretهای Production را در External Secrets، Sealed Secrets، Vault یا CI secret
  store نگه دارید و هر Secretی که قبلاً commit شده است را rotate کنید.
- backupهای PostgreSQL را روی storage مستقل از PVC دیتابیس نگه دارید و restore را
  به‌صورت دوره‌ای تست کنید. `backup.yaml` از PVC محلی استفاده می‌کند و به‌تنهایی
  راهکار Disaster Recovery نیست.
- مقدار CPU، Memory، تعداد replica، probeها و connectionهای دیتابیس را با توجه به
  metricهای واقعی Production تنظیم کنید.
