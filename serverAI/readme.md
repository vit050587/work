# Полное руководство по развертыванию сервисов NTC Sphere
📋 Оглавление
Общая архитектура

Требования

Настройка MikroTik

Настройка DNS

Подготовка сервера Ubuntu

Настройка Docker и Docker Compose

Конфигурация сервисов

Запуск и проверка

Устранение неполадок

Безопасность и обслуживание

🏗️ Общая архитектура
text
┌─────────────────────────────────────────────────────────────────┐
│                     Внешняя сеть (WAN)                          │
│                         86.62.91.34                             │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                 ┌──────▼────────┐
                 │   MikroTik    │
                 │  Роутер/Фаервол │
                 └──────┬────────┘
                        │
                ┌───────▼──────────┐
                │  Внутренняя сеть  │
                │  192.168.200.0/24 │
                └───────┬──────────┘
                        │
                 ┌──────▼──────────────┐
                 │ Сервер Ubuntu       │
                 │ 192.168.200.220     │
                 │ Docker + Caddy      │
                 └─────────────────────┘
📋 Требования
Оборудование:
Сервер Ubuntu 20.04+ (рекомендуется 22.04 LTS)

MikroTik роутер с внешним IP

Минимум 4GB RAM, 50GB HDD

Софт:
Docker и Docker Compose

SSH доступ к серверу

Доступ к MikroTik WebFig или WinBox

🔧 Настройка MikroTik
1. Настройка статического IP для сервера
bash
# В MikroTik:
/ip dhcp-server lease
add address=192.168.200.220 mac-address=XX:XX:XX:XX:XX:XX server=dhcp1
comment="NTC Sphere Server" disabled=no
2. Настройка NAT (проброс портов)
bash
# Проброс порта 80 (HTTP)
/ip firewall nat add chain=dstnat dst-address=86.62.91.34 protocol=tcp dst-port=80 \
  action=dst-nat to-addresses=192.168.200.220 to-ports=80

# Проброс порта 443 (HTTPS)
/ip firewall nat add chain=dstnat dst-address=86.62.91.34 protocol=tcp dst-port=443 \
  action=dst-nat to-addresses=192.168.200.220 to-ports=443
3. Настройка правил фаервола
bash
# Разрешить входящий трафик на порты 80/443
/ip firewall filter add chain=input protocol=tcp dst-port=80,443 action=accept \
  comment="Allow HTTP/HTTPS from WAN"

# Разрешить форвардинг трафика на сервер
/ip firewall filter add chain=forward dst-address=192.168.200.220 protocol=tcp \
  dst-port=80,443 action=accept comment="Forward to NTC Sphere Server"

# Важно: Добавить это правило ПОСЛЕ reject правил!
# Найдите номер reject правила:
/ip firewall filter print where action=reject

# Разместите новое правило перед reject:
/ip firewall filter move [find comment="Forward to NTC Sphere Server"] 0
4. Проверка настроек
bash
# Проверить все NAT правила
/ip firewall nat print

# Проверить фильтры
/ip firewall filter print

# Проверить активные соединения
/ip firewall connection print where dst-port=80 or dst-port=443
🌐 Настройка DNS
1. Создание A записей у DNS провайдера
text
n8n.ntcsphere.com    A    86.62.91.34    TTL: 3600
webui.ntcsphere.com  A    86.62.91.34    TTL: 3600
pgadmin.ntcsphere.com A    86.62.91.34    TTL: 3600
2. Проверка DNS записей
bash
# Из внешней сети
nslookup n8n.ntcsphere.com 8.8.8.8

# Из внутренней сети
nslookup n8n.ntcsphere.com
🖥️ Подготовка сервера Ubuntu
1. Обновление системы
bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git ufw htop
2. Настройка часового пояса
bash
sudo timedatectl set-timezone Europe/Moscow
3. Настройка SSH (опционально)
bash
# Смена порта SSH
sudo nano /etc/ssh/sshd_config
# Port 2222
# PermitRootLogin no
# PasswordAuthentication no

sudo systemctl restart sshd
4. Настройка firewall (UFW)
bash
# Открыть только необходимые порты
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
🐳 Настройка Docker и Docker Compose
1. Установка Docker
bash
# Удаляем старые версии
sudo apt remove docker docker-engine docker.io containerd runc

