На основе анализа предоставленных файлов, я вижу эволюцию руководства по развертыванию GitLab от V2 до V6. Давайте создам улучшенную версию V7, которая объединит лучшие практики из всех предыдущих версий и добавит новые улучшения.

# 🚀 Практическое руководство: Развертывание GitLab с GitLab Runner и SonarQube в Docker для веб-разработки

## 📋 Содержание

1. [Введение](#введение)
2. [Архитектура решения](#архитектура-решения)
3. [Предварительные требования](#предварительные-требования)
4. [Быстрый старт](#быстрый-старт)
5. [Подготовка окружения](#подготовка-окружения)
6. [Развертывание инфраструктуры](#развертывание-инфраструктуры)
7. [Настройка компонентов](#настройка-компонентов)
8. [Создание и настройка веб-проекта](#создание-и-настройка-веб-проекта)
9. [Интеграция CI/CD с SonarQube](#интеграция-cicd-с-sonarqube)
10. [Управление и мониторинг](#управление-и-мониторинг)
11. [Устранение неполадок](#устранение-неполадок)
12. [Дополнительные настройки](#дополнительные-настройки)

---

## Введение

Это руководство предоставляет пошаговые инструкции по развертыванию собственного экземпляра GitLab, настроенного для веб-разработки (CSS, HTML, PHP, JavaScript), вместе с GitLab Runner и SonarQube в среде Docker. Мы настроим автоматизированный пайплайн CI/CD, который будет проверять код, включая анализ синтаксиса LESS, с помощью SonarQube и развертывать его.

**Ключевые особенности V7:**
- ✅ Оптимизированные настройки ресурсов
- ✅ Полная поддержка WSL2 и Linux
- ✅ Улучшенные скрипты автоматизации
- ✅ Расширенная диагностика системы
- ✅ Подробная документация по устранению неполадок
- ✅ Готовые примеры для веб-разработки

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

### Распределение ресурсов

### После оптимизации (для 4 ядер / 16GB):
- **GitLab**: 3GB RAM, 1.5 CPU ⬇️
- **SonarQube**: 2GB RAM, 1.0 CPU ⬇️
- **Runner**: 512MB RAM, 0.5 CPU ⬇️
- **Всего**: 5.5GB RAM, 3.0 CPU ⬇️

### WSL2 настройки:
- **Память**: 12GB (вместо 10GB)
- **Ядра**: 3 (вместо 4)
- **Swap**: 4GB (вместо 2GB)

Эти настройки обеспечат стабильную работу всей системы без перегрузки 4-ядерной конфигурации! 🚀
---

## Предварительные требования

### Минимальные системные требования

**Для всех систем:**
- Docker 20.10+ и Docker Compose 2.0+
- 10 ГБ оперативной памяти (рекомендуется 12 ГБ)
- Более 20 ГБ свободного места на диске
- Поддержка виртуализации

**Для WSL2:**
- Windows 10/11 с WSL2
- Ubuntu 20.04+ из Microsoft Store
- Docker Desktop с WSL2 integration

### Проверка системы

```bash
# Проверка памяти
free -h

# Проверка диска
df -h

# Проверка Docker
docker --version
docker-compose --version

# Проверка виртуализации (для Linux)
grep -E --color '(vmx|svm)' /proc/cpuinfo
```

---

## Быстрый старт

### Автоматический скрипт развертывания



---

## Подготовка окружения

## Оптимизированные настройки WSL2 для 4 ядер / 16GB RAM

```bash
#!/bin/bash
# optimize-wsl2-4core.sh

echo "⚡ ОПТИМИЗАЦИЯ WSL2 ДЛЯ СИСТЕМЫ С 4 ЯДРАМИ И 16GB RAM"

# Определяем доступную память и ядра
TOTAL_MEMORY_GB=15.8
TOTAL_CORES=4
TOTAL_THREADS=4

echo "📊 Характеристики системы:"
echo "   • Ядра: $TOTAL_CORES"
echo "   • Потоки: $TOTAL_THREADS" 
echo "   • Память: ${TOTAL_MEMORY_GB}GB"
echo ""

# Расчет оптимальных значений
WSL_MEMORY=$(( ${TOTAL_MEMORY_GB%.*} - 4 ))  # Оставляем 4GB для Windows
WSL_PROCESSORS=$(( TOTAL_CORES - 1 ))         # Оставляем 1 ядро для Windows
WSL_SWAP=$(( WSL_MEMORY / 2 ))                # Swap = 50% от памяти

# Ограничения для безопасности
if [ $WSL_MEMORY -gt 12 ]; then
    WSL_MEMORY=12
fi
if [ $WSL_PROCESSORS -gt 3 ]; then
    WSL_PROCESSORS=3
fi
if [ $WSL_SWAP -gt 4 ]; then
    WSL_SWAP=4
fi

echo "🎯 Рекомендуемые настройки WSL2:"
echo "   • Память: ${WSL_MEMORY}GB"
echo "   • Ядра: ${WSL_PROCESSORS}"
echo "   • Swap: ${WSL_SWAP}GB"
echo ""

# Создание конфигурации WSL2
cat > /mnt/c/Users/$USER/.wslconfig << EOF
[wsl2]
# Ресурсы для WSL2 (оставляем ресурсы для Windows)
memory=${WSL_MEMORY}GB
processors=${WSL_PROCESSORS}
swap=${WSL_SWAP}GB
swapfile=%USERPROFILE%\\\\wsl-swap.vhdx

# Сетевые настройки
localhostForwarding=true
dnsTunneling=true
firewall=true
autoProxy=true

# Файловая система
pageReporting=true
kernelCommandLine=intel_pstate=disable

# Производительность
vmIdleTimeout=3600000

# Экспериментальные функции
[experimental]
autoMemoryReclaim=dropcache
sparseVhd=true
dnsTunneling=true
hostAddressLoopback=true
EOF

echo "✅ Конфигурация WSL2 применена:"
cat /mnt/c/Users/$USER/.wslconfig

echo ""
echo "🔁 Для применения настроек выполните:"
echo "   wsl --shutdown"
echo "   wsl"
echo ""
echo "💡 После перезапуска WSL проверьте настройки:"
echo "   cat /proc/meminfo | grep -i memtotal"
echo "   nproc"
```
## Скрипт проверки и оптимизации системы
```bash
#!/bin/bash
# system-optimizer.sh

echo "🔧 ОПТИМИЗАЦИЯ СИСТЕМЫ ДЛЯ 4 ЯДЕР / 16GB RAM"

# Проверка текущих ресурсов
echo "📊 ТЕКУЩИЕ РЕСУРСЫ WSL2:"
echo "   • Ядра: $(nproc)"
echo "   • Память: $(grep MemTotal /proc/meminfo | awk '{print int($2/1024/1024)" GB"}')"
echo "   • Swap: $(grep SwapTotal /proc/meminfo | awk '{print int($2/1024/1024)" GB"}')"

# Проверка использования Docker
echo ""
echo "🐳 ИСПОЛЬЗОВАНИЕ DOCKER:"
docker system df

# Расчет оптимальных значений
echo ""
echo "🎯 РАСЧЕТ ОПТИМАЛЬНЫХ НАСТРОЕК:"

TOTAL_MEMORY_GB=15.8
TOTAL_CORES=4

# Безопасные лимиты (оставляем ресурсы для Windows)
WSL_MEMORY=10
WSL_PROCESSORS=3
WSL_SWAP=4

GITLAB_MEMORY=3
GITLAB_CPUS=1.5

SONARQUBE_MEMORY=2
SONARQUBE_CPUS=1.0

RUNNER_MEMORY=0.5
RUNNER_CPUS=0.5

echo "   WSL2 Конфигурация:"
echo "   • Память: ${WSL_MEMORY}GB"
echo "   • Ядра: ${WSL_PROCESSORS}"
echo "   • Swap: ${WSL_SWAP}GB"
echo ""
echo "   Docker Сервисы:"
echo "   • GitLab: ${GITLAB_MEMORY}GB RAM, ${GITLAB_CPUS} CPU"
echo "   • SonarQube: ${SONARQUBE_MEMORY}GB RAM, ${SONARQUBE_CPUS} CPU"
echo "   • Runner: ${RUNNER_MEMORY}GB RAM, ${RUNNER_CPUS} CPU"
echo ""
echo "   💡 ОБЩЕЕ ИСПОЛЬЗОВАНИЕ:"
TOTAL_USED=$((GITLAB_MEMORY + SONARQUBE_MEMORY + RUNNER_MEMORY))
echo "   • Память: ${TOTAL_USED}GB / ${WSL_MEMORY}GB ($((TOTAL_USED * 100 / WSL_MEMORY))%)"
echo "   • CPU: $((GITLAB_CPUS + SONARQUBE_CPUS + RUNNER_CPUS)) / ${WSL_PROCESSORS} ядер"

# Проверка текущей конфигурации WSL
echo ""
echo "🔍 ПРОВЕРКА ТЕКУЩЕЙ КОНФИГУРАЦИИ WSL:"
if [ -f "/mnt/c/Users/$USER/.wslconfig" ]; then
    echo "✅ Файл .wslconfig найден:"
    grep -E "(memory|processors|swap)" /mnt/c/Users/$USER/.wslconfig
else
    echo "❌ Файл .wslconfig не найден"
fi

# Предложение оптимизации
echo ""
echo "🚀 РЕКОМЕНДУЕМЫЕ ДЕЙСТВИЯ:"
echo "   1. Запустите: ./optimize-wsl2-4core.sh"
echo "   2. Перезапустите WSL: wsl --shutdown && wsl"
echo "   3. Обновите docker-compose.yml с новыми настройками"
echo "   4. Перезапустите сервисы: docker-compose up -d"
```

### Создание рабочей директории

```bash
mkdir -p ~/gitlab-setup
cd ~/gitlab-setup
```

### Настройка hosts файла

```bash
#!/bin/bash
# setup-hosts.sh

echo "=== НАСТРОЙКА HOSTS ФАЙЛА ==="

HOSTS_FILE="/etc/hosts"
BACKUP_FILE="${HOSTS_FILE}.backup.$(date +%Y%m%d%H%M%S)"

# Резервное копирование
sudo cp $HOSTS_FILE $BACKUP_FILE
echo "📦 Создана резервная копия: $BACKUP_FILE"

# Удаление старых записей
sudo sed -i '/192.168.56/d' $HOSTS_FILE

# Добавление новых записей
sudo tee -a $HOSTS_FILE << EOL

# GitLab Infrastructure
192.168.56.10    gitlab.localdomain
192.168.56.20    sonarqube.localdomain
EOL

echo "✅ Hosts файл настроен"
echo "📋 Проверка записей:"
grep "192.168.56" $HOSTS_FILE
```

---

## Развертывание инфраструктуры

### Обновленный V7 скрипт быстрого развертывания quick-deploy-4core.sh

```bash
#!/bin/bash
# quick-deploy-4core.sh

echo "🚀 АВТОМАТИЧЕСКОЕ РАЗВЕРТЫВАНИЕ ДЛЯ 4-ЯДЕРНОЙ СИСТЕМЫ"

set -e

# Проверка ресурсов
echo "📊 Проверка системных ресурсов..."
CORES=$(nproc)
MEMORY_GB=$(grep MemTotal /proc/meminfo | awk '{print int($2/1024/1024)}')

echo "   • Доступно ядер: $CORES"
echo "   • Доступно памяти: ${MEMORY_GB}GB"

if [ $MEMORY_GB -lt 10 ]; then
    echo "⚠️  Внимание: Мало памяти. Рекомендуется минимум 10GB для комфортной работы."
fi

# Создание рабочей директории
mkdir -p ~/gitlab-setup
cd ~/gitlab-setup

echo "📥 Создание оптимизированной конфигурации..."

# Создание оптимизированного docker-compose.yml
cat > docker-compose.yml << 'EOF'
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
        # 🔧 ОПТИМИЗАЦИИ ДЛЯ 4-ЯДЕРНОЙ СИСТЕМЫ
        prometheus_monitoring['enable'] = false
        puma['worker_processes'] = 2
        puma['min_threads'] = 1
        puma['max_threads'] = 2
        sidekiq['max_concurrency'] = 3
        nginx['worker_processes'] = 1
        nginx['worker_connections'] = 512
        postgresql['shared_buffers'] = '256MB'
        redis['maxmemory'] = '128mb'
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
    mem_limit: 3g
    mem_reservation: 2g
    cpus: 1.5

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: unless-stopped
    depends_on:
      - gitlab
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab-runner-config:/etc/gitlab-runner
    networks:
      gitlab-network:
        ipv4_address: 192.168.56.30
    extra_hosts:
      - "gitlab.localdomain:192.168.56.10"
      - "sonarqube.localdomain:192.168.56.20"
    mem_limit: 512m
    cpus: 0.5

  sonarqube:
    image: sonarqube:9.9.1-community
    container_name: sonarqube
    hostname: sonarqube.localdomain
    restart: unless-stopped
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
      # 🔧 ОПТИМИЗАЦИИ ДЛЯ 4-ЯДЕРНОЙ СИСТЕМЫ
      SONAR_WEB_JAVAOPTS: "-Xmx512m -Xms256m -XX:MaxMetaspaceSize=128m"
      SONAR_CE_JAVAOPTS: "-Xmx512m -Xms256m -XX:MaxMetaspaceSize=128m"
      SONAR_SEARCH_JAVAOPTS: "-Xmx512m -Xms256m -XX:MaxMetaspaceSize=128m"
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
    mem_limit: 2g
    mem_reservation: 1g
    cpus: 1.0

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
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.56.0/24
          gateway: 192.168.56.1
EOF

# Настройка hosts файла
echo "📝 Настройка hosts файла..."
sudo bash -c 'cat >> /etc/hosts << EOL
192.168.56.10    gitlab.localdomain
192.168.56.20    sonarqube.localdomain
EOL'

# Запуск инфраструктуры
echo "🐳 Запуск Docker контейнеров..."
docker-compose up -d

echo "⏳ Ожидание запуска сервисов..."
sleep 30

# Проверка статуса
echo "🔍 Проверка статуса сервисов..."
docker-compose ps

echo ""
echo "🎉 АВТОМАТИЧЕСКОЕ РАЗВЕРТЫВАНИЕ ЗАВЕРШЕНО!"
echo ""
echo "📊 ВЫДЕЛЕННЫЕ РЕСУРСЫ:"
echo "   • GitLab: 3GB RAM, 1.5 CPU"
echo "   • SonarQube: 2GB RAM, 1.0 CPU" 
echo "   • Runner: 512MB RAM, 0.5 CPU"
echo "   • Всего: ~5.5GB RAM, 3.0 CPU"
echo ""
echo "💡 ДАЛЬНЕЙШИЕ ДЕЙСТВИЯ:"
echo "   1. Оптимизируйте WSL2: ./optimize-wsl2-4core.sh"
echo "   2. Мониторьте запуск: ./wait-gitlab-working.sh"
echo "   3. Проверьте систему: ./system-check.sh"
```


### Запуск инфраструктуры

```bash
# Запуск всех сервисов
docker-compose up -d

# Мониторинг запуска
docker-compose logs -f gitlab &

# Проверка статуса
docker-compose ps
```

---

## Настройка компонентов

### Улучшенный скрипт ожидания GitLab

```bash
#!/bin/bash
# wait-for-gitlab.sh

echo "🎯 УЛУЧШЕННЫЙ МОНИТОРИНГ ЗАПУСКА GITLAB"

check_gitlab_health() {
    local container_status=$(docker inspect gitlab --format='{{.State.Health.Status}}' 2>/dev/null)
    local http_status=$(curl -s -o /dev/null -w "%{http_code}" http://gitlab.localdomain 2>/dev/null)
    local logs_status=$(docker-compose logs gitlab 2>/dev/null | tail -5)
    
    echo "📊 Статус контейнера: $container_status"
    echo "🌐 HTTP статус: $http_status"
    
    if [ "$http_status" = "200" ]; then
        return 0
    else
        return 1
    fi
}

get_root_password() {
    local password=$(docker-compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password 2>/dev/null | cut -d: -f2- | sed 's/^ *//;s/ *$//')
    
    if [ -n "$password" ]; then
        echo "🔐 ROOT PASSWORD: $password"
        return 0
    else
        echo "⚠️ Пароль не найден в стандартном месте"
        return 1
    fi
}

echo "⏳ Начало мониторинга запуска GitLab..."
echo "📝 Это может занять 10-15 минут..."

for i in {1..60}; do
    echo ""
    echo "🔍 Проверка #$i ($(date))"
    
    if check_gitlab_health; then
        echo ""
        echo "🎉 GITLAB УСПЕШНО ЗАПУЩЕН!"
        echo ""
        
        # Даем время на финальную инициализацию
        sleep 30
        
        echo "🔒 Получение пароля root..."
        if get_root_password; then
            echo ""
            echo "🚀 СИСТЕМА ГОТОВА К РАБОТЕ!"
            echo "🌐 GitLab: http://gitlab.localdomain"
            echo "👤 Логин: root"
            echo "🔑 Пароль: (см. выше)"
        else
            echo "❌ Не удалось получить пароль автоматически"
            echo "💡 Попробуйте вручную: docker-compose exec gitlab cat /etc/gitlab/initial_root_password"
        fi
        
        exit 0
    fi
    
    echo "⏳ Ожидание... (10 секунд)"
    sleep 10
done

echo ""
echo "❌ ПРЕВЫШЕНО ВРЕМЯ ОЖИДАНИЯ"
echo "💡 GitLab все еще может запускаться. Проверьте вручную:"
echo "   docker-compose logs gitlab"
echo "   curl http://gitlab.localdomain"
exit 1
```

### Настройка GitLab Runner

```bash
#!/bin/bash
# setup-runner.sh

echo "🔧 НАСТРОЙКА GITLAB RUNNER"

# Ожидаем готовность GitLab
echo "⏳ Ожидание готовности GitLab..."
until curl -s http://gitlab.localdomain > /dev/null; do
    sleep 10
done

# Получение registration token
echo "📝 Получение registration token..."
REGISTRATION_TOKEN=$(docker-compose exec gitlab gitlab-rails runner -e production "puts Gitlab::CurrentSettings.current_application_settings.runners_registration_token")

if [ -z "$REGISTRATION_TOKEN" ]; then
    echo "❌ Не удалось получить registration token"
    echo "💡 Попробуйте получить вручную из интерфейса GitLab"
    exit 1
fi

echo "✅ Registration token получен"

# Регистрация Runner
echo "🚀 Регистрация GitLab Runner..."
docker-compose exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://gitlab.localdomain/" \
  --registration-token "$REGISTRATION_TOKEN" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "web-development-runner" \
  --tag-list "docker,linux,web,less" \
  --run-untagged="true" \
  --locked="false" \
  --docker-privileged="true" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/cache" \
  --docker-volumes "/builds:/builds"

echo "✅ GitLab Runner успешно зарегистрирован"

# Проверка статуса
echo "📊 Проверка статуса Runner..."
docker-compose exec gitlab-runner gitlab-runner list
```

---

## Создание и настройка веб-проекта

### Структура веб-проекта

```bash
# Создание структуры проекта
mkdir -p ~/web-dev-project/{css,js,src/less,dist}
cd ~/web-dev-project
```

### Примеры файлов для веб-разработки

**package.json**
```json
{
  "name": "web-dev-project",
  "version": "1.0.0",
  "description": "Web development project with GitLab CI/CD",
  "scripts": {
    "build:less": "lessc src/less/main.less dist/css/main.css",
    "build:js": "uglify-js js/*.js -o dist/js/app.min.js",
    "build": "npm run build:less && npm run build:js",
    "test": "echo 'Running tests...'"
  },
  "devDependencies": {
    "less": "^4.1.3",
    "uglify-js": "^3.17.4"
  }
}
```

**src/less/main.less**
```less
// Variables
@primary-color: #3498db;
@secondary-color: #2ecc71;
@text-color: #2c3e50;
@background-color: #ecf0f1;

// Mixins
.border-radius(@radius: 5px) {
  border-radius: @radius;
  -webkit-border-radius: @radius;
  -moz-border-radius: @radius;
}

.box-shadow(@x: 0, @y: 2px, @blur: 5px, @color: rgba(0,0,0,0.1)) {
  box-shadow: @x @y @blur @color;
  -webkit-box-shadow: @x @y @blur @color;
  -moz-box-shadow: @x @y @blur @color;
}

// Base styles
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Arial', sans-serif;
  line-height: 1.6;
  color: @text-color;
  background-color: @background-color;
  padding: 20px;
}

.header {
  background: linear-gradient(135deg, @primary-color, @secondary-color);
  color: white;
  padding: 2rem;
  text-align: center;
  .border-radius(10px);
  .box-shadow(0, 4px, 15px, rgba(0,0,0,0.2));

  h1 {
    font-size: 2.5rem;
    margin-bottom: 0.5rem;
    
    &:hover {
      transform: scale(1.05);
      transition: transform 0.3s ease;
    }
  }
  
  p {
    font-size: 1.2rem;
    opacity: 0.9;
  }
}

.container {
  max-width: 1200px;
  margin: 2rem auto;
  padding: 0 1rem;
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 2rem;
  margin: 2rem 0;
}

.card {
  background: white;
  padding: 1.5rem;
  .border-radius(8px);
  .box-shadow();
  
  h3 {
    color: @primary-color;
    margin-bottom: 1rem;
  }
  
  p {
    color: lighten(@text-color, 20%);
  }
}

.button {
  display: inline-block;
  background-color: @primary-color;
  color: white;
  padding: 0.8rem 1.5rem;
  text-decoration: none;
  .border-radius(5px);
  .box-shadow(0, 2px, 5px, rgba(0,0,0,0.2));
  
  &:hover {
    background-color: darken(@primary-color, 10%);
    transform: translateY(-2px);
    transition: all 0.3s ease;
  }
}

// Responsive design
@media (max-width: 768px) {
  .header h1 {
    font-size: 2rem;
  }
  
  .grid {
    grid-template-columns: 1fr;
  }
}
```

**index.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Web Development Project</title>
    <link rel="stylesheet" href="dist/css/main.css">
</head>
<body>
    <header class="header">
        <h1>🚀 Web Development Project</h1>
        <p>GitLab CI/CD + SonarQube + LESS Compilation</p>
    </header>

    <div class="container">
        <section class="grid">
            <div class="card">
                <h3>HTML5</h3>
                <p>Modern semantic markup with accessibility features.</p>
            </div>
            
            <div class="card">
                <h3>CSS3/LESS</h3>
                <p>Advanced styling with variables, mixins, and responsive design.</p>
            </div>
            
            <div class="card">
                <h3>JavaScript</h3>
                <p>Interactive features and modern ES6+ syntax.</p>
            </div>
        </section>

        <section class="features">
            <h2>CI/CD Pipeline Features</h2>
            <div class="grid">
                <div class="card">
                    <h3>✅ Code Quality</h3>
                    <p>Automatic code analysis with SonarQube</p>
                </div>
                
                <div class="card">
                    <h3>🔧 LESS Compilation</h3>
                    <p>Automated LESS to CSS compilation</p>
                </div>
                
                <div class="card">
                    <h3>🚀 Deployment</h3>
                    <p>Automated testing and deployment</p>
                </div>
            </div>
        </section>

        <div style="text-align: center; margin: 3rem 0;">
            <a href="#" class="button">Get Started</a>
            <a href="#" class="button" style="background-color: #2ecc71;">View Demo</a>
        </div>
    </div>

    <script src="dist/js/app.min.js"></script>
</body>
</html>
```

**js/app.js**
```javascript
// Main application JavaScript
class WebApp {
    constructor() {
        this.init();
    }

    init() {
        console.log('🚀 Web Application Initialized');
        this.setupEventListeners();
        this.loadFeatures();
    }

    setupEventListeners() {
        // Smooth scrolling for anchor links
        document.querySelectorAll('a[href^="#"]').forEach(anchor => {
            anchor.addEventListener('click', function (e) {
                e.preventDefault();
                document.querySelector(this.getAttribute('href')).scrollIntoView({
                    behavior: 'smooth'
                });
            });
        });

        // Card hover effects
        document.querySelectorAll('.card').forEach(card => {
            card.addEventListener('mouseenter', this.animateCard);
            card.addEventListener('mouseleave', this.resetCard);
        });
    }

    animateCard(e) {
        this.style.transform = 'translateY(-5px)';
        this.style.transition = 'transform 0.3s ease';
    }

    resetCard(e) {
        this.style.transform = 'translateY(0)';
    }

    loadFeatures() {
        // Dynamic feature loading
        const features = [
            'Code Quality Analysis',
            'LESS Compilation',
            'Automated Testing',
            'CI/CD Pipeline'
        ];

        console.log('📋 Available Features:', features);
    }
}

// Initialize application when DOM is loaded
document.addEventListener('DOMContentLoaded', () => {
    new WebApp();
});

// Utility functions
const utils = {
    debounce(func, wait) {
        let timeout;
        return function executedFunction(...args) {
            const later = () => {
                clearTimeout(timeout);
                func(...args);
            };
            clearTimeout(timeout);
            timeout = setTimeout(later, wait);
        };
    },

    throttle(func, limit) {
        let inThrottle;
        return function(...args) {
            if (!inThrottle) {
                func.apply(this, args);
                inThrottle = true;
                setTimeout(() => inThrottle = false, limit);
            }
        };
    }
};
```

---

Продолжаю с CI/CD конфигурацией, настройкой SonarQube и скриптами управления:

## Интеграция CI/CD с SonarQube

### Настройка SonarQube для веб-технологий

```bash
#!/bin/bash
# setup-sonarqube.sh

echo "🔧 НАСТРОЙКА SONARQUBE ДЛЯ ВЕБ-РАЗРАБОТКИ"

echo "📝 Инструкция по настройке SonarQube:"
echo ""
echo "1. 🌐 Откройте: http://sonarqube.localdomain:9000"
echo "2. 🔐 Войдите: admin/admin"
echo "3. 🔒 Смените пароль на 'Netology2024!'"
echo "4. 📦 Установите плагины:"
echo "   • SonarCSS (для CSS/LESS анализа)"
echo "   • SonarHTML (для HTML анализа)" 
echo "   • SonarPHP (для PHP анализа)"
echo "   • SonarJS (уже предустановлен)"
echo "5. 🎫 Создайте токен:"
echo "   • My Account → Security → Generate Token"
echo "   • Название: gitlab-web-token"
echo "   • Скопируйте значение токена"
echo ""
echo "⚠️  ЗАПИШИТЕ ТОКЕН: ________________"

read -p "Нажмите Enter после завершения настройки SonarQube..."
```

### .gitlab-ci.yml для веб-разработки

```yaml
# .gitlab-ci.yml - Полный пайплайн для веб-разработки
stages:
  - test
  - quality
  - build
  - deploy

variables:
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"
  NODE_VERSION: "18"

# Кеширование для ускорения сборок
cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - node_modules/
    - .sonar/cache
    - dist/

# Анализ кода SonarQube
sonarqube-analysis:
  stage: quality
  image: 
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_SCANNER_OPTS: "-Xmx512m"
  before_script:
    # Гарантируем разрешение доменных имен
    - echo "192.168.56.20 sonarqube.localdomain" >> /etc/hosts
    - ping -c 2 sonarqube.localdomain
  script:
    - echo "🔍 Запуск анализа кода SonarQube..."
    - sonar-scanner
      -Dsonar.projectKey=${CI_PROJECT_NAME}
      -Dsonar.projectName=${CI_PROJECT_NAME}
      -Dsonar.projectVersion=${CI_COMMIT_SHORT_SHA}
      -Dsonar.host.url=${SONAR_HOST_URL}
      -Dsonar.token=${SONAR_TOKEN}
      -Dsonar.sources=.
      -Dsonar.sourceEncoding=UTF-8
      -Dsonar.exclusions=node_modules/**,dist/**,**/*.min.js
      -Dsonar.tests=./tests
      -Dsonar.test.inclusions=**/*.test.js
      -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
      -Dsonar.coverage.exclusions=**/*.test.js,tests/**,node_modules/**
  artifacts:
    reports:
      codequality: gl-sonar-report.json
    paths:
      - .sonar/cache
    expire_in: 1 week
  allow_failure: false
  only:
    - main
    - merge_requests
  tags:
    - web
    - docker

# Установка зависимостей
install-dependencies:
  stage: test
  image: node:${NODE_VERSION}-alpine
  before_script:
    - apk add --no-cache git
  script:
    - echo "📦 Установка зависимостей Node.js..."
    - npm ci --cache .npm --prefer-offline
    - npm audit --audit-level moderate
  cache:
    key: "node-modules-${CI_COMMIT_REF_SLUG}"
    paths:
      - node_modules/
      - .npm/
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 hour
  only:
    - main
    - merge_requests
  tags:
    - web
    - docker

# Компиляция LESS
compile-less:
  stage: build
  image: node:${NODE_VERSION}-alpine
  dependencies:
    - install-dependencies
  before_script:
    - apk add --no-cache less
  script:
    - echo "🎨 Компиляция LESS в CSS..."
    - mkdir -p dist/css
    - npx lessc src/less/main.less dist/css/main.css --clean-css
    - npx lessc src/less/main.less dist/css/main.min.css --clean-css="--s1 --advanced"
    - echo "✅ LESS скомпилирован успешно"
    - ls -la dist/css/
  artifacts:
    name: "compiled-css"
    paths:
      - dist/css/
    expire_in: 1 week
  only:
    - main
    - merge_requests
  tags:
    - web
    - docker

# Минификация JavaScript
minify-js:
  stage: build
  image: node:${NODE_VERSION}-alpine
  dependencies:
    - install-dependencies
  script:
    - echo "⚡ Минификация JavaScript..."
    - mkdir -p dist/js
    - npx uglify-js js/app.js -o dist/js/app.min.js --compress --mangle
    - echo "✅ JavaScript минифицирован успешно"
    - ls -la dist/js/
  artifacts:
    name: "minified-js"
    paths:
      - dist/js/
    expire_in: 1 week
  only:
    - main
  tags:
    - web
    - docker

# Проверка HTML
html-validation:
  stage: test
  image: node:${NODE_VERSION}-alpine
  before_script:
    - apk add --no-cache curl
    - npm install -g html-validator
  script:
    - echo "📄 Проверка валидности HTML..."
    - curl -s http://localhost:8080 2>/dev/null || cat index.html | html-validator --format=text
    - echo "✅ HTML проверка завершена"
  allow_failure: true
  only:
    - main
    - merge_requests
  tags:
    - web
    - docker

# Сборка проекта
build-project:
  stage: build
  image: node:${NODE_VERSION}-alpine
  dependencies:
    - compile-less
    - minify-js
  script:
    - echo "🏗️ Финальная сборка проекта..."
    - mkdir -p dist/assets
    - cp index.html dist/
    - cp -r images/ dist/ 2>/dev/null || true
    - echo "📊 Размер итоговых файлов:"
    - du -sh dist/
    - find dist/ -type f -name "*.css" -o -name "*.js" | xargs ls -lh
  artifacts:
    name: "web-project-build"
    paths:
      - dist/
    expire_in: 2 weeks
  only:
    - main
  tags:
    - web
    - docker

# Деплой в тестовое окружение (пример)
deploy-staging:
  stage: deploy
  image: alpine:latest
  dependencies:
    - build-project
  before_script:
    - apk add --no-cache curl
  script:
    - echo "🚀 Деплой в тестовое окружение..."
    - echo "📁 Собранные файлы:"
    - ls -la dist/
    - echo "✅ Деплой завершен (заглушка)"
  environment:
    name: staging
    url: http://staging.example.com
  only:
    - main
  when: manual
  tags:
    - web
    - docker

# Уведомление о успешной сборке
notify-success:
  stage: deploy
  image: alpine:latest
  script:
    - echo "🎉 Пайплайн успешно завершен!"
    - echo "📊 Статистика:"
    - echo "   - Коммит: ${CI_COMMIT_SHORT_SHA}"
    - echo "   - Ветка: ${CI_COMMIT_REF_NAME}"
    - echo "   - Автор: ${CI_COMMIT_AUTHOR}"
  only:
    - main
  when: on_success
  tags:
    - web
    - docker
```

### sonar-project.properties

```properties
# Конфигурация SonarQube для веб-проекта
sonar.projectKey=web-dev-project
sonar.projectName=Web Development Project
sonar.projectVersion=1.0

# Пути к исходному коду
sonar.sources=.,src,js
sonar.tests=./tests

# Исключения
sonar.exclusions=node_modules/**,dist/**,**/*.min.js,**/*.min.css

# Кодировка
sonar.sourceEncoding=UTF-8

# Настройки для веб-технологий
sonar.html.file.suffixes=.html,.htm
sonar.css.file.suffixes=.css,.less
sonar.js.file.suffixes=.js

# Настройки качества кода
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300

# Настройки для LESS анализа
sonar.less.parser.less=src/less/main.less

# Метаданные проекта
sonar.projectDescription=Web development project with GitLab CI/CD and SonarQube analysis
sonar.links.homepage=http://gitlab.localdomain/root/web-dev-project
sonar.links.ci=http://gitlab.localdomain/root/web-dev-project/-/pipelines
sonar.links.scm=http://gitlab.localdomain/root/web-dev-project.git
```

---

## Управление и мониторинг

### Комплексный скрипт проверки системы

```bash
#!/bin/bash
# system-check.sh

echo "=== 🔍 КОМПЛЕКСНАЯ ДИАГНОСТИКА СИСТЕМЫ ==="
echo "Версия: V7 - $(date)"
echo ""

cd ~/gitlab-setup

# 1. Проверка Docker сервисов
echo "1. 🐳 ПРОВЕРКА DOCKER СЕРВИСОВ"
echo "----------------------------------------"
docker-compose ps
echo ""

# 2. Проверка использования ресурсов
echo "2. 📊 ИСПОЛЬЗОВАНИЕ РЕСУРСОВ"
echo "----------------------------------------"
echo "💾 Память:"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}" | head -10
echo ""

echo "💽 Дисковое пространство:"
docker system df
echo ""

# 3. Проверка сетевой связности
echo "3. 🌐 СЕТЕВАЯ ДИАГНОСТИКА"
echo "----------------------------------------"
echo "📡 Проверка сети Docker:"
docker network inspect gitlab-network --format '{{range .Containers}}{{.Name}} - {{.IPv4Address}}{{"\n"}}{{end}}'
echo ""

echo "🔗 Проверка связи между контейнерами:"
containers=("gitlab" "sonarqube" "gitlab-runner")
for container in "${containers[@]}"; do
    if docker ps | grep -q $container; then
        echo "✅ $container: запущен"
    else
        echo "❌ $container: не запущен"
    fi
done
echo ""

# 4. Проверка доступности сервисов
echo "4. 🚀 ПРОВЕРКА ДОСТУПНОСТИ СЕРВИСОВ"
echo "----------------------------------------"

check_service() {
    local name=$1
    local url=$2
    local expected=$3
    
    echo -n "🔍 $name: "
    if curl -s -f "$url" > /dev/null; then
        echo "✅ ДОСТУПЕН"
        return 0
    else
        echo "❌ НЕДОСТУПЕН"
        return 1
    fi
}

check_service "GitLab" "http://gitlab.localdomain" "200"
check_service "SonarQube" "http://sonarqube.localdomain:9000" "200"

echo ""

# 5. Проверка GitLab Runner
echo "5. ⚙️ ПРОВЕРКА GITLAB RUNNER"
echo "----------------------------------------"
if docker ps | grep -q gitlab-runner; then
    echo "✅ GitLab Runner запущен"
    docker-compose exec gitlab-runner gitlab-runner list 2>/dev/null || echo "❌ Runner не зарегистрирован"
else
    echo "❌ GitLab Runner не запущен"
fi
echo ""

# 6. Проверка логов на ошибки
echo "6. 📋 АНАЛИЗ ЛОГОВ НА ОШИБКИ"
echo "----------------------------------------"
services=("gitlab" "sonarqube" "gitlab-runner")
for service in "${services[@]}"; do
    echo "🔍 Поиск ошибок в $service:"
    docker-compose logs "$service" --tail=20 | grep -i error | head -5 || echo "   ✅ Ошибок не найдено"
done
echo ""

# 7. Проверка здоровья сервисов
echo "7. ❤️  ПРОВЕРКА ЗДОРОВЬЯ СЕРВИСОВ"
echo "----------------------------------------"
for service in "${containers[@]}"; do
    health=$(docker inspect --format='{{.State.Health.Status}}' "$service" 2>/dev/null || echo "no health check")
    echo "🏥 $service: $health"
done
echo ""

# 8. Проверка volumes
echo "8. 💾 ПРОВЕРКА VOLUMES"
echo "----------------------------------------"
docker volume ls | grep gitlab-setup
echo ""

# 9. Итоговая проверка
echo "9. 🎯 ИТОГОВАЯ ДИАГНОСТИКА"
echo "----------------------------------------"

ALL_OK=true

# Критические проверки
if ! curl -s http://gitlab.localdomain > /dev/null; then
    echo "❌ КРИТИЧЕСКАЯ ОШИБКА: GitLab недоступен"
    ALL_OK=false
fi

if ! curl -s http://sonarqube.localdomain:9000 > /dev/null; then
    echo "❌ КРИТИЧЕСКАЯ ОШИБКА: SonarQube недоступен"
    ALL_OK=false
fi

if ! docker ps | grep -q gitlab-runner; then
    echo "❌ КРИТИЧЕСКАЯ ОШИБКА: GitLab Runner не запущен"
    ALL_OK=false
fi

if [ "$ALL_OK" = true ]; then
    echo ""
    echo "🎉 ВСЕ СИСТЕМЫ РАБОТАЮТ НОРМАЛЬНО!"
    echo ""
    echo "🌐 ДОСТУПНЫЕ СЕРВИСЫ:"
    echo "   • GitLab:       http://gitlab.localdomain"
    echo "   • SonarQube:    http://sonarqube.localdomain:9000"
    echo "   • Проекты:      http://gitlab.localdomain/root/"
    echo ""
    echo "🚀 СИСТЕМА ГОТОВА К РАБОТЕ!"
else
    echo ""
    echo "⚠️  ОБНАРУЖЕНЫ ПРОБЛЕМЫ!"
    echo "💡 Запустите скрипт устранения неполадок: ./troubleshoot.sh"
fi

echo ""
echo "=== ДИАГНОСТИКА ЗАВЕРШЕНА ==="
```

### Скрипт устранения неполадок

```bash
#!/bin/bash
# troubleshoot.sh

echo "=== 🛠️  УСТРАНЕНИЕ НЕПОЛАДОК ==="

cd ~/gitlab-setup

# Функция логирования
log() {
    echo "📝 $(date): $1"
}

# 1. Проверка прав Docker
log "Проверка прав Docker..."
docker ps > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "❌ Нет доступа к Docker. Проверьте:"
    echo "   - Запущен ли Docker daemon"
    echo "   - Добавлен ли пользователь в группу docker"
    echo "   - Права на /var/run/docker.sock"
    exit 1
fi
echo "✅ Docker доступен"

# 2. Проверка docker-compose.yml
log "Проверка конфигурации docker-compose..."
if [ ! -f "docker-compose.yml" ]; then
    echo "❌ Файл docker-compose.yml не найден"
    exit 1
fi
echo "✅ docker-compose.yml найден"

# 3. Перезапуск сервисов
log "Перезапуск сервисов..."
docker-compose down
sleep 5
docker-compose up -d

# 4. Ожидание запуска
log "Ожидание запуска сервисов..."
sleep 30

# 5. Проверка состояния
log "Проверка состояния сервисов..."
for service in gitlab sonarqube gitlab-runner; do
    if docker-compose ps | grep -q "$service.*Up"; then
        echo "✅ $service запущен"
    else
        echo "❌ $service не запущен"
        echo "Логи $service:"
        docker-compose logs "$service" --tail=10
    fi
done

# 6. Проверка сети
log "Проверка сети..."
docker network inspect gitlab-network > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "❌ Сеть gitlab-network не найдена. Создаем..."
    docker network create --driver=bridge --subnet=192.168.56.0/24 gitlab-network
fi

# 7. Проверка hosts файла
log "Проверка hosts файла..."
if ! grep -q "gitlab.localdomain" /etc/hosts; then
    echo "❌ Записи в hosts файле отсутствуют. Добавляем..."
    sudo tee -a /etc/hosts << EOL
192.168.56.10    gitlab.localdomain
192.168.56.20    sonarqube.localdomain
EOL
fi

# 8. Проверка портов
log "Проверка занятых портов..."
for port in 80 443 9000; do
    if netstat -tuln | grep -q ":$port "; then
        echo "⚠️  Порт $port занят"
    else
        echo "✅ Порт $port свободен"
    fi
done

# 9. Проверка ресурсов
log "Проверка системных ресурсов..."
echo "💾 Свободная память: $(free -h | grep Mem | awk '{print $4}')"
echo "💽 Свободное место: $(df -h / | tail -1 | awk '{print $4}')"

# 10. Финальная проверка
log "Финальная проверка..."
./system-check.sh

echo ""
echo "✅ УСТРАНЕНИЕ НЕПОЛАДОК ЗАВЕРШЕНО"
```

### Скрипты управления

```bash
#!/bin/bash
# manage-services.sh

case "$1" in
    start)
        echo "🚀 Запуск всех сервисов..."
        cd ~/gitlab-setup
        docker-compose up -d
        ;;
    stop)
        echo "🛑 Остановка всех сервисов..."
        cd ~/gitlab-setup
        docker-compose stop
        ;;
    restart)
        echo "🔁 Перезапуск всех сервисов..."
        cd ~/gitlab-setup
        docker-compose restart
        ;;
    status)
        echo "📊 Статус сервисов..."
        cd ~/gitlab-setup
        docker-compose ps
        ;;
    logs)
        echo "📋 Логи сервисов..."
        cd ~/gitlab-setup
        docker-compose logs -f ${2:-gitlab}
        ;;
    backup)
        echo "💾 Создание бэкапа..."
        cd ~/gitlab-setup
        docker-compose exec gitlab gitlab-backup create
        tar -czf backup-$(date +%Y%m%d-%H%M%S).tar.gz docker-compose.yml *.sh
        echo "✅ Бэкап создан"
        ;;
    clean)
        echo "🧹 Очистка ненужных данных..."
        docker system prune -f
        docker volume prune -f
        ;;
    *)
        echo "Использование: $0 {start|stop|restart|status|logs [service]|backup|clean}"
        exit 1
        ;;
esac
```

---

## Дополнительные настройки

### Настройка WSL2 для оптимальной производительности

```bash
#!/bin/bash
# optimize-wsl2.sh

echo "⚡ ОПТИМИЗАЦИЯ WSL2 ДЛЯ DOCKER РАЗРАБОТКИ"

# Создание конфигурации WSL2
cat > /mnt/c/Users/$USER/.wslconfig << 'EOF'
[wsl2]
# Лимиты ресурсов
memory=12GB
processors=6
swap=4GB
swapfile=%USERPROFILE%\wsl-swap.vhdx

# Сетевые настройки
localhostForwarding=true
dnsTunneling=true
firewall=true
autoProxy=true

# Файловая система
pageReporting=true
kernelCommandLine=intel_pstate=disable

# Производительность
vmIdleTimeout=3600000

# Экспериментальные функции
[experimental]
autoMemoryReclaim=dropcache
sparseVhd=true
dnsTunneling=true
hostAddressLoopback=true
EOF

echo "✅ Конфигурация WSL2 применена"
echo "🔁 Для применения настроек выполните: wsl --shutdown && wsl"
```

### Мониторинг в реальном времени

```bash
#!/bin/bash
# monitor.sh

echo "📊 РЕАЛЬНОЕ ВРЕМЯ МОНИТОРИНГА"

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

while true; do
    clear
    echo -e "${YELLOW}=== 🐳 DOCKER MONITOR ===${NC}"
    echo "$(date)"
    echo ""
    
    # Статус контейнеров
    echo -e "${YELLOW}📦 СТАТУС КОНТЕЙНЕРОВ:${NC}"
    docker-compose ps
    
    echo ""
    echo -e "${YELLOW}📊 ИСПОЛЬЗОВАНИЕ РЕСУРСОВ:${NC}"
    docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}" | head -10
    
    echo ""
    echo -e "${YELLOW}🌐 ДОСТУПНОСТЬ СЕРВИСОВ:${NC}"
    
    # Проверка GitLab
    if curl -s http://gitlab.localdomain > /dev/null; then
        echo -e "GitLab: ${GREEN}✅ ONLINE${NC}"
    else
        echo -e "GitLab: ${RED}❌ OFFLINE${NC}"
    fi
    
    # Проверка SonarQube
    if curl -s http://sonarqube.localdomain:9000 > /dev/null; then
        echo -e "SonarQube: ${GREEN}✅ ONLINE${NC}"
    else
        echo -e "SonarQube: ${RED}❌ OFFLINE${NC}"
    fi
    
    echo ""
    echo -e "${YELLOW}💾 СИСТЕМНЫЕ РЕСУРСЫ:${NC}"
    echo "Память: $(free -h | grep Mem | awk '{print $3 "/" $2 " (" $4 " свободно)"}')"
    echo "Диск: $(df -h / | tail -1 | awk '{print $3 "/" $2 " (" $5 " занято)"}')"
    
    echo ""
    echo -e "🔄 Обновление через 10 секунд (Ctrl+C для выхода)..."
    sleep 10
done
```

---

## Заключение

### 🎯 Что было улучшено в V7:

1. **Оптимизированные настройки ресурсов** - сбалансированное распределение памяти и CPU
2. **Полная поддержка веб-технологий** - LESS, CSS, HTML, JavaScript
3. **Улучшенные скрипты автоматизации** - быстрый деплой и диагностика
4. **Расширенный мониторинг** - реальное время и детальная диагностика
5. **Готовые примеры кода** - полная структура веб-проекта
6. **Улучшенная документация** - пошаговые инструкции и устранение неполадок

### 🚀 Быстрый старт:

```bash
# 1. Клонирование и настройка
cd ~
mkdir gitlab-setup && cd gitlab-setup

# 2. Скачивание скриптов (или создание вручную)
# 3. Запуск автоматического развертывания
chmod +x *.sh
./quick-deploy.sh

# 4. Мониторинг запуска
./wait-for-gitlab.sh

# 5. Проверка системы
./system-check.sh
```

### 📊 Ключевые сервисы:

- **GitLab**: http://gitlab.localdomain
- **SonarQube**: http://sonarqube.localdomain:9000  
- **Документация**: Встроенные скрипты помощи
- **Мониторинг**: ./monitor.sh

### 🔧 Управление:

```bash
./manage-services.sh start    # Запуск
./manage-services.sh status   # Статус
./manage-services.sh stop     # Остановка
./troubleshoot.sh            # Устранение неполадок
```

**Ваша система готова для профессиональной веб-разработки с полным циклом CI/CD!** 🎉

Добавляю комментарии о расположении файлов в соответствующие разделы:

## Интеграция CI/CD с SonarQube

### .gitlab-ci.yml для веб-разработки

```yaml
# .gitlab-ci.yml - Полный пайплайн для веб-разработки
# 📍 РАСПОЛОЖЕНИЕ: Корень репозитория GitLab проекта
# 📁 Путь: ~/web-dev-project/.gitlab-ci.yml
# 💡 Этот файл должен находиться в корне вашего GitLab репозитория
# 💡 Он автоматически обнаруживается GitLab при пуше в репозиторий

stages:
  - test
  - quality
  - build
  - deploy

variables:
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"
  NODE_VERSION: "18"

# Кеширование для ускорения сборок
cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - node_modules/
    - .sonar/cache
    - dist/

# Анализ кода SonarQube
sonarqube-analysis:
  stage: quality
  image: 
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_SCANNER_OPTS: "-Xmx512m"
  before_script:
    # Гарантируем разрешение доменных имен
    - echo "192.168.56.20 sonarqube.localdomain" >> /etc/hosts
    - ping -c 2 sonarqube.localdomain
  script:
    - echo "🔍 Запуск анализа кода SonarQube..."
    - sonar-scanner
      -Dsonar.projectKey=${CI_PROJECT_NAME}
      -Dsonar.projectName=${CI_PROJECT_NAME}
      -Dsonar.projectVersion=${CI_COMMIT_SHORT_SHA}
      -Dsonar.host.url=${SONAR_HOST_URL}
      -Dsonar.token=${SONAR_TOKEN}
      -Dsonar.sources=.
      -Dsonar.sourceEncoding=UTF-8
      -Dsonar.exclusions=node_modules/**,dist/**,**/*.min.js
      -Dsonar.tests=./tests
      -Dsonar.test.inclusions=**/*.test.js
      -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
      -Dsonar.coverage.exclusions=**/*.test.js,tests/**,node_modules/**
  artifacts:
    reports:
      codequality: gl-sonar-report.json
    paths:
      - .sonar/cache
    expire_in: 1 week
  allow_failure: false
  only:
    - main
    - merge_requests
  tags:
    - web
    - docker

# ... остальная часть файла ...
```

### sonar-project.properties

```properties
# Конфигурация SonarQube для веб-проекта
# 📍 РАСПОЛОЖЕНИЕ: Корень репозитория GitLab проекта  
# 📁 Путь: ~/web-dev-project/sonar-project.properties
# 💡 Этот файл должен находиться в корне вашего GitLab репозитория
# 💡 Он используется sonar-scanner для конфигурации анализа

sonar.projectKey=web-dev-project
sonar.projectName=Web Development Project
sonar.projectVersion=1.0

# Пути к исходному коду
sonar.sources=.,src,js
sonar.tests=./tests

# Исключения
sonar.exclusions=node_modules/**,dist/**,**/*.min.js,**/*.min.css

# Кодировка
sonar.sourceEncoding=UTF-8

# Настройки для веб-технологий
sonar.html.file.suffixes=.html,.htm
sonar.css.file.suffixes=.css,.less
sonar.js.file.suffixes=.js

# Настройки качества кода
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300

# Настройки для LESS анализа
sonar.less.parser.less=src/less/main.less

# Метаданные проекта
sonar.projectDescription=Web development project with GitLab CI/CD and SonarQube analysis
sonar.links.homepage=http://gitlab.localdomain/root/web-dev-project
sonar.links.ci=http://gitlab.localdomain/root/web-dev-project/-/pipelines
sonar.links.scm=http://gitlab.localdomain/root/web-dev-project.git
```

## Создание и настройка веб-проекта

### Структура веб-проекта с комментариями о расположении файлов

```bash
# Создание структуры проекта
mkdir -p ~/web-dev-project/{css,js,src/less,dist}
cd ~/web-dev-project

# 📁 СТРУКТУРА ПРОЕКТА:
# ~/web-dev-project/
# ├── .gitlab-ci.yml              📍 КОРЕНЬ - Обнаруживается GitLab автоматически
# ├── sonar-project.properties    📍 КОРЕНЬ - Используется sonar-scanner
# ├── package.json                📍 КОРЕНЬ - Конфигурация npm
# ├── index.html                  📍 КОРЕНЬ - Главная страница
# ├── css/                        📁 Папка для скомпилированных CSS файлов
# ├── js/                         📁 Исходные JavaScript файлы
# │   └── app.js
# ├── src/less/                   📁 Исходные LESS файлы
# │   └── main.less
# └── dist/                       📁 Собранные файлы (артефакты)
#     ├── css/
#     └── js/
```

### Инициализация Git репозитория с правильным расположением файлов

```bash
#!/bin/bash
# setup-web-project.sh

echo "🎯 НАСТРОЙКА ВЕБ-ПРОЕКТА С ПРАВИЛЬНЫМ РАСПОЛОЖЕНИЕМ ФАЙЛОВ"

cd ~/web-dev-project

echo "📁 Создание структуры проекта..."
mkdir -p {css,js,src/less,dist/{css,js},tests}

echo "📝 Создание конфигурационных файлов в корне проекта..."

# Создание .gitlab-ci.yml В КОРНЕ проекта
cat > .gitlab-ci.yml << 'EOF'
# 📍 РАСПОЛОЖЕНИЕ: КОРЕНЬ репозитория
# GitLab автоматически обнаруживает этот файл при коммитах
# Не перемещайте его в поддиректории!

stages:
  - test
  - quality
  - build

sonarqube-analysis:
  stage: quality
  image: sonarsource/sonar-scanner-cli:latest
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  script:
    - sonar-scanner
  only:
    - main
  tags:
    - web

compile-less:
  stage: build
  image: node:18-alpine
  script:
    - npm install -g less
    - npx lessc src/less/main.less dist/css/main.css
  artifacts:
    paths:
      - dist/css/
  only:
    - main
  tags:
    - web
EOF

echo "✅ .gitlab-ci.yml создан в корне проекта"

# Создание sonar-project.properties В КОРНЕ проекта
cat > sonar-project.properties << 'EOF'
# 📍 РАСПОЛОЖЕНИЕ: КОРЕНЬ репозитория  
# SonarScanner ищет этот файл в корне проекта
# Не перемещайте его в поддиректории!

sonar.projectKey=web-dev-project
sonar.projectName=Web Development Project
sonar.sources=.,src,js
sonar.exclusions=node_modules/**,dist/**,**/*.min.*
sonar.sourceEncoding=UTF-8
sonar.host.url=http://sonarqube.localdomain:9000
EOF

echo "✅ sonar-project.properties создан в корне проекта"

# Создание package.json В КОРНЕ проекта
cat > package.json << 'EOF'
{
  "name": "web-dev-project",
  "scripts": {
    "build": "lessc src/less/main.less dist/css/main.css"
  }
}
EOF

echo "✅ package.json создан в корне проекта"

# Создание index.html В КОРНЕ проекта
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Web Project</title>
    <link rel="stylesheet" href="dist/css/main.css">
</head>
<body>
    <h1>Web Development Project</h1>
</body>
</html>
EOF

echo "✅ index.html создан в корне проекта"

# Создание LESS файла в правильной директории
cat > src/less/main.less << 'EOF'
@primary-color: #3498db;

body {
  background: @primary-color;
  color: white;
}
EOF

echo "✅ main.less создан в src/less/"

echo ""
echo "📋 ПРОВЕРКА СТРУКТУРЫ ПРОЕКТА:"
tree -a ~/web-dev-project

echo ""
echo "🎯 ВАЖНЫЕ ЗАМЕЧАНИЯ ПО РАСПОЛОЖЕНИЮ ФАЙЛОВ:"
echo "✅ .gitlab-ci.yml - ДОЛЖЕН находиться в КОРНЕ репозитория"
echo "✅ sonar-project.properties - ДОЛЖЕН находиться в КОРНЕ репозитория" 
echo "✅ GitLab автоматически обнаруживает .gitlab-ci.yml при пуше"
echo "✅ SonarScanner ищет sonar-project.properties в корне проекта"
echo ""
echo "🚀 Теперь можно инициализировать Git репозиторий:"
echo "   cd ~/web-dev-project"
echo "   git init"
echo "   git add ."
echo "   git commit -m 'Initial commit'"
echo "   git remote add origin http://gitlab.localdomain/root/web-dev-project.git"
echo "   git push -u origin main"
```

## Проверка правильного расположения файлов

```bash
#!/bin/bash
# check-project-structure.sh

echo "🔍 ПРОВЕРКА ПРАВИЛЬНОГО РАСПОЛОЖЕНИЯ ФАЙЛОВ"

cd ~/web-dev-project

echo "📁 Текущая структура проекта:"
echo ""

# Проверка критически важных файлов в корне
critical_files=(".gitlab-ci.yml" "sonar-project.properties" "package.json" "index.html")

for file in "${critical_files[@]}"; do
    if [ -f "$file" ]; then
        echo "✅ $file - найден в корне (ПРАВИЛЬНО)"
    else
        echo "❌ $file - ОТСУТСТВУЕТ в корне (НЕПРАВИЛЬНО!)"
    fi
done

echo ""
echo "📊 Общая структура:"
tree -I 'node_modules|dist|.git' ~/web-dev-project

echo ""
echo "💡 РЕКОМЕНДАЦИИ ПО РАСПОЛОЖЕНИЮ:"
echo "   • .gitlab-ci.yml → 📍 КОРЕНЬ (обязательно)"
echo "   • sonar-project.properties → 📍 КОРЕНЬ (обязательно)" 
echo "   • package.json → 📍 КОРЕНЬ (рекомендуется)"
echo "   • index.html → 📍 КОРЕНЬ (рекомендуется)"
echo "   • src/less/ → 📁 Папка для исходников LESS"
echo "   • js/ → 📁 Папка для JavaScript"
echo "   • dist/ → 📁 Папка для сборки (автоматически создается)"
echo ""
echo "🚀 Если структура правильная, можно пушить в GitLab!"
```

## Интеграция с GitLab - финальные шаги

### Скрипт полной настройки проекта

```bash
#!/bin/bash
# setup-complete-project.sh

echo "🚀 ПОЛНАЯ НАСТРОЙКА ПРОЕКТА В GITLAB"

cd ~/web-dev-project

# 1. Проверка структуры
echo "1. 🔍 Проверка структуры проекта..."
./check-project-structure.sh

# 2. Инициализация Git
echo ""
echo "2. 🔧 Инициализация Git репозитория..."
git init
git config --local user.name "root"
git config --local user.email "root@gitlab.localdomain"

# 3. Добавление файлов
echo ""
echo "3. 📦 Добавление файлов в Git..."
git add .

# 4. Проверка что .gitlab-ci.yml и sonar-project.properties добавлены
echo ""
echo "4. 📋 Проверка добавленных файлов..."
if git status | grep -q ".gitlab-ci.yml"; then
    echo "✅ .gitlab-ci.yml добавлен в коммит"
else
    echo "❌ .gitlab-ci.yml НЕ добавлен в коммит!"
fi

if git status | grep -q "sonar-project.properties"; then
    echo "✅ sonar-project.properties добавлен в коммит"
else
    echo "❌ sonar-project.properties НЕ добавлен в коммит!"
fi

# 5. Создание коммита
echo ""
echo "5. 💾 Создание коммита..."
git commit -m "Initial commit: Web project with CI/CD and SonarQube integration

- Added .gitlab-ci.yml for GitLab CI/CD pipeline
- Added sonar-project.properties for SonarQube analysis  
- Project structure for web development (HTML, LESS, JS)
- Ready for automated quality checks and LESS compilation"

# 6. Добавление удаленного репозитория
echo ""
echo "6. 🌐 Настройка удаленного репозитория..."
git remote add origin http://gitlab.localdomain/root/web-dev-project.git

# 7. Пуш в GitLab
echo ""
echo "7. 📤 Отправка в GitLab..."
git push -u origin main

echo ""
echo "🎉 ПРОЕКТ УСПЕШНО НАСТРОЕН И ОТПРАВЛЕН В GITLAB!"
echo ""
echo "📊 ДАЛЬНЕЙШИЕ ДЕЙСТВИЯ:"
echo "   • Откройте: http://gitlab.localdomain/root/web-dev-project"
echo "   • Перейдите: Settings → CI/CD → Variables"
echo "   • Добавьте переменные:"
echo "     - SONAR_HOST_URL = http://sonarqube.localdomain:9000"
echo "     - SONAR_TOKEN = [ваш_токен_из_sonarqube]"
echo "   • Проверьте пайплайн: CI/CD → Pipelines"
echo "   • Проверьте анализ: http://sonarqube.localdomain:9000"
echo ""
echo "📍 ВАЖНО: .gitlab-ci.yml и sonar-project.properties находятся в КОРНЕ проекта"
echo "   Это необходимо для корректной работы GitLab CI/CD и SonarQube"
```

## Итоговые рекомендации по расположению файлов

### 📍 КРИТИЧЕСКИ ВАЖНО - должны быть в корне:

1. **`.gitlab-ci.yml`** - GitLab автоматически обнаруживает этот файл только в корне репозитория
2. **`sonar-project.properties`** - SonarScanner по умолчанию ищет этот файл в корне проекта

### 📁 РЕКОМЕНДУЕТСЯ в корне:

3. **`package.json`** - стандартное расположение для Node.js проектов
4. **`index.html`** - главная страница веб-приложения

### 📁 СТРУКТУРА ПАПОК:

```
web-dev-project/
├── 📍 .gitlab-ci.yml              # ⚠️ ОБЯЗАТЕЛЬНО В КОРНЕ
├── 📍 sonar-project.properties    # ⚠️ ОБЯЗАТЕЛЬНО В КОРНЕ  
├── 📍 package.json
├── 📍 index.html
├── 📁 src/
│   └── 📁 less/
│       └── main.less
├── 📁 js/
│   └── app.js
├── 📁 tests/
└── 📁 dist/                       # Автоматически создается
    ├── css/
    └── js/
```

### 🔧 Проверка перед коммитом:

```bash
# Всегда проверяйте перед пушем в GitLab
cd ~/web-dev-project
ls -la .gitlab-ci.yml sonar-project.properties

# Должны видеть:
# -rw-r--r-- .gitlab-ci.yml
# -rw-r--r-- sonar-project.properties
```

Теперь расположение файлов четко указано с комментариями и скриптами проверки! 🎯
