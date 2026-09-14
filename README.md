# CryptoCurrency Exchange

اپلیکیشن صرافی ارز دیجیتال بر پایهٔ Django 4.1 که با Gunicorn اجرا می‌شود و
manifestهای Kubernetes برای PostgreSQL تک‌نمونه دارد.

## پیش‌نیازهای Production

1. image را با `dockerfile` نسخه‌دار بسازید. container با کاربر non-root دارای
   UID/GID برابر `10001` اجرا می‌شود و Gunicorn روی پورت `8000` گوش می‌دهد.
2. Secret با نام `cryptocurrency-exchange-secrets` را در namespace `crypto` از
   Secret Manager یا متغیرهای امن CI بسازید. مقدار واقعی `DJANGO_SECRET_KEY` و
   رمزهای دیتابیس را هرگز commit نکنید. فایل `secret.example.yaml` فقط template است.
3. نام image نمونه در `deployment.yaml` را با image واقعی registry خود جایگزین کنید.
   در Production، image باید تا جای ممکن با digest غیرقابل‌تغییر pin شود.
4. فایل‌های `configmap.yaml`، `secret.example.yaml`، `postgresql.yaml`,
   `deployment.yaml` و `backup.yaml` را در namespace `crypto` اعمال کنید.

## طراحی CI/CD

Pipeline باید به دو بخش CI و CD تقسیم شود؛ image بدون نسخه را مستقیم روی nodeهای
Kubernetes نسازید.

1. با هر commit، تست‌ها و دستور زیر را با تنظیمات امن اجرا کنید:

   ```bash
   python manage.py check --deploy