# Устанавливаем зависимости
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Добавляем GPG ключ Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Добавляем репозиторий
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Устанавливаем Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Проверяем установку
sudo docker --version
2. Настройка Docker без sudo
bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker

# Проверяем
docker run hello-world
3. Создание структуры папок
bash
mkdir -p ~/ntcsphere/{caddy/{data,config},postgres/{data,init},redis/data,open-webui/data,n8n/data,docling/data,pgadmin/data,ollama/models}
cd ~/ntcsphere
4. Создание файла переменных окружения
bash
nano .env
env
# PostgreSQL суперпользователь
POSTGRES_SUPERUSER=ntcsphere_admin
POSTGRES_SUPERPASS=Strong_Password_123!

# OpenAI API ключ (для n8n и Open WebUI)
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# PgAdmin учетные данные
PGADMIN_EMAIL=admin@ntcsphere.com
PGADMIN_PASSWORD=PgAdmin_Strong_Pass_123!

# Дополнительные настройки (опционально)
TZ=Europe/Moscow
⚙️ Конфигурация сервисов
1. Создание Caddyfile
bash
nano Caddyfile
caddy
{
    email ai.system@ntcsphere.com
    admin off
}

# Open WebUI
webui.ntcsphere.com {
    reverse_proxy open-webui:8080
}

# n8n
n8n.ntcsphere.com {
    reverse_proxy n8n:5678
}

# pgAdmin на порту 5050
pgadmin.ntcsphere.com {
    reverse_proxy pgadmin:5050
}
2. Создание docker-compose.yaml
bash
nano docker-compose.yaml
yaml
version: '3.8'

networks:
  webnet:
    driver: bridge

services:
  caddy:
    image: caddy:2.8
    container_name: caddy
    restart: unless-stopped
    ports:
      - "443:443"
      - "80:80"
      - "443:443/udp"  # Для HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy/data:/data
      - ./caddy/config:/config
    networks: [webnet]
    extra_hosts:
      - "host.docker.internal:host-gateway"

  postgres:
    image: postgres:16
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_SUPERUSER}
      POSTGRES_PASSWORD: ${POSTGRES_SUPERPASS}
      POSTGRES_DB: postgres
      TZ: ${TZ}
    volumes:
      - ./postgres/data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d
    networks: [webnet]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_SUPERUSER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - ./redis/data:/data
    networks: [webnet]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    environment:
      OLLAMA_HOST: 0.0.0.0
      OLLAMA_KEEP_ALIVE: 24h
      OLLAMA_NUM_PARALLEL: 2
    volumes:
      - ./ollama/models:/root/.ollama
    networks: [webnet]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]  # Если есть GPU

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    environment:
      DATABASE_URL: postgresql://${POSTGRES_SUPERUSER}:${POSTGRES_SUPERPASS}@postgres:5432/openwebui
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      OLLAMA_BASE_URL: http://ollama:11434
      WEBUI_AUTH: "true"
      ENABLE_SIGNUP: "true"
      REQUIRE_USER_LOGIN: "true"
      ALLOW_GUEST_USERS: "false"
      ALLOWED_EMAIL_DOMAINS: "ntcsphere.com"
      REQUIRE_EMAIL_DOMAIN: "true"
      REQUIRE_EMAIL_VERIFICATION: "false"
      EMAIL_VERIFICATION: "false"
      DISABLE_SIGNUP: "false"
      APP_TITLE: "NTC Sphere AI Assistant"
      APP_DESCRIPTION: "Корпоративный AI ассистент NTC Sphere"
      WEBUI_SECRET_KEY: "change-this-to-a-random-secret-key"
    volumes:
      - ./open-webui/data:/app/backend/data
    networks: [webnet]
    depends_on:
      postgres:
        condition: service_healthy
      ollama:
        condition: service_started

  n8n:
    image: n8nio/n8n:2.0.3
    container_name: n8n
    restart: unless-stopped
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: ${POSTGRES_SUPERUSER}
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_SUPERPASS}
      N8N_HOST: n8n.ntcsphere.com
      N8N_PROTOCOL: https
      N8N_PORT: 5678
      N8N_EDITOR_BASE_URL: https://n8n.ntcsphere.com
      WEBHOOK_URL: https://n8n.ntcsphere.com
      N8N_ENDPOINT_WEBHOOK: webhook
      N8N_PROXY_HOPS: 1
      EXPRESS_TRUST_PROXY: true
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      N8N_METRICS: "false"
      N8N_ENABLE_PRODUCTION_WEBHOOKS_ON_MAIN_PROCESS: "false"
      NODE_ENV: production
      WEBHOOK_TUNNEL_URL: https://n8n.ntcsphere.com/
    volumes:
      - ./n8n/data:/home/node/.n8n
    networks: [webnet]
    depends_on:
      postgres:
        condition: service_healthy

  docling:
    image: ghcr.io/docling-project/docling-serve:latest
    container_name: docling
    restart: unless-stopped
    environment:
      DATABASE_URL: postgresql://${POSTGRES_SUPERUSER}:${POSTGRES_SUPERPASS}@postgres:5432/docling
    volumes:
      - ./docling/data:/data
    networks: [webnet]
    depends_on:
      postgres:
        condition: service_healthy

  pgadmin:
    image: dpage/pgadmin4:8
    container_name: pgadmin
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD}
      PGADMIN_LISTEN_PORT: 5050
      PGADMIN_CONFIG_SERVER_MODE: "False"
      PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED: "False"
    volumes:
      - ./pgadmin/data:/var/lib/pgadmin
      - ./pgadmin/servers.json:/pgadmin4/servers.json  # Для предварительной настройки
    networks: [webnet]
    depends_on:
      - postgres
