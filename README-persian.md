[فارسی](README-persian.md) | [English](README.md)

# راه‌اندازی cAdvisor

ابزار cAdvisor (مخفف Container Advisor) یک ابزار متن‌باز است که توسط گوگل توسعه داده شده است. وظیفه اصلی آن بررسی، تحلیل و اکسپوز کردن دیتای مصرف منابع (CPU, RAM, Network و...) کانتینرهای در حال اجرا (مثل داکر) است.
از آنجایی که کانتینرها درون خودشان Node Exporter ندارند، cAdvisor عملاً نقش "Node Exporter برای کانتینرها" را بازی می‌کند.

## نصب cAdvisor به صورت باینری (سرویس systemd)

با وجود اینکه رایج‌ترین راه برای اجرای cAdvisor استفاده از داکر است، اما شما می‌تونید اون رو به صورت یک فایل باینری (سرویس systemd) هم روی ماشین خود بالا بیارید، **البته این روش اصلاً پیشنهاد نمیشه.**

### ۱. دانلود و نصب فایل باینری

```bash
# نکته: دقت کنید که معماری درست (مثلاً amd64) را برای سرور خود دانلود کنید
VERSION=$(curl -s https://api.github.com/repos/google/cadvisor/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget -O cadvisor https://github.com/google/cadvisor/releases/download/${VERSION}/cadvisor-${VERSION}-linux-amd64
chmod +x cadvisor
sudo mv cadvisor /usr/local/bin/
```

### ۲. ساخت سرویس systemd

> ### هشدار (چرا برای cAdvisor یوزر جدا نمی‌سازیم؟)
>
> برخلاف Node Exporter یا Prometheus که با یک یوزر ایزوله کار می‌کنند، **cAdvisor برای کار کردن نیاز به دسترسی Root دارد.** دلیلش این است که باید بتواند مصرف منابع کانتینرها را از مسیرهای محافظت‌شده‌ای مثل `/var/lib/docker` و سیستم `cgroup` کرنل لینوکس بخواند. اگر یک یوزر مجزا برای آن بسازید، با ارور **Permission Denied** مواجه شده و هیچ دیتایی از کانتینرها استخراج نمی‌شود.

فایل `/etc/systemd/system/cadvisor.service` را با ادیتور باز کنید:

```bash
sudo nano /etc/systemd/system/cadvisor.service
```

سپس تنظیمات زیر را در آن قرار دهید:

```ini
[Unit]
Description=cAdvisor
Wants=network-online.target
After=network-online.target

[Service]
# ابزار cAdvisor برای خواندن اطلاعات کانتینرها به دسترسی روت نیاز دارد
User=root
ExecStart=/usr/local/bin/cadvisor
Restart=always

[Install]
WantedBy=multi-user.target
```

حالا سرویس را فعال و اجرا کنید:

```bash
sudo systemctl daemon-reload
sudo systemctl enable cadvisor
sudo systemctl start cadvisor
```

اکنون cAdvisor روی پورت **8080** در دسترس است:

- `http://{IP_ADDRESS}:8080/metrics`

## راه‌اندازی cAdvisor با Docker Compose (پیشنهادی)

از آنجایی که وظیفه اصلی cAdvisor مانیتور کردن کانتینرهاست، اجرای آن به شکل یک کانتینر داکر کاملاً منطقی و استاندارد است.

یک فایل `docker-compose.yml` به شکل زیر بسازید:

```yaml
services:
  cadvisor:
    image: ghcr.io/google/cadvisor:v0.60.5
    container_name: cadvisor
    restart: unless-stopped
    devices:
      - /dev/kmsg:/dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
```

سپس کانتینر را اجرا کنید:

```bash
docker compose up -d
```

## اتصال پرومتئوس به cAdvisor

بعد از اجرای cAdvisor، باید پرومتئوس را کانفیگ کنید تا دیتای آن را اِسکریپ (Scrape) کند. فایل `prometheus.yml` را باز کرده و یک Job جدید اضافه کنید:

```yaml
scrape_configs:
  - job_name: "cadvisor"
    static_configs:
      - targets: ["{NODE_IP}:8080"]
```

> ### نکته مهم (فایروال)
>
> برای اینکه سرور پرومتئوس بتواند به cAdvisor وصل شود و دیتاها را جمع‌آوری کند، پورت `8080/tcp` ماشین شما باید باز و در دسترس باشد.
>
> اگر محدودیت شبکه دارید و نمی‌توانید پورت‌های ورودی (Inbound) سرور را باز کنید، راه‌حل این است که از **Prometheus Agent** روی ماشین استفاده کنید (در آینده در یک ریپازیتوری مجزا در مورد آن صحبت خواهیم کرد).

## داشبوردهای گرافانا برای cAdvisor

برای دیدن متریک‌های cAdvisor به شکل گرافیکی، می‌توانید از داشبوردهای آماده کامیونیتی استفاده کنید. چند نمونه از بهترین داشبوردهایی که می‌توانید تست کنید:

- [cAdvisor Exporter (14282)](https://grafana.com/grafana/dashboards/14282-cadvisor-exporter/)
- [cAdvisor Dashboard (19792)](https://grafana.com/grafana/dashboards/19792-cadvisor-dashboard/)
- [Docker Container Monitoring (19908)](https://grafana.com/grafana/dashboards/19908-docker-container-monitoring-with-prometheus-and-cadvisor/)

> ### توجه بسیار مهم (cgroup v2 و containerd snapshotter)
>
> ترجیحاً از نسخه‌های جدیدتر cAdvisor استفاده کنید. از آنجایی که لینوکس به سمت استفاده از **cgroup v2** رفته است و همچنین داکر به جای `overlay2` از **containerd snapshotter** استفاده می‌کند، نسخه‌های قدیمی cAdvisor با این تغییرات سازگار نیستند. حتی در زمان فعلی، نسخه `v0.60.5` با بخشی از تغییرات containerd snapshotter سازگاری کامل ندارد. بنابراین در صورت انتشار نسخه جدیدتر، حتماً از آن استفاده کنید.
>
> همچنین به یاد داشته باشید که بعضی از داشبوردهای قدیمی‌ترِ گرافانا صرفاً برای متریک‌های **cgroup v1** نوشته شده‌اند و برخی برای نسخه‌های جدید (v2) بهینه‌سازی شده‌اند. اگر داشبوردی ایمپورت کردید و بعضی پنل‌های آن خالی بود و خطای **No Data** می‌داد، به احتمال خیلی زیاد دلیلش این است که کوئری‌های آن داشبورد با ورژن cgroup سرور شما همخوانی ندارد.

### چگونه ورژن cgroup سرور را چک کنیم؟

خیلی ساده با اجرای دستور زیر در ترمینال لینوکس می‌توانید ورژن cgroup خود را متوجه بشید:

```bash
stat -fc %T /sys/fs/cgroup/
```

- اگر خروجی دستور `cgroup2fs` بود، سیستم شما از **cgroup v2** استفاده می‌کنه (که در اوبونتو 22.04 به بالا و لینوکس‌های مدرن رایجه).
- اگر خروجی `tmpfs` یا `cgroup` بود، سیستم شما احتمالاً از **cgroup v1** استفاده می‌کنه.

اگه داشبوردی که ایمپورت کردید روی سیستم شما کار نکرد، پیشنهاد می‌کنم داشبوردهای دیگه رو امتحان کنید یا اینکه خودتون به صورت دستی کوئری‌ها رو ادیت کنید و اسم متریک‌ها رو آپدیت کنید.
