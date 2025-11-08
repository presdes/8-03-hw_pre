# 🚀 Практическое руководство: Развертывание GitLab с GitLab Runner и SonarQube в Docker для веб-разработки

## 📋 Содержание

1. [Введение](#введение)
2. [Архитектура решения](#архитектура-решения)
3. [Предварительные требования](#предварительные-требования)
4. [Установка Docker и Docker Compose](#установка-docker-и-docker-compose)
5. [Подготовка окружения](#подготовка-окружения)
6. [Развертывание инфраструктуры](#развертывание-инфраструктуры)
7. [Настройка компонентов](#настройка-компонентов)
8. [Создание и настройка веб-проекта](#создание-и-настройка-веб-проекта)
9. [Интеграция CI/CD с SonarQube](#интеграция-cicd-с-sonarqube)
10. [Управление и мониторинг](#управление-и-мониторинг)
11. [Устранение неполадок](#устранение-неполадок)

---

## Введение

Это руководство предоставляет пошаговые инструкции по развертыванию собственного экземпляра GitLab, настроенного для веб-разработки (CSS, HTML, PHP, JavaScript), вместе с GitLab Runner и SonarQube в среде Docker. Мы настроим автоматизированный пайплайн CI/CD, который будет проверять код, включая анализ синтаксиса LESS, с помощью SonarQube и развертывать его.

---

## 🏗️ Архитектура решения

### Сетевая схема

```
+-----------------------------------------------------------------------+
| Docker Host (WSL2 Ubuntu / Физический сервер)                         |
|                                                                       |
|  +----------------+      +------------------+      +---------------+  |
|  | GitLab         |      | GitLab Runner    |      | SonarQube     |  |
|  | 192.168.56.10  |      | 192.168.56.30    |      | 192.168.56.20 |  |
|  | gitlab.localdomain|   |                  |      | sonarqube.localdomain|
|  | Ports: 80,443,2224 |  |                  |      | Port: 9000    |  |
|  +----------------+      +------------------+      +---------------+  |
|                                                                       |
|                  Docker Network: 192.168.56.0/24                     |
+-----------------------------------------------------------------------+
```

### Особенности для WSL2:

```
+-----------------------------------------------------------------------+
| Windows Host                    | WSL2 Ubuntu                        |
|                                 |                                     |
|  Браузер:                      |  +----------------+                 |
|  http://gitlab.localdomain     |  | GitLab         |                 |
|  http://localhost:9000         |  | 192.168.56.10  |                 |
|                                 |  +----------------+                 |
|  Hosts файл:                   |                                     |
|  C:\Windows\System32\drivers\  |  Docker Network: 192.168.56.0/24    |
|  etc\hosts                     |                                     |
+---------------------------------+-------------------------------------+
```

### Структура проекта

```
~/gitlab-setup/                      # Директория проекта Docker Compose
├── docker-compose.yml              # Конфигурация инфраструктуры
├── setup-hosts.sh                  # Скрипт настройки hosts
├── check-network.sh                # Скрипт проверки сети
├── wait-gitlab-working.sh          # Скрипт ожидания готовности GitLab
├── reset-project.sh                # Скрипт безопасной очистки проекта
└── full_reset_docker.sh            # Скрипт полной очистки Docker

/var/opt/gitlab/                    # ВНУТРИ КОНТЕЙНЕРА GitLab
├── git-data/
│   └── repositories/              # 👈 ЗДЕСЬ ХРАНЯТСЯ ПРОЕКТЫ GITLAB
│       ├── @hashed/               # Хэшированные пути к репозиториям
│       │   ├── ab/ct/.../         # Структура хэш-директорий
│       │   └── ...
│       └── user/                  # Пользовательские репозитории
│           ├── project1.git/      # Репозиторий project1
│           └── group/project2.git/# Репозиторий project2 в группе
├── postgresql/                    # База данных GitLab
└── uploads/                       # Загруженные файлы
```

### Расположение данных

- **Проекты GitLab**: `/var/opt/gitlab/git-data/repositories/` (внутри контейнера)
- **Конфигурация**: Docker volumes
- **Логи**: Docker volumes

---

## Предварительные требования

### Для всех систем:
- Установленные Docker и Docker Compose
- Минимум 8 ГБ оперативной памяти (6 ГБ для GitLab + 2 ГБ для SonarQube)
- Права sudo для настройки hosts файла
- Достаточное дисковое пространство

### Особые требования для WSL2:
- Windows 10 версии 2004 или выше / Windows 11
- Включенная функция WSL2
- Установленный дистрибутив Ubuntu из Microsoft Store
- Docker Desktop с включенной опцией WSL2 integration

### Оптимальные настройки WSL2
```
[wsl2]
memory=10GB
processors=4
swap=2GB
localhostForwarding=true
```
Универсальный скрипт для оптимальных настроек WSL2: setup-wsl-config.sh
```bash
#!/bin/bash

echo "=== 🛠️ НАСТРОЙКА WSL2 КОНФИГУРАЦИИ ==="

# Функция для поиска правильного пути
find_wsl_config_path() {
    local possible_paths=(
        "/mnt/c/Users/$USER/.wslconfig"
        "/mnt/c/Users/$(whoami)/.wslconfig"
        "/mnt/c/Users/$(echo $USER | tr '[:upper:]' '[:lower:]')/.wslcon>
        "/mnt/c/Users/$(echo $USER | tr '[:lower:]' '[:upper:]')/.wslcon>
    )

    for path in "${possible_paths[@]}"; do
        if [ -d "$(dirname "$path")" ]; then
            echo "$path"
            return 0
        fi
    done

    # Если стандартные пути не работают, покажем доступные варианты
    echo "🔍 Доступные пользователи в /mnt/c/Users/:"
    ls -la /mnt/c/Users/ | grep '^d'
    return 1
}

# Находим правильный путь
WSL_CONFIG_PATH=$(find_wsl_config_path)

if [ $? -ne 0 ]; then
    echo "❌ Не удалось автоматически определить путь"
    echo "📝 Пожалуйста, укажите путь вручную:"
    read -p "Введите полный путь к .wslconfig: " WSL_CONFIG_PATH
fi

# Создаем резервную копию если файл существует
if [ -f "$WSL_CONFIG_PATH" ]; then
    BACKUP_PATH="${WSL_CONFIG_PATH}.backup.$(date +%Y%m%d%H%M%S)"
    cp "$WSL_CONFIG_PATH" "$BACKUP_PATH"
    echo "📦 Создана резервная копия: $BACKUP_PATH"
fi

# Создаем новую конфигурацию
cat > "$WSL_CONFIG_PATH" << 'EOF'
[wsl2]
# Лимиты памяти и CPU
memory=10GB
processors=4
swap=2GB
swapfile=%USERPROFILE%\swap.vhdx

# Сетевые настройки
localhostForwarding=true

# ОПТИМИЗАЦИИ ПРОИЗВОДИТЕЛЬНОСТИ
vmIdleTimeout=3600000  # Сохранять память при простое (1 час)

# Оптимизации производительности
[experimental]
autoMemoryReclaim=dropcache
sparseVhd=true
```
```bash
chmod +x setup-wsl-config.sh
./setup-wsl-config.sh
```

---

## Установка Docker и Docker Compose

### Обновление пакетов и установка Docker

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

### Добавление пользователя в группу docker

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Установка Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Проверка установки

```bash
docker --version
docker-compose --version
```

---

## Подготовка окружения

### Создание рабочей директории

```bash
cd ~
mkdir gitlab-setup
cd gitlab-setup
```

### Настройка hosts файла

Создайте файл `setup-hosts.sh`:

```bash
#!/bin/bash

echo "=== НАСТРОЙКА HOSTS ФАЙЛА ==="
echo "Добавляем записи аналогично Vagrantfile..."

HOSTS_FILE="/etc/hosts"
TEMP_FILE="/tmp/hosts.tmp"

# Создаем резервную копию
sudo cp $HOSTS_FILE "${HOSTS_FILE}.backup.$(date +%Y%m%d%H%M%S)"

# Удаляем старые записи если есть
sudo grep -v -e "192.168.56.10" -e "192.168.56.20" -e "192.168.56.30" $HOSTS_FILE > $TEMP_FILE

# Добавляем новые записи (аналогично Vagrantfile)
echo "192.168.56.10    gitlab.localdomain" | sudo tee -a $TEMP_FILE
echo "192.168.56.20    sonarqube.localdomain" | sudo tee -a $TEMP_FILE
# echo "192.168.56.30    gitlab-runner.localdomain" | sudo tee -a $TEMP_FILE
# gitlab-runner.localdomain НЕ добавляем - он не веб-сервис!

# Заменяем оригинальный файл
sudo cp $TEMP_FILE $HOSTS_FILE
sudo rm $TEMP_FILE

echo "Hosts файл успешно настроен!"
echo "============================"
grep "192.168.56" $HOSTS_FILE
```

Выполните настройку hosts файла:
```bash
# Выполняем в директории gitlab-setup
cd ~/gitlab-setup
chmod +x setup-hosts.sh
sudo ./setup-hosts.sh
```

Для Windows (если используете WSL2)
Дополнительно настройте hosts файл Windows (в PowerShell от администратора):
```bash
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "192.168.56.10    gitlab.localdomain"
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "192.168.56.20    sonarqube.localdomain"
```

### Создание Docker сети

```bash
docker network create --driver=bridge --subnet=192.168.56.0/24 gitlab-network
```

### Проверка сети

```bash
docker network ls
docker network inspect gitlab-network
```

---

## Развертывание инфраструктуры

### Шаг 1: Экономная конфигурация Docker Compose

Создайте `docker-compose.yml`:

```yaml
version: '3.8'

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    hostname: gitlab.localdomain
    restart: unless-stopped
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.localdomain'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
        prometheus_monitoring['enable'] = false
        puma['worker_processes'] = 2
        puma['min_threads'] = 1
        puma['max_threads'] = 4
        sidekiq['max_concurrency'] = 5
        gitlab_rails['gitlab_default_can_create_group'] = 'false'
        nginx['worker_processes'] = 2
        nginx['worker_connections'] = 1024
        postgresql['shared_buffers'] = '512MB'  # ⬆️ УВЕЛИЧЕНО
        redis['maxmemory'] = '256mb'           # ⬆️ УВЕЛИЧЕНО
        redis['maxmemory_policy'] = 'allkeys-lru'
    ports:
      - "80:80"
      - "443:443"
      - "2224:22"
    volumes:
      - gitlab_config:/etc/gitlab
      - gitlab_logs:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab
    networks:
      gitlab-network:
        ipv4_address: 192.168.56.10
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 20
      start_period: 600s
    mem_limit: 4g        # ⬆️ УВЕЛИЧЕНО С 3GB
    mem_reservation: 3g  # ⬆️ УВЕЛИЧЕНО
    cpus: 2.0            # ⬆️ УВЕЛИЧЕНО

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab-runner-config:/etc/gitlab-runner
    networks:
      gitlab-network:
        ipv4_address: 192.168.56.30
    extra_hosts:
      - "gitlab.localdomain:192.168.56.10"
      - "sonarqube.localdomain:192.168.56.20"
    mem_limit: 1g        # ⬆️ УВЕЛИЧЕНО
    cpus: 1.0            # ⬆️ УВЕЛИЧЕНО

  sonarqube:
    image: sonarqube:9.9.1-community
    container_name: sonarqube
    hostname: sonarqube.localdomain
    restart: unless-stopped
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
      # УВЕЛИЧЕННЫЕ ЛИМИТЫ
      SONAR_WEB_JAVAOPTS: "-Xmx1g -Xms512m -XX:MaxMetaspaceSize=512m"
      SONAR_CE_JAVAOPTS: "-Xmx1g -Xms512m -XX:MaxMetaspaceSize=512m"
      SONAR_SEARCH_JAVAOPTS: "-Xmx1g -Xms512m -XX:MaxMetaspaceSize=512m"
      SONAR_CLUSTER_ENABLED: "false"
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    networks:
      gitlab-network:
        ipv4_address: 192.168.56.20
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/api/system/status"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 180s
    mem_limit: 3g        # ⬆️ УВЕЛИЧЕНО С 2GB
    mem_reservation: 2g  # ⬆️ УВЕЛИЧЕНО
    cpus: 2.0            # ⬆️ УВЕЛИЧЕНО

volumes:
  gitlab_config:
  gitlab_logs:
  gitlab_data:
  gitlab-runner-config:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:

networks:
  gitlab-network:
    external: true
    name: gitlab-network
```

### Шаг 2: Запуск инфраструктуры

```bash
# Выполняем в директории gitlab-setup
cd ~/gitlab-setup

# Проверяем доступность памяти (особенно важно для WSL2)
free -h
docker system df

# Запуск всех сервисов
docker-compose up -d

# Проверка статуса
docker-compose ps

echo "⏳ Ожидайте полный запуск GitLab (10-15 минут)..."
```
При нормальной работе получаем примерно такой вывод:
```
[+] Running 24/24
 ✔ sonarqube Pulled                                               272.0s
   ✔ d4f38e2e0926 Pull complete                                    42.5s
   ✔ 0c474f64746d Pull complete                                    61.2s
   ✔ 0d1e9704d175 Pull complete                                    61.2s
   ✔ 2326993b9fe6 Pull complete                                    61.2s
   ✔ eb91fa841f89 Pull complete                                   261.5s
   ✔ 9e0d86e3b9d8 Pull complete                                   268.4s
   ✔ 4f4fb700ef54 Pull complete                                   268.7s
 ✔ gitlab Pulled                                                  555.9s
   ✔ 4b3ffd8ccb52 Pull complete                                     7.6s
   ✔ f504df53e287 Pull complete                                     7.7s
   ✔ 2edd9432c4d6 Pull complete                                    11.5s
   ✔ 535c43257f7d Pull complete                                    11.6s
   ✔ db2d07e90c80 Pull complete                                    11.6s
   ✔ 8f1685a742d2 Pull complete                                    11.7s
   ✔ eff7b5192583 Pull complete                                    11.7s
   ✔ 95c6ee077a68 Pull complete                                    11.7s
   ✔ d3d71c5c4f6e Pull complete                                   531.1s
 ✔ gitlab-runner Pulled                                           148.9s
   ✔ de66fc90c55d Pull complete                                    21.5s
   ✔ 41d67aeda2be Pull complete                                    36.1s
   ✔ 4d48ea5044a3 Pull complete                                    36.1s
   ✔ 5a0e5eb3034e Pull complete                                   144.1s
   ✔ 734a07a1271e Pull complete                                   145.5s
[+] Running 10/10
 ✔ Volume gitlab-setup_gitlab_logs           Created                0.0s
 ✔ Volume gitlab-setup_gitlab_data           Created                0.1s
 ✔ Volume gitlab-setup_gitlab-runner-config  Created                0.1s
 ✔ Volume gitlab-setup_sonarqube_data        Created                0.1s
 ✔ Volume gitlab-setup_sonarqube_extensions  Created                0.0s
 ✔ Volume gitlab-setup_sonarqube_logs        Created                0.1s
 ✔ Volume gitlab-setup_gitlab_config         Created                0.1s
 ✔ Container gitlab                          Started                9.9s
 ✔ Container gitlab-runner                   Started                9.5s
 ✔ Container sonarqube                       Started                9.8s
 ```

### Шаг 3: Проверка сетевой конфигурации

Создайте файл `check-network.sh`:

```bash
#!/bin/bash

echo "=== ПРОВЕРКА СЕТЕВОЙ КОНФИГУРАЦИИ ==="

echo "1. Проверка hosts файла:"
grep "192.168.56" /etc/hosts || echo "Записи не найдены в hosts файле"

echo ""
echo "2. Проверка разрешения имен:"
echo "gitlab.localdomain -> $(dig +short gitlab.localdomain 2>/dev/null || nslookup gitlab.localdomain 2>/dev/null | grep Address | tail -1 || echo 'проверка не удалась')"
echo "sonarqube.localdomain -> $(dig +short sonarqube.localdomain 2>/dev/null || nslookup sonarqube.localdomain 2>/dev/null | grep Address | tail -1 || echo 'проверка не удалась')"

echo ""
echo "3. Проверка сети Docker:"
docker network inspect gitlab-network --format '{{range .Containers}}{{.Name}} - {{.IPv4Address}}{{"\n"}}{{end}}'

echo ""
echo "4. Проверка связи между контейнерами:"
echo "Из Runner в GitLab:"
docker-compose exec gitlab-runner ping -c 2 gitlab.localdomain

echo "Из Runner в SonarQube:"
docker-compose exec gitlab-runner ping -c 2 sonarqube.localdomain

echo ""
echo "5. Проверка доступности сервисов:"
echo "GitLab: http://gitlab.localdomain"
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://gitlab.localdomain || echo "Недоступен"

echo "SonarQube: http://sonarqube.localdomain:9000"
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://sonarqube.localdomain:9000 || echo "Недоступен"
```

Выполните проверку:
```bash
# Выполняем в директории gitlab-setup
cd ~/gitlab-setup
chmod +x check-network.sh
./check-network.sh
```

---

## Настройка компонентов

### Шаг 1: Мониторинг запуска GitLab

Создайте файл `wait-gitlab-working.sh`:

```bash
#!/bin/bash

echo "⏳ Проверяем статус GitLab..."
cd ~/gitlab-setup

# Функция для получения статуса разными способами
get_gitlab_status() {
    # Способ 1: через docker-compose ps --format json
    local status1=$(docker-compose ps gitlab --format json 2>/dev/null | jq -r '.[].Status' 2>/dev/null)
    
    # Способ 2: через парсинг вывода docker-compose ps
    local status2=$(docker-compose ps gitlab 2>/dev/null | grep gitlab | awk '{print $4}')
    
    # Способ 3: через docker inspect
    local status3=$(docker inspect gitlab --format='{{.State.Health.Status}}' 2>/dev/null)
    
    # Возвращаем первый непустой статус
    if [ -n "$status1" ]; then
        echo "$status1"
    elif [ -n "$status2" ]; then
        echo "$status2"
    elif [ -n "$status3" ]; then
        echo "$status3"
    else
        echo "unknown"
    fi
}

# Функция проверки доступности GitLab
check_gitlab_accessible() {
    if curl -s -f http://gitlab.localdomain > /dev/null 2>&1; then
        return 0
    else
        return 1
    fi
}

echo "🔍 Проверяем текущее состояние..."

# Проверяем доступность сразу
if check_gitlab_accessible; then
    echo "✅ GitLab уже доступен в браузере!"
else
    echo "❌ GitLab недоступен по http://gitlab.localdomain"
fi

# Проверяем статус контейнера
STATUS=$(get_gitlab_status)
echo "📊 Статус контейнера: $STATUS"

# Если GitLab доступен в браузере, сразу получаем пароль
if check_gitlab_accessible; then
    echo ""
    echo "🎉 GitLab готов! Получаем пароль..."
    
    # Даем дополнительное время для стабилизации
    sleep 10
    
    # Получаем пароль
    PASSWORD=$(docker-compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password 2>/dev/null | cut -d: -f2- | sed 's/^ *//;s/ *$//')
    
    if [ -n "$PASSWORD" ]; then
        echo "========================================"
        echo "🔐 ROOT PASSWORD: $PASSWORD"
        echo "========================================"
        echo ""
        echo "🌐 GitLab: http://gitlab.localdomain"
        echo "👤 Логин: root"
        echo "🔑 Пароль: (см. выше)"
    else
        echo "❌ Пароль не найден в файле"
        echo "Попробуйте получить вручную:"
        echo "docker-compose exec gitlab cat /etc/gitlab/initial_root_password"
    fi
    exit 0
fi

# Если GitLab еще не доступен, ждем
echo ""
echo "⏳ Ожидаем полную готовность GitLab..."

for i in {1..30}; do
    STATUS=$(get_gitlab_status)
    echo "Проверка $i/30: Статус=$STATUS"
    
    if check_gitlab_accessible; then
        echo "✅ GitLab стал доступен!"
        
        # Даем время для полной инициализации
        sleep 30
        
        # Получаем пароль
        PASSWORD=$(docker-compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password 2>/dev/null | cut -d: -f2- | sed 's/^ *//;s/ *$//')
        
        if [ -n "$PASSWORD" ]; then
            echo "========================================"
            echo "🔐 ROOT PASSWORD: $PASSWORD"
            echo "========================================"
        else
            echo "⚠️ Пароль не найден автоматически"
        fi
        
        echo "🌐 GitLab: http://gitlab.localdomain"
        exit 0
    fi
    
    sleep 10
done

echo "❌ GitLab не стал доступен за 5 минут"
echo "Но попробуйте открыть: http://gitlab.localdomain"
```

Выполните скрипт ожидания:
```bash
cd ~/gitlab-setup
chmod +x wait-gitlab-working.sh
./wait-gitlab-working.sh
```

### Шаг 2: Настройка GitLab Runner

```bash
# Выполняем в директории gitlab-setup
cd ~/gitlab-setup

# Регистрация общего Runner
docker-compose exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://gitlab.localdomain/" \
  --registration-token "$(docker-compose exec gitlab gitlab-rails runner -e production "puts Gitlab::CurrentSettings.current_application_settings.runners_registration_token")" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "docker-runner" \
  --tag-list "docker,linux" \
  --run-untagged="true" \
  --locked="false" \
  --docker-privileged="true" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/cache"

echo "✅ GitLab Runner зарегистрирован!"
```

---

## Создание и настройка веб-проекта

### Шаг 1: Создание проекта в GitLab

1. **Откройте**: http://gitlab.localdomain
2. **Войдите** как root (пароль из скрипта ожидания)
3. **Создайте проект**:
   - Нажмите **"New project"**
   - Выберите **"Create blank project"**
   - **Project name**: `my-web-app`
   - **Visibility**: `Private`
   - **Initialize with README**: ❌ НЕ отмечать
   - Нажмите **"Create project"**

### Шаг 2: Создание структуры файлов

```bash
# Создаем директорию для проекта
mkdir -p ~/my-web-app
cd ~/my-web-app

# Создаем структуру проекта
mkdir css
```

**Файл: `index.html`**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Web App</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <h1>Welcome to My Web App!</h1>
    <p>This project uses GitLab CI/CD with SonarQube analysis and LESS compilation.</p>
</body>
</html>
```

**Файл: `style.less`**
```less
@primary-color: #3498db;
@background-color: #f4f4f4;

body {
    background-color: @background-color;
    font-family: Arial, sans-serif;
    margin: 0;
    padding: 20px;
}

h1 {
    color: @primary-color;
    text-align: center;

    &:hover {
        text-decoration: underline;
    }
}
```

**Файл: `sonar-project.properties`**
```properties
sonar.projectKey=my-web-app
sonar.projectName=My Web App

sonar.sources=.
sonar.exclusions=node_modules/**, css/**

# Для LESS анализа через плагин SonarCSS
sonar.css.less.parser.less=style.less
```

---

## Интеграция CI/CD с SonarQube

### Шаг 1: Настройка SonarQube

```bash
echo "🔧 Настройка SonarQube:"
echo "1. Откройте: http://sonarqube.localdomain:9000"
echo "2. Войдите: admin/admin"
echo "3. Смените пароль на 'netology'"
echo "4. Создайте токен для GitLab:"
echo "   • My Account → Security → Generate Token"
echo "   • Название: gitlab-ci-token"
echo "   • Скопируйте значение токена"
echo ""
echo "⚠️  ЗАПИШИТЕ ТОКЕН: ________________"
```

### Установка плагинов SonarQube

1. **Установка необходимых плагинов:**
   - В верхнем меню перейдите в **Administration** -> **Marketplace**.
   - В поиске найдите и установите следующие плагины:
     - **SonarCSS** (для анализа CSS и LESS)
     - **SonarHTML** (для анализа HTML)
     - **SonarPHP** (для анализа PHP)
   - *Примечание: Плагины для JavaScript (SonarJS) и для общего анализа кода (SonarWay) обычно предустановлены.*
   - После установки плагинов потребуется перезагрузка SonarQube (система предложит это сделать).

### Шаг 2: Настройка переменных CI/CD в GitLab

1. В проекте перейдите: **Settings → CI/CD → Variables**
2. Нажмите **"Add variable"**
3. Добавьте переменные:
   - **SONAR_TOKEN**: [ваш токен из SonarQube]
   - **SONAR_HOST_URL**: `http://sonarqube.localdomain:9000`

### Шаг 3: Создание .gitlab-ci.yml

**Файл: `.gitlab-ci.yml`**
```yaml
stages:
  - test
  - build

sonarqube-check:
  stage: test
  image: 
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    paths:
      - .sonar/cache
  script:
    - sonar-scanner
  allow_failure: false
  only:
    - main

compile-less:
  stage: build
  image: node:latest
  before_script:
    - npm install -g less
  script:
    - lessc style.less css/style.css
    - cat css/style.css
  artifacts:
    paths:
      - css/
  only:
    - main
```

### Шаг 4: Инициализация Git репозитория

```bash
cd ~/my-web-app

# Инициализируем Git
git init
git config --local user.name "root"
git config --local user.email "root@gitlab.localdomain"

# Добавляем файлы и делаем первый коммит
git add .
git commit -m "Initial commit with CI/CD and SonarQube integration"

# Добавляем удаленный репозиторий
git remote add origin http://gitlab.localdomain/root/my-web-app.git

# Пушим в GitLab
git push -u origin main

echo "✅ Проект загружен в GitLab!"
```

---

## Управление и мониторинг

### Основные команды управления

```bash
cd ~/gitlab-setup

# Просмотр статуса
docker-compose ps

# Логи сервисов
docker-compose logs gitlab
docker-compose logs sonarqube --follow
docker-compose logs gitlab-runner

# Остановка/запуск
docker-compose stop
docker-compose start
docker-compose restart

# Полная остановка с сохранением данных
docker-compose down

# Полная остановка с удалением данных (⚠️ осторожно!)
docker-compose down -v
```

### Мониторинг ресурсов

```bash
# Потребление ресурсов
docker stats

# Размер volumes
docker system df

# Очистка неиспользуемых ресурсов
docker system prune -f
```

### Скрипт безопасной очистки проекта

Создайте файл `reset-project.sh`:

```bash
#!/bin/bash

echo "=== 🗑️ БЕЗОПАСНАЯ ОЧИСТКА ПРОЕКТA GITLAB-SONARQUBE ==="

cd ~/gitlab-setup

if [ -f "docker-compose.yml" ]; then
    echo "Останавливаем и удаляем контейнеры проекта..."
    docker-compose down -v
    
    echo "Удаляем volumes проекта..."
    docker volume rm gitlab-setup_gitlab_config \
                     gitlab-setup_gitlab_logs \
                     gitlab-setup_gitlab_data \
                     gitlab-setup_gitlab-runner-config \
                     gitlab-setup_sonarqube_data \
                     gitlab-setup_sonarqube_extensions \
                     gitlab-setup_sonarqube_logs 2>/dev/null || true
    
    echo "Удаляем сеть проекта..."
    docker network rm gitlab-network 2>/dev/null || true
    
    echo "✅ Очистка проекта завершена!"
else
    echo "❌ docker-compose.yml не найден в директории проекта"
fi
```

### Скрипт полной очистки Docker

Создайте файл `full_reset_docker.sh`:

```bash
#!/bin/bash

# ВЫПОЛНЯЙТЕ ЭТИ КОМАНДЫ ТОЛЬКО ЕСЛИ ВЫ ПОНИМАЕТЕ ПОСЛЕДСТВИЯ!
# ЭТО УДАЛИТ ВСЕ ВАШИ DOCKER КОНТЕЙНЕРЫ, VOLUMES И СЕТИ!

echo "=== 🔥 ОПАСНО: ПОЛНАЯ ОЧИСТКА DOCKER ==="
echo "ЭТО УДАЛИТ:"
echo "✅ Все запущенные контейнеры"
echo "✅ Все Docker volumes (включая данные GitLab, БД и т.д.)"
echo "✅ Все Docker сети"
echo "✅ Все неиспользуемые образы"
echo ""
echo "ВСЕ ДАННЫЕ БУДУТ БЕЗВОЗВРАТНО УТЕРЯНЫ!"
echo "========================================"

read -p "Вы уверены, что хотите продолжить? (yes/NO): " confirmation

if [ "$confirmation" != "yes" ]; then
    echo "Очистка отменена."
    exit 1
fi

# Выполняем в любой директории - это глобальная очистка Docker
echo "Начинаем полную очистку Docker..."

# 1. Останавливаем все работающие контейнеры
echo "1. Останавливаем все контейнеры..."
docker stop $(docker ps -aq) 2>/dev/null || echo "Нет контейнеров для остановки"

# 2. Удаляем все контейнеры
echo "2. Удаляем все контейнеры..."
docker rm $(docker ps -aq) 2>/dev/null || echo "Нет контейнеров для удаления"

# 3. Удаляем все volumes
echo "3. Удаляем все volumes..."
docker volume rm $(docker volume ls -q) 2>/dev/null || echo "Нет volumes для удаления"

# 4. Удаляем все сети (кроме предустановленных)
echo "4. Удаляем все пользовательские сети..."
docker network rm $(docker network ls -q --filter type=custom) 2>/dev/null || echo "Нет сетей для удаления"

# 5. Очищаем неиспользуемые образы
echo "5. Удаляем неиспользуемые образы..."
docker image prune -a -f

# 6. Полная система очистка
echo "6. Полная очистка системы Docker..."
docker system prune -a -f --volumes

echo ""
echo "✅ Очистка завершена! Docker полностью очищен."
echo "⚠️  Все данные безвозвратно удалены!"
```

---

## Устранение неполадок

### Проверка логов

```bash
# Логи GitLab
docker logs gitlab

# Логи GitLab Runner
docker logs gitlab-runner

# Логи SonarQube
docker logs sonarqube
```

### Проверка сетевой связности

```bash
# Проверить связность между контейнерами
docker exec gitlab ping -c 3 192.168.56.20
docker exec gitlab-runner ping -c 3 192.168.56.10
docker exec sonarqube ping -c 3 192.168.56.10
```

### Перезапуск сервисов

```bash
docker-compose restart
```

### Проверка состояния сети

```bash
docker network inspect gitlab-network
```

### Проверка работоспособности

```bash
cat > check-system.sh << 'EOF'
#!/bin/bash

echo "=== ✅ ПРОВЕРКА СИСТЕМЫ ==="
cd ~/gitlab-setup

echo ""
echo "📊 СТАТУС СЕРВИСОВ:"
docker-compose ps

echo ""
echo "🌐 ДОСТУП К СЕРВИСАМ:"
echo "----------------------------------------"
echo "🔧 GitLab:"
curl -s http://gitlab.localdomain > /dev/null && echo "✅ Доступен" || echo "❌ Недоступен"

echo "📊 SonarQube:"
curl -s http://sonarqube.localdomain:9000 > /dev/null && echo "✅ Доступен" || echo "❌ Недоступен"

echo ""
echo "🔗 СЕТЕВАЯ СВЯЗНОСТЬ:"
docker-compose exec gitlab-runner ping -c 1 gitlab.localdomain > /dev/null && \
  echo "✅ GitLab доступен из Runner" || echo "❌ Проблема с GitLab"

docker-compose exec gitlab-runner ping -c 1 sonarqube.localdomain > /dev/null && \
  echo "✅ SonarQube доступен из Runner" || echo "❌ Проблема с SonarQube"

echo ""
echo "⚙️ GITLAB RUNNER:"
docker-compose exec gitlab-runner gitlab-runner list

echo ""
echo "🎯 ПРОВЕРКА ПРОЕКТА:"
echo "----------------------------------------"
echo "🌐 GitLab проект: http://gitlab.localdomain/root/my-web-app"
echo "📊 Pipelines: http://gitlab.localdomain/root/my-web-app/-/pipelines"
echo "🔍 SonarQube: http://sonarqube.localdomain:9000"

echo ""
echo "✅ СИСТЕМА ГОТОВА К РАБОТЕ!"
EOF

chmod +x check-system.sh
./check-system.sh
```

---

## Заключение

Вы успешно развернули полнофункциональную среду CI/CD для веб-разработки с правильно настроенной сетевой архитектурой, включающую:

- **GitLab** (192.168.56.10) для управления исходным кодом
- **GitLab Runner** (192.168.56.30) для выполнения пайплайнов
- **SonarQube** (192.168.56.20) для анализа качества кода веб-технологий
- **Автоматизированный пайплайн** для проверки LESS-синтаксиса и компиляции в CSS

Эта система обеспечивает автоматическую проверку качества кода и сборку ваших веб-проектов при каждом коммите в репозиторий.

---

## Дальнейшие шаги

- Настройка SSL-сертификатов для GitLab
- Настройка резервного копирования
- Добавление этапа деплоя в пайплайн
- Интеграция с другими инструментами качества кода
- Настройка уведомлений о результатах пайплайнов
- Масштабирование Runner для обработки нескольких проектов

---

*Версия V6 - Обновлено с полным набором скриптов управления, WSL2 оптимизацией и улучшенной структурой проекта*