3. Создание скрипта инициализации базы данных
bash
nano postgres/init/01-init-databases.sql
sql
-- Создание баз данных для сервисов
CREATE DATABASE openwebui;
CREATE DATABASE n8n;
CREATE DATABASE docling;

-- Создание пользователя с ограниченными правами (опционально)
CREATE USER ntcsphere_app WITH PASSWORD 'App_Password_123!';

-- Назначение прав на базы данных
GRANT ALL PRIVILEGES ON DATABASE openwebui TO ntcsphere_app;
GRANT ALL PRIVILEGES ON DATABASE n8n TO ntcsphere_app;
GRANT ALL PRIVILEGES ON DATABASE docling TO ntcsphere_app;

-- Создание расширений
\c openwebui
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

\c n8n
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

\c docling
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
4. Создание конфигурации pgAdmin
bash
nano pgadmin/servers.json
json
{
    "Servers": {
        "1": {
            "Name": "NTC Sphere PostgreSQL",
            "Group": "Servers",
            "Host": "postgres",
            "Port": 5432,
            "MaintenanceDB": "postgres",
            "Username": "${POSTGRES_SUPERUSER}",
            "PassFile": "/pgpass",
            "SSLMode": "prefer",
            "Timeout": 10,
            "UseSSHTunnel": 0,
            "TunnelPort": "22",
            "TunnelAuthentication": 0
        }
    }
}
bash
# Создание файла с паролем
echo "postgres:5432:*:${POSTGRES_SUPERUSER}:${POSTGRES_SUPERPASS}" > pgadmin/pgpass
chmod 600 pgadmin/pgpass
🚀 Запуск и проверка
1. Запуск сервисов
bash
# Проверка конфигурации
docker-compose config

# Запуск в фоновом режиме
docker-compose up -d

# Просмотр логов
docker-compose logs -f

# Проверка состояния контейнеров
docker-compose ps
docker-compose logs --tail=50
2. Проверка работы сервисов
bash
# Проверка Caddy
curl -I https://n8n.ntcsphere.com
curl -I https://webui.ntcsphere.com

# Проверка контейнеров
docker exec -it postgres psql -U ${POSTGRES_SUPERUSER} -c "\l"

# Проверка здоровья
docker-compose exec postgres pg_isready -U ${POSTGRES_SUPERUSER}
3. Настройка моделей Ollama
bash
# Загрузка моделей
docker exec -it ollama ollama pull llama2
docker exec -it ollama ollama pull mistral

# Список моделей
docker exec -it ollama ollama list
4. Настройка Open WebUI
Откройте https://webui.ntcsphere.com

Зарегистрируйте первого пользователя с email @ntcsphere.com

Настройте подключение к Ollama в настройках

Добавьте OpenAI API ключ для доступа к GPT моделям

5. Настройка n8n
Откройте https://n8n.ntcsphere.com

