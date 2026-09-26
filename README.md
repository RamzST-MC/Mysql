# Установка и настройка phpMyAdmin (Ubuntu + Nginx + PHP-FPM)

Пошаговое руководство по установке phpMyAdmin с последней официальной версии, настройке Nginx, SSL/TLS через Let's Encrypt и усилению безопасности.

## Содержание

- [Требования](#требования)
- [Пошаговая установка](#пошаговая-установка)
- [Настройка SSL/TLS сертификата](#настройка-ssltls-сертификата)
- [Критически важные настройки безопасности](#критически-важные-настройки-безопасности)
- [Сравнение альтернатив](#сравнение-альтернатив)
- [Автоматизация и скрипты](#автоматизация-и-скрипты)
- [Мониторинг и логирование](#мониторинг-и-логирование)
- [Интересные факты и нестандартные способы использования](#интересные-факты-и-нестандартные-способы-использования)
- [Типичные ошибки и их решения](#типичные-ошибки-и-их-решения)
- [Производительность и оптимизация](#производительность-и-оптимизация)
- [Альтернативные решения](#альтернативные-решения)
- [Заключение и рекомендации](#заключение-и-рекомендации)

## Требования

- Ubuntu Server (актуальная LTS-версия)
- Права `sudo`
- Настроенный домен/поддомен (для SSL и конфигурации Nginx)
- Установленный и работающий MySQL/MariaDB сервер

## Пошаговая установка

### Шаг 1: Обновление системы и установка зависимостей

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install nginx php8.3-fpm php8.3-mysql php8.3-mbstring php8.3-zip php8.3-gd php8.3-json php8.3-curl php8.3-xml wget unzip -y
```

### Шаг 2: Установка phpMyAdmin

Рекомендуется скачивать phpMyAdmin с официального сайта, а не из репозитория Ubuntu, чтобы получить последнюю версию:

```bash
cd /tmp
wget https://www.phpmyadmin.net/downloads/phpMyAdmin-latest-all-languages.tar.gz
tar xzf phpMyAdmin-latest-all-languages.tar.gz
sudo mv phpMyAdmin-*-all-languages /usr/share/phpmyadmin
sudo chown -R www-data:www-data /usr/share/phpmyadmin
```

### Шаг 3: Создание конфигурационного файла phpMyAdmin

```bash
sudo cp /usr/share/phpmyadmin/config.sample.inc.php /usr/share/phpmyadmin/config.inc.php
sudo nano /usr/share/phpmyadmin/config.inc.php
```

Найдите строку с `blowfish_secret` и замените на случайную строку длиной 32 символа:

```php
$cfg['blowfish_secret'] = 'your-32-character-random-string-here';
```

### Шаг 4: Настройка Nginx

Создайте конфигурационный файл для phpMyAdmin:

```bash
sudo nano /etc/nginx/sites-available/phpmyadmin
```

Добавьте следующую конфигурацию:

```nginx
server {
    listen 80;
    server_name phpmyadmin.yourdomain.com;
    root /usr/share/phpmyadmin;
    index index.php;

    # Безопасность: скрыть версию Nginx
    server_tokens off;

    # Ограничить доступ к служебным файлам
    location ~ ^/(libraries|templates|setup)/ {
        deny all;
    }

    location ~ ^/config.inc.php$ {
        deny all;
    }

    # Основная обработка PHP
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Обработка статических файлов
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, no-transform";
    }

    # Общие настройки безопасности
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Шаг 5: Активация сайта

```bash
sudo ln -s /etc/nginx/sites-available/phpmyadmin /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Настройка SSL/TLS сертификата

> ⚠️ Никогда не используйте phpMyAdmin без HTTPS!

Установка Let's Encrypt:

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d phpmyadmin.yourdomain.com
```

Обновлённая конфигурация Nginx для принудительного использования HTTPS:

```nginx
server {
    listen 80;
    server_name phpmyadmin.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name phpmyadmin.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/phpmyadmin.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/phpmyadmin.yourdomain.com/privkey.pem;

    # Современные SSL настройки
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Остальная конфигурация...
}
```

## Критически важные настройки безопасности

### 1. Ограничение доступа по IP

Добавьте в блок `server` следующие строки для ограничения доступа только с определённых IP:

```nginx
allow 192.168.1.0/24;  # Ваша локальная сеть
allow 203.0.113.0/24;  # Ваш офисный IP
deny all;
```

### 2. HTTP-аутентификация

Создайте дополнительный слой защиты:

```bash
sudo apt install apache2-utils -y
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

Добавьте в конфигурацию Nginx:

```nginx
auth_basic "Restricted Access";
auth_basic_user_file /etc/nginx/.htpasswd;
```

### 3. Настройка PHP-FPM для безопасности

Отредактируйте конфигурацию PHP-FPM:

```bash
sudo nano /etc/php/8.3/fpm/pool.d/www.conf
```

Измените следующие параметры:

```ini
security.limit_extensions = .php
php_admin_value[expose_php] = Off
php_admin_value[allow_url_fopen] = Off
php_admin_value[allow_url_include] = Off
```

## Сравнение альтернатив

| Решение | Плюсы | Минусы | Безопасность |
|---|---|---|---|
| phpMyAdmin | Простота, популярность, богатый функционал | Частые уязвимости, тяжелый | Средняя |
| Adminer | Один PHP-файл, быстрый, современный | Меньше функций | Высокая |
| MySQL Workbench | Профессиональный инструмент | Требует desktop, сложность | Высокая |
| CLI (mysql) | Максимальная безопасность | Кривая обучения | Очень высокая |

## Автоматизация и скрипты

Скрипт для автоматического резервного копирования конфигурации phpMyAdmin:

```bash
#!/bin/bash
# backup-phpmyadmin.sh

BACKUP_DIR="/backup/phpmyadmin"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Резервное копирование конфигурации
tar -czf $BACKUP_DIR/phpmyadmin_config_$DATE.tar.gz /usr/share/phpmyadmin/config.inc.php

# Резервное копирование конфигурации Nginx
tar -czf $BACKUP_DIR/nginx_phpmyadmin_$DATE.tar.gz /etc/nginx/sites-available/phpmyadmin

# Удаление старых бэкапов (старше 30 дней)
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
```

## Мониторинг и логирование

Настройте логирование для отслеживания подозрительной активности:

```bash
sudo nano /etc/nginx/sites-available/phpmyadmin
```

Добавьте в `server` block:

```nginx
access_log /var/log/nginx/phpmyadmin.access.log;
error_log /var/log/nginx/phpmyadmin.error.log;

# Логирование неудачных попыток входа
location = /index.php {
    try_files $uri =404;
    fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;

    # Дополнительные заголовки для логирования
    fastcgi_param HTTP_X_FORWARDED_FOR $remote_addr;
}
```

## Интересные факты и нестандартные способы использования

- **Multi-server setup**: phpMyAdmin может управлять несколькими серверами MySQL одновременно через конфигурацию `$cfg['Servers']`.
- **Кастомные темы**: можно создавать собственные темы для phpMyAdmin, изменяя CSS в папке `/themes/`.
- **API интеграция**: phpMyAdmin можно интегрировать с системами мониторинга через его SQL-интерфейс.
- **Автоматизация через curl**: некоторые операции можно автоматизировать через HTTP-запросы.

## Типичные ошибки и их решения

**Ошибка:** «The configuration file now needs a secret passphrase»
**Решение:** проверьте, что `blowfish_secret` содержит строку длиной 32 символа.

**Ошибка:** «Cannot connect: invalid settings»
**Решение:** убедитесь, что MySQL работает и пользователь имеет необходимые права.

**Ошибка:** 502 Bad Gateway
**Решение:** проверьте статус PHP-FPM и правильность сокета в конфигурации Nginx.

```bash
sudo systemctl status php8.3-fpm
sudo systemctl restart php8.3-fpm
```

## Производительность и оптимизация

Для улучшения производительности phpMyAdmin отредактируйте:

```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

Оптимизируйте следующие параметры:

```ini
memory_limit = 256M
max_execution_time = 300
max_input_vars = 3000
post_max_size = 64M
upload_max_filesize = 64M
```

## Альтернативные решения

Если безопасность критична, рассмотрите следующие альтернативы:

- [Adminer](https://www.adminer.org/) — легкий и безопасный аналог
- phpLiteAdmin — для работы с SQLite
- Sequel Pro — для macOS пользователей
- DataGrip — профессиональная IDE от JetBrains

## Заключение и рекомендации

phpMyAdmin остаётся мощным и удобным инструментом для администрирования MySQL/MariaDB, но его безопасность целиком зависит от правильной настройки. Ключевые рекомендации:

1. **Всегда используйте HTTPS** — это не опционально.
2. **Ограничивайте доступ по IP** — не делайте phpMyAdmin публично доступным.
3. **Используйте HTTP-аутентификацию** как дополнительный слой защиты.
4. **Регулярно обновляйте** phpMyAdmin до последней версии.
5. **Мониторьте логи** на предмет подозрительной активности.
