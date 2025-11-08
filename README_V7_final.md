# 🚀 Практическое руководство: Развертывание GitLab с GitLab Runner и SonarQube в Docker для веб-разработки (V7 - ФИНАЛЬНАЯ ВЕРСИЯ)

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

---

## Введение

Это руководство предоставляет пошаговые инструкции по развертыванию собственного экземпляра GitLab, настроенного для веб-разработки (CSS, HTML, PHP, JavaScript), вместе с GitLab Runner и SonarQube в среде Docker. Мы настроим автоматизированный пайплайн CI/CD, который будет проверять код, включая анализ синтаксиса LESS, с помощью SonarQube и развертывать его.

**Ключевые особенности V7:**
- ✅ Оптимизированные настройки ресурсов для 4 ядер / 16GB RAM
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

### Распределение ресурсов

**После оптимизации (для 4 ядер / 16GB):**
- **GitLab**: 3GB RAM, 1.5 CPU ⬇️
- **SonarQube**: 2GB RAM, 1.0 CPU ⬇️
- **Runner**: 512MB RAM, 0.5 CPU ⬇️
- **Всего**: 5.5GB RAM, 3.0 CPU ⬇️

**WSL2 настройки:**
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

---

## Подготовка окружения

### Оптимизированные настройки WSL2 для 4 ядер / 16GB RAM

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

### Шаг 1: Создание структуры проекта

```bash
# Создаем директорию проекта
mkdir -p ~/web-dev-project
cd ~/web-dev-project

# Создаем структуру папок
mkdir -p css js src/less dist/css dist/js tests
```

### Шаг 2: Создание файлов проекта

**📍 Файл: `~/web-dev-project/index.html`**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simple Web Project</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <header>
        <h1>Simple Web Project</h1>
        <p>Testing GitLab CI/CD with SonarQube</p>
    </header>
    
    <main>
        <section>
            <h2>Features</h2>
            <ul>
                <li>HTML5 Structure</li>
                <li>LESS Stylesheets</li>
                <li>JavaScript Functionality</li>
                <li>CI/CD Pipeline</li>
            </ul>
        </section>
    </main>
    
    <footer>
        <p>&copy; 2024 Web Project</p>
    </footer>
    
    <script src="js/app.js"></script>
</body>
</html>
```

**📍 Файл: `~/web-dev-project/src/less/style.less`**
```less
// Variables
@primary-color: #3498db;
@secondary-color: #2ecc71;
@text-color: #2c3e50;
@background-color: #ecf0f1;

// Mixins
.border-radius(@radius: 5px) {
  border-radius: @radius;
}

.box-shadow(@x: 0, @y: 2px, @blur: 5px, @color: rgba(0,0,0,0.1)) {
  box-shadow: @x @y @blur @color;
}

// Base styles
body {
  font-family: Arial, sans-serif;
  line-height: 1.6;
  color: @text-color;
  background-color: @background-color;
  margin: 0;
  padding: 20px;
}

header {
  background: @primary-color;
  color: white;
  padding: 2rem;
  text-align: center;
  .border-radius(10px);
  .box-shadow(0, 4px, 10px, rgba(0,0,0,0.2));

  h1 {
    margin: 0;
    
    &:hover {
      color: lighten(@primary-color, 20%);
      transition: color 0.3s ease;
    }
  }
  
  p {
    margin: 0.5rem 0 0 0;
    opacity: 0.9;
  }
}

main {
  max-width: 800px;
  margin: 2rem auto;
  padding: 0 1rem;
}

section {
  background: white;
  padding: 1.5rem;
  .border-radius(8px);
  .box-shadow();
  
  h2 {
    color: @primary-color;
    margin-bottom: 1rem;
  }
  
  ul {
    list-style: none;
    padding: 0;
    
    li {
      padding: 0.5rem 0;
      border-bottom: 1px solid #eee;
      
      &:last-child {
        border-bottom: none;
      }
      
      &:hover {
        color: @secondary-color;
        transform: translateX(5px);
        transition: all 0.3s ease;
      }
    }
  }
}

footer {
  text-align: center;
  margin-top: 3rem;
  padding: 1rem;
  color: #7f8c8d;
}
```

**📍 Файл: `~/web-dev-project/js/app.js`**
```javascript
// Simple JavaScript functionality
document.addEventListener('DOMContentLoaded', function() {
    console.log('Web application loaded');
    
    // Add interactivity to list items
    const listItems = document.querySelectorAll('li');
    listItems.forEach(item => {
        item.addEventListener('click', function() {
            this.style.backgroundColor = '#f8f9fa';
            setTimeout(() => {
                this.style.backgroundColor = '';
            }, 300);
        });
    });
    
    // Add current year to footer
    const yearElement = document.querySelector('footer p');
    if (yearElement) {
        const currentYear = new Date().getFullYear();
        yearElement.textContent = `© ${currentYear} Web Project`;
    }
});
```

**📍 Файл: `~/web-dev-project/package.json`**
```json
{
  "name": "web-dev-project",
  "version": "1.0.0",
  "description": "Simple web project for GitLab CI/CD demonstration",
  "scripts": {
    "build:less": "lessc src/less/style.less css/style.css",
    "build": "npm run build:less"
  },
  "devDependencies": {
    "less": "^4.1.3"
  }
}
```

### Шаг 3: Инициализация Git репозитория

```bash
#!/bin/bash
# init-project.sh

echo "🎯 ИНИЦИАЛИЗАЦИЯ GIT РЕПОЗИТОРИЯ"

cd ~/web-dev-project

# Инициализация Git
git init

# Настройка пользователя
git config --local user.name "root"
git config --local user.email "root@gitlab.localdomain"

# Добавление файлов
git add .

# Создание коммита
git commit -m "Initial commit: Simple web project with HTML, LESS, and JavaScript

- Basic HTML structure
- LESS styles with variables and mixins
- Simple JavaScript functionality
- Package.json for build scripts
- Ready for CI/CD integration"

echo "✅ Git репозиторий инициализирован"
echo "📊 Статус репозитория:"
git status
```

---

## Интеграция CI/CD с SonarQube

### Шаг 1: Настройка SonarQube

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

### Шаг 2: Создание конфигурационных файлов CI/CD

**📍 Файл: `~/web-dev-project/.gitlab-ci.yml`**
```yaml
# 📍 РАСПОЛОЖЕНИЕ: Корень репозитория GitLab проекта
# GitLab автоматически обнаруживает этот файл при пуше в репозиторий

stages:
  - test
  - quality
  - build

variables:
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"
  NODE_VERSION: "18"

cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - node_modules/
    - .sonar/cache
    - css/

# Анализ кода SonarQube
sonarqube-analysis:
  stage: quality
  image: 
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_SCANNER_OPTS: "-Xmx512m"
  before_script:
    - echo "192.168.56.20 sonarqube.localdomain" >> /etc/hosts
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
      -Dsonar.exclusions=node_modules/**,css/**,**/*.min.js
      -Dsonar.tests=./tests
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

# Компиляция LESS
compile-less:
  stage: build
  image: node:${NODE_VERSION}-alpine
  before_script:
    - apk add --no-cache less
  script:
    - echo "🎨 Компиляция LESS в CSS..."
    - npx lessc src/less/style.less css/style.css --clean-css
    - echo "✅ LESS скомпилирован успешно"
    - ls -la css/
  artifacts:
    name: "compiled-css"
    paths:
      - css/
    expire_in: 1 week
  only:
    - main
    - merge_requests
  tags:
    - web
    - docker

# Уведомление об успешной сборке
notify-success:
  stage: build
  image: alpine:latest
  script:
    - echo "🎉 Пайплайн успешно завершен!"
    - echo "📊 Статистика:"
    - echo "   - Коммит: ${CI_COMMIT_SHORT_SHA}"
    - echo "   - Ветка: ${CI_COMMIT_REF_NAME}"
  only:
    - main
  when: on_success
  tags:
    - web
    - docker
```

**📍 Файл: `~/web-dev-project/sonar-project.properties`**
```properties
# 📍 РАСПОЛОЖЕНИЕ: Корень репозитория GitLab проекта
# SonarScanner ищет этот файл в корне проекта

sonar.projectKey=web-dev-project
sonar.projectName=Web Development Project
sonar.projectVersion=1.0

# Пути к исходному коду
sonar.sources=.,src,js
sonar.tests=./tests

# Исключения
sonar.exclusions=node_modules/**,css/**,**/*.min.js,**/*.min.css

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
sonar.less.parser.less=src/less/style.less
```

### Шаг 3: Настройка проекта в GitLab

```bash
#!/bin/bash
# setup-gitlab-project.sh

echo "🚀 НАСТРОЙКА ПРОЕКТА В GITLAB"

cd ~/web-dev-project

echo "📋 Проверка структуры проекта..."
./check-project-structure.sh

echo "🌐 Добавление удаленного репозитория..."
git remote add origin http://gitlab.localdomain/root/web-dev-project.git

echo "📤 Отправка кода в GitLab..."
git push -u origin main

echo ""
echo "🎉 ПРОЕКТ УСПЕШНО ЗАГРУЖЕН В GITLAB!"
echo ""
echo "📝 ДАЛЬНЕЙШИЕ ДЕЙСТВИЯ:"
echo "1. Откройте: http://gitlab.localdomain/root/web-dev-project"
echo "2. Перейдите: Settings → CI/CD → Variables"
echo "3. Добавьте переменные:"
echo "   - SONAR_HOST_URL = http://sonarqube.localdomain:9000"
echo "   - SONAR_TOKEN = [ваш_токен_из_sonarqube]"
echo "4. Проверьте пайплайн: CI/CD → Pipelines"
echo "5. Проверьте анализ: http://sonarqube.localdomain:9000"
```

---

## Управление и мониторинг

### Основные команды управления

```bash
# Все команды выполняются в директории gitlab-setup
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

```bash
#!/bin/bash
# reset-project.sh

echo "=== 🗑️ БЕЗОПАСНАЯ ОЧИСТКА ПРОЕКТА GITLAB-SONARQUBE ==="

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

```bash
#!/bin/bash
# full_reset_docker.sh

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

# 3. Проверка доступности сервисов
echo "3. 🚀 ПРОВЕРКА ДОСТУПНОСТИ СЕРВИСОВ"
echo "----------------------------------------"

check_service() {
    local name=$1
    local url=$2
    
    echo -n "🔍 $name: "
    if curl -s -f "$url" > /dev/null; then
        echo "✅ ДОСТУПЕН"
        return 0
    else
        echo "❌ НЕДОСТУПЕН"
        return 1
    fi
}

check_service "GitLab" "http://gitlab.localdomain"
check_service "SonarQube" "http://sonarqube.localdomain:9000"

echo ""

# 4. Проверка GitLab Runner
echo "4. ⚙️ ПРОВЕРКА GITLAB RUNNER"
echo "----------------------------------------"
if docker ps | grep -q gitlab-runner; then
    echo "✅ GitLab Runner запущен"
    docker-compose exec gitlab-runner gitlab-runner list 2>/dev/null || echo "❌ Runner не зарегистрирован"
else
    echo "❌ GitLab Runner не запущен"
fi
echo ""

# 5. Итоговая проверка
echo "5. 🎯 ИТОГОВАЯ ДИАГНОСТИКА"
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

---

## Устранение неполадок

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

# 8. Финальная проверка
log "Финальная проверка..."
./system-check.sh

echo ""
echo "✅ УСТРАНЕНИЕ НЕПОЛАДОК ЗАВЕРШЕНО"
```

---

## Заключение

### 🎯 Что было улучшено в V7:

1. **Оптимизированные настройки ресурсов** - сбалансированное распределение памяти и CPU для 4-ядерных систем
2. **Полная поддержка веб-технологий** - LESS, CSS, HTML, JavaScript
3. **Улучшенные скрипты автоматизации** - быстрый деплой и диагностика
4. **Расширенный мониторинг** - реальное время и детальная диагностика
5. **Готовые примеры кода** - простая структура веб-проекта
6. **Улучшенная документация** - пошаговые инструкции и устранение неполадок
7. **Четкое расположение файлов** - комментарии где должны находиться критические файлы

### 🚀 Быстрый старт:

```bash
# 1. Клонирование и настройка
cd ~
mkdir gitlab-setup && cd gitlab-setup

# 2. Скачивание скриптов (или создание вручную)
# 3. Запуск автоматического развертывания
chmod +x *.sh
./quick-deploy-4core.sh

# 4. Мониторинг запуска
./wait-for-gitlab.sh

# 5. Проверка системы
./system-check.sh
```

### 📊 Ключевые сервисы:

- **GitLab**: http://gitlab.localdomain
- **SonarQube**: http://sonarqube.localdomain:9000  
- **Документация**: Встроенные скрипты помощи
- **Мониторинг**: ./system-check.sh

### 🔧 Управление:

```bash
cd ~/gitlab-setup
docker-compose start    # Запуск
docker-compose status   # Статус  
docker-compose stop     # Остановка
./troubleshoot.sh       # Устранение неполадок
```

### 📍 ВАЖНЫЕ ЗАМЕЧАНИЯ ПО РАСПОЛОЖЕНИЮ ФАЙЛОВ:

- **`.gitlab-ci.yml`** - ⚠️ ОБЯЗАТЕЛЬНО В КОРНЕ репозитория GitLab проекта
- **`sonar-project.properties`** - ⚠️ ОБЯЗАТЕЛЬНО В КОРНЕ репозитория GitLab проекта
- GitLab автоматически обнаруживает `.gitlab-ci.yml` при пуше в репозиторий
- SonarScanner ищет `sonar-project.properties` в корне проекта

**Ваша система готова для профессиональной веб-разработки с полным циклом CI/CD!** 🎉

---

*Версия V7 - ФИНАЛЬНАЯ - Обновлено с оптимизацией для 4-ядерных систем, улучшенными скриптами и полной документацией*