Завершите начальную настройку

Настройте переменные окружения в n8n

Проверьте работу вебхуков

🔧 Устранение неполадок
1. Проверка сети
bash
# Проверка DNS
nslookup n8n.ntcsphere.com 8.8.8.8

# Проверка портов снаружи
telnet 86.62.91.34 443
nc -zv 86.62.91.34 443

# Проверка портов изнутри
ss -tulpn | grep -E ':80|:443'
2. Логи Caddy
bash
docker logs caddy --tail 100 --follow

# Детальные логи
docker exec caddy caddy validate --config /etc/caddy/Caddyfile
3. Проверка подключения между контейнерами
bash
# Из контейнера Caddy к бэкендам
docker exec caddy curl -I http://open-webui:8080
docker exec caddy curl -I http://n8n:5678

# Проверка сети Docker
docker network inspect ntcsphere_webnet
4. Проблемы с SSL сертификатами
bash
# Проверка сертификатов
docker exec caddy ls -la /data/caddy/certificates/

# Принудительное обновление
docker-compose restart caddy
5. Очистка и перезапуск
bash
# Остановка всех сервисов
docker-compose down

# Удаление томов (осторожно!)
docker-compose down -v

# Пересборка
docker-compose build --no-cache
docker-compose up -d
🔒 Безопасность и обслуживание
1. Регулярные обновления
bash
# Обновление образов
docker-compose pull
docker-compose up -d

# Очистка Docker
docker system prune -a --volumes
2. Резервное копирование
bash
#!/bin/bash
# backup.sh
BACKUP_DIR="/backup/ntcsphere"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR/$DATE

# Дамп базы данных
docker exec postgres pg_dumpall -U ${POSTGRES_SUPERUSER} > $BACKUP_DIR/$DATE/postgres.sql

# Копирование данных
cp -r ~/ntcsphere/* $BACKUP_DIR/$DATE/

# Архивирование
tar -czf $BACKUP_DIR/ntcsphere_backup_$DATE.tar.gz -C $BACKUP_DIR/$DATE .

# Удаление старых бэкапов (старше 7 дней)
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/ntcsphere_backup_$DATE.tar.gz"
3. Мониторинг
bash
# Установка cAdvisor для мониторинга
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8081:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor:v0.47.0
4. Обновление конфигурации
bash
# Создание новой версии конфигурации
git init
git add .
git commit -m "Initial configuration"

# Обновление
git pull origin main
docker-compose down
docker-compose up -d --build
📞 Поддержка и мониторинг
Полезные команды:
bash
# Просмотр использования ресурсов
docker stats

# Проверка логов в реальном времени
docker-compose logs -f --tail=50

# Вход в контейнер
docker exec -it postgres bash

# Проверка объема данных
du -sh ~/ntcsphere/*

# Мониторинг сетевого трафика
sudo nethogs
Критические точки проверки:
Сертификаты Let's Encrypt - обновляются автоматически каждые 90 дней

Дисковое пространство - особенно для моделей Ollama

Память - Open WebUI и n8n могут потреблять много памяти

Бэкапы - регулярное резервное копирование обязательно

🎯 Заключение
Теперь у вас полностью настроенная платформа NTC Sphere с:

✅ Open WebUI - AI ассистент с поддержкой локальных и облачных моделей
✅ n8n - платформа автоматизации рабочих процессов
✅ PostgreSQL - централизованная база данных
✅ PgAdmin - веб-интерфейс управления БД
✅ Ollama - локальное выполнение LLM моделей
✅ Redis - кэширование и сессии
✅ Caddy - автоматическое SSL и reverse proxy

Все сервисы доступны как из внутренней сети, так и извне через защищенные доменные имена.

🔄 Чеклист развертывания
Настройка MikroTik (NAT + firewall)

Настройка DNS записей

Подготовка сервера Ubuntu

Установка Docker и Docker Compose

Создание .env файла с секретами

Настройка Caddyfile

Настройка docker-compose.yaml

Запуск сервисов

Проверка доступа из LAN

Проверка доступа из WAN

Настройка моделей Ollama

Настройка пользователей Open WebUI

Настройка n8n

Настройка резервного копирования

Последнее обновление: 2026-02-06
Версия конфигурации: 1.0
Статус: Протестировано и работает ✅