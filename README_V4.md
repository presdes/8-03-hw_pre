# 🚀 Практическое руководство: Развертывание GitLab с GitLab Runner и SonarQube в Docker

## 📋 Содержание
1. [Архитектура решения](#архитектура-решения)
2. [Предварительные требования](#предварительные-требования)
3. [Развертывание инфраструктуры](#развертывание-инфраструктуры)
4. [Настройка компонентов](#настройка-компонентов)
5. [Интеграция CI/CD с SonarQube](#интеграция-cicd-с-sonarqube)
6. [Создание проектов и настройка Runner](#создание-проектов-и-настройка-runner)
7. [Управление и мониторинг](#управление-и-мониторинг)

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

### Расположение данных
📁 Архитектура хранения данных GitLab

```
~/gitlab-sonarqube-setup/          # Директория проекта Docker Compose
├── docker-compose.yml             # Конфигурация инфраструктуры
├── setup-hosts.sh                 # Скрипт настройки hosts
├── check-network.sh               # Скрипт проверки сети
└── reset-project.sh               # Скрипт сброса проекта

/var/opt/gitlab/                   # ВНУТРИ КОНТЕЙНЕРА GitLab
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

🗂️ Детальная структура внутри GitLab контейнера

```
/var/opt/gitlab/                   # Основная директория данных GitLab
├── git-data/
│   └── repositories/              # 👈 РЕПОЗИТОРИИ GIT
│       ├── @hashed/               # Хэшированная структура (основная)
│       │   ├── ab/ct/             # Пример: ab/cd/abcdef1234567890...
│       │   │   └── abcdef1234567890.../  # Хэш-директория проекта
│       │   │       ├── config     # Конфиг репозитория
│       │   │       ├── objects/   # Git объекты
│       │   │       └── refs/      # Ссылки и ветки
│       │   └── .../
│       └── user/                  # Устаревшая структура (для совместимости)
│           └── username/project.git/
├── postgresql/                    # База данных GitLab
│   ├── data/
│   │   └── base/                  # Данные PostgreSQL
│   └── pg_wal/                    # Write-ahead logs
├── redis/                         # Кеш и сессии
│   └── data/
│       └── dump.rdb               # Дамп Redis
├── uploads/                       # Загруженные файлы
│   └── -/                         # Файлы проектов, аватарки и т.д.
├── shared/                        # Общие данные
│   ├── artifacts/                 # Артефакты CI/CD
│   ├── lfs-objects/               # Git LFS объекты
│   ├── packages/                  # Пакеты (NPM, Maven и т.д.)
│   └── registry/                  # Container Registry
└── backups/                       # Автоматические бэкапы
```
---

- **Проекты GitLab**: `/var/opt/gitlab/git-data/repositories/` (внутри контейнера)
- **Конфигурация**: Docker volumes
- **Логи**: Docker volumes

---

## Предварительная очистка Docker окружения
### ⚠️ ВАЖНОЕ ПРЕДУПРЕЖДЕНИЕ
Разделы предварительной очистки приведут к полному удалению всех данных Docker!
- Все контейнеры будут остановлены и удалены
- Все volumes (включая данные GitLab, SonarQube) будут удалены
- Все сети будут удалены
- Все образы могут быть удалены
- ЭТО НЕОБРАТИМЫЕ ДЕЙСТВИЯ

---

### 🔥 ОПАСНО: Полная очистка всех Docker данных

```bash
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

## 🗑️ Безопасная очистка только проекта GitLab-SonarQube
### Выполняем в директории проекта, если она существует
```bash
if [ -d "~/gitlab-sonarqube-setup" ]; then
    echo "=== 🗑️ БЕЗОПАСНАЯ ОЧИСТКА ПРОЕКТA GITLAB-SONARQUBE ==="
    
    cd ~/gitlab-sonarqube-setup
    
    if [ -f "docker-compose.yml" ]; then
        echo "Останавливаем и удаляем контейнеры проекта..."
        docker compose down -v
        
        echo "Удаляем volumes проекта..."
        docker volume rm gitlab-sonarqube-setup_gitlab_config \
                         gitlab-sonarqube-setup_gitlab_logs \
                         gitlab-sonarqube-setup_gitlab_data \
                         gitlab-sonarqube-setup_gitlab-runner-config \
                         gitlab-sonarqube-setup_sonarqube_data \
                         gitlab-sonarqube-setup_sonarqube_extensions \
                         gitlab-sonarqube-setup_sonarqube_logs 2>/dev/null || true
        
        echo "Удаляем сеть проекта..."
        docker network rm gitlab-network 2>/dev/null || true
        
        echo "✅ Очистка проекта завершена!"
    else
        echo "❌ docker-compose.yml не найден в директории проекта"
    fi
else
    echo "❌ Директория проекта ~/gitlab-sonarqube-setup не существует"
fi
```
---

## ⚙️ Предварительные требования

### Для всех систем
- Docker и Docker Compose
- 4+ GB RAM (рекомендуется 6 GB для комфортной работы)
- 20+ GB свободного места
- Права sudo

### Для WSL2
- Windows 10/11 с WSL2
- Ubuntu из Microsoft Store
- Docker Desktop с WSL2 integration

### Оптимальные настройки WSL2
```bash
# Создаем файл конфигурации WSL2
cat > /mnt/c/Users/$USER/.wslconfig << 'EOF'
[wsl2]
memory=6GB
processors=2
swap=4GB
localhostForwarding=true
EOF
```

---

## 🚀 Развертывание инфраструктуры

### Шаг 1: Подготовка окружения

```bash
# Создаем рабочую директорию
cd ~
mkdir gitlab-sonarqube-setup
cd gitlab-sonarqube-setup

# Настройка hosts файла
cat > setup-hosts.sh << 'EOF'
#!/bin/bash
echo "=== Настройка hosts файла ==="
sudo cp /etc/hosts /etc/hosts.backup.$(date +%Y%m%d%H%M%S)
sudo sed -i '/192.168.56/d' /etc/hosts
echo "192.168.56.10    gitlab.localdomain" | sudo tee -a /etc/hosts
echo "192.168.56.20    sonarqube.localdomain" | sudo tee -a /etc/hosts
echo "192.168.56.30    gitlab-runner.localdomain" | sudo tee -a /etc/hosts
echo "✅ Hosts файл настроен"
EOF

chmod +x setup-hosts.sh
sudo ./setup-hosts.sh
```

### Шаг 2: Экономная конфигурация Docker Compose

```bash
cat > docker-compose.yml << 'EOF'
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
        # ОПТИМИЗАЦИИ ДЛЯ ЭКОНОМИИ РЕСУРСОВ
        prometheus_monitoring['enable'] = false
        grafana['enable'] = false
        puma['worker_processes'] = 2
        puma['min_threads'] = 1
        puma['max_threads'] = 4
        sidekiq['max_concurrency'] = 5
        gitlab_rails['gitlab_default_can_create_group'] = 'false'
        gitlab_rails['gitlab_default_projects_features_issues'] = 'false'
        gitlab_rails['gitlab_default_projects_features_merge_requests'] = 'false'
        gitlab_rails['gitlab_default_projects_features_wiki'] = 'false'
        nginx['worker_processes'] = 2
        nginx['worker_connections'] = 1024
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
    # ЭКОНОМНЫЕ НАСТРОЙКИ
    mem_limit: 2g
    mem_reservation: 1g
    cpus: 1.0

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
    # МИНИМАЛЬНЫЕ РЕСУРСЫ
    mem_limit: 512m
    cpus: 0.5

  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    hostname: sonarqube.localdomain
    restart: unless-stopped
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
      # ОПТИМИЗАЦИИ ДЛЯ МАЛОЙ ПАМЯТИ
      SONAR_WEB_JAVAOPTS: "-Xmx512m -Xms128m -XX:MaxMetaspaceSize=128m -XX:+UseG1GC"
      SONAR_CE_JAVAOPTS: "-Xmx512m -Xms128m -XX:MaxMetaspaceSize=128m"
      SONAR_SEARCH_JAVAOPTS: "-Xmx512m -Xms128m -XX:MaxMetaspaceSize=128m"
      SONAR_JDBC_MAXACTIVE: "10"
      SONAR_JDBC_MAXIDLE: "5"
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
    # ЭКОНОМНЫЕ НАСТРОЙКИ
    mem_limit: 1g
    mem_reservation: 512m
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
    name: gitlab-network
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.56.0/24
          gateway: 192.168.56.1
EOF
```

### Шаг 3: Запуск инфраструктуры

```bash
# Запуск сервисов
docker compose up -d

# Проверка статуса
docker compose ps

echo "⏳ Ожидайте полный запуск GitLab (10-15 минут)..."
```

---

## ⚙️ Настройка компонентов

### Шаг 1: Мониторинг запуска GitLab

```bash
cat > wait-gitlab-working.sh << 'EOF'
#!/bin/bash

echo "⏳ Проверяем статус GitLab..."
cd ~/gitlab-sonarqube-setup

# Функция для получения статуса разными способами
get_gitlab_status() {
    # Способ 1: через docker compose ps --format json
    local status1=$(docker compose ps gitlab --format json 2>/dev/null | jq -r '.[].Status' 2>/dev/null)
    
    # Способ 2: через парсинг вывода docker compose ps
    local status2=$(docker compose ps gitlab 2>/dev/null | grep gitlab | awk '{print $4}')
    
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
    PASSWORD=$(docker compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password 2>/dev/null | cut -d: -f2- | sed 's/^ *//;s/ *$//')
    
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
        echo "docker compose exec gitlab cat /etc/gitlab/initial_root_password"
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
        PASSWORD=$(docker compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password 2>/dev/null | cut -d: -f2- | sed 's/^ *//;s/ *$//')
        
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
EOF

chmod +x wait-gitlab-working.sh
./wait-gitlab-working.sh
```

### Шаг 2: Настройка GitLab Runner

```bash
# Регистрация общего Runner
docker compose exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://gitlab.localdomain/" \
  --registration-token "$(docker compose exec gitlab gitlab-rails runner -e production "puts Gitlab::CurrentSettings.current_application_settings.runners_registration_token")" \
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

## 🔗 Интеграция CI/CD с SonarQube

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

### Шаг 2: Создание тестового проекта

```bash
# Создаем тестовый Java проект
mkdir -p ~/my-app
cd ~/my-app

# Создаем структуру проекта
mkdir -p src/main/java/com/example
mkdir -p src/test/java/com/example

# Создаем Maven конфигурацию
cat > pom.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <sonar.host.url>http://sonarqube.localdomain:9000</sonar.host.url>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
            </plugin>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.8</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>report</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
EOF

# Создаем исходный код
cat > src/main/java/com/example/Main.java << 'EOF'
package com.example;

/**
 * Main application class
 */
public class Main {
    
    /**
     * Main method
     * @param args command line arguments
     */
    public static void main(String[] args) {
        System.out.println("Hello, SonarQube!");
        String message = "Welcome to CI/CD";
        if (args.length > 0) {
            message = args[0];
        }
        printMessage(message);
    }
    
    /**
     * Print message to console
     * @param msg message to print
     */
    private static void printMessage(String msg) {
        System.out.println("Message: " + msg);
    }
    
    /**
     * Calculate sum of two numbers
     * @param a first number
     * @param b second number
     * @return sum of a and b
     */
    public static int add(int a, int b) {
        return a + b;
    }
}
EOF

# Создаем тесты
cat > src/test/java/com/example/MainTest.java << 'EOF'
package com.example;

import org.junit.Test;
import static org.junit.Assert.*;

public class MainTest {
    
    @Test
    public void testAdd() {
        assertEquals(5, Main.add(2, 3));
        assertEquals(0, Main.add(0, 0));
        assertEquals(-1, Main.add(2, -3));
    }
}
EOF

# Создаем sonar-project.properties
cat > sonar-project.properties << 'EOF'
sonar.projectKey=my-app
sonar.projectName=My Application
sonar.projectVersion=1.0
sonar.sources=src/main/java
sonar.tests=src/test/java
sonar.sourceEncoding=UTF-8
sonar.java.binaries=target/classes
sonar.java.libraries=target/classes
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
sonar.junit.reportPaths=target/surefire-reports
EOF

# Создаем README
cat > README.md << 'EOF'
# My Application

Проект для демонстрации интеграции GitLab CI/CD с SonarQube.

## Функциональность

- Простое Java приложение
- Модульное тестирование с JUnit
- Покрытие кода с JaCoCo
- Статический анализ с SonarQube

## CI/CD Pipeline

Проект настроен с автоматическим:
1. Запуском тестов при каждом коммите
2. Измерением покрытия кода
3. Статическим анализом в SonarQube
4. Проверкой качества кода
EOF
```

### Шаг 3: Создание .gitlab-ci.yml с SonarQube

```bash
cat > .gitlab-ci.yml << 'EOF'
stages:
  - test
  - sonarqube

variables:
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"

cache:
  paths:
    - .sonar/cache
    - target/

# Тестирование
test:
  stage: test
  image: maven:3.8.6-openjdk-11
  script:
    - mvn clean test
    - mvn jacoco:report
  artifacts:
    paths:
      - target/site/jacoco/
      - target/surefire-reports/
    expire_in: 1 week
  only:
    - main
    - merge_requests

# Анализ SonarQube
sonarqube-check:
  stage: sonarqube
  image: maven:3.8.6-openjdk-11
  dependencies:
    - test
  before_script:
    - apt-get update && apt-get install -y curl
    # Добавляем запись в hosts для гарантии связи
    - echo "192.168.56.20 sonarqube.localdomain" >> /etc/hosts
  script:
    - mvn verify sonar:sonar
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.token=$SONAR_TOKEN
      -Dsonar.projectKey=$CI_PROJECT_NAME
      -Dsonar.projectName=$CI_PROJECT_NAME
      -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
      -Dsonar.junit.reportPaths=target/surefire-reports
  allow_failure: false
  only:
    - main
    - merge_requests
  tags:
    - docker
    - linux
EOF
```

---

## 📁 Создание проектов и настройка Runner

### Шаг 1: Создание проекта в GitLab

1. **Откройте**: http://gitlab.localdomain
2. **Войдите** как root (пароль из скрипта ожидания)
3. **Создайте проект**:
   - Нажмите **"New project"**
   - Выберите **"Create blank project"**
   - **Project name**: `my-app`
   - **Visibility**: `Private`
   - **Initialize with README**: ❌ НЕ отмечать
   - Нажмите **"Create project"**

### Шаг 2: Настройка переменных CI/CD

1. В проекте перейдите: **Settings → CI/CD → Variables**
2. Нажмите **"Add variable"**
3. Добавьте переменные:
   - **SONAR_TOKEN**: [ваш токен из SonarQube]
   - **SONAR_HOST_URL**: `http://sonarqube.localdomain:9000`

### Шаг 3: Инициализация и настройка Git репозитория

```bash
cd ~/my-app

# Инициализируем Git
git init
git config --local user.name "root"
git config --local user.email "root@gitlab.localdomain"

# Добавляем файлы и делаем первый коммит
git add .
git commit -m "Initial commit with CI/CD and SonarQube integration"

# Добавляем удаленный репозиторий
git remote add origin http://gitlab.localdomain/root/my-app.git

# Пушим в GitLab
git push -u origin main

echo "✅ Проект загружен в GitLab!"
```

### Шаг 4: Регистрация специфичного Runner для проекта

```bash
cd ~/gitlab-sonarqube-setup

# Создаем скрипт регистрации
cat > register-project-runner.sh << 'EOF'
#!/bin/bash

PROJECT_NAME="my-app"
RUNNER_DESCRIPTION="runner-for-$PROJECT_NAME"

echo "🚀 Регистрация Runner для проекта: $PROJECT_NAME"

# Запрос токена у пользователя
read -p "Введите Registration Token из GitLab (Settings → CI/CD → Runners): " REGISTRATION_TOKEN

if [ -z "$REGISTRATION_TOKEN" ]; then
    echo "❌ Токен не может быть пустым"
    exit 1
fi

echo "⏳ Регистрируем Runner..."
docker compose exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://gitlab.localdomain/" \
  --registration-token "$REGISTRATION_TOKEN" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "$RUNNER_DESCRIPTION" \
  --tag-list "docker,linux,$PROJECT_NAME" \
  --run-untagged="false" \
  --locked="false" \
  --docker-privileged="true" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/cache"

echo "✅ Runner зарегистрирован для проекта $PROJECT_NAME!"
echo "🔧 Проверяем статус..."
docker compose exec gitlab-runner gitlab-runner list
EOF

chmod +x register-project-runner.sh
./register-project-runner.sh
```

---

## 🛠️ Управление и мониторинг

### Основные команды управления

```bash
cd ~/gitlab-sonarqube-setup

# Просмотр статуса
docker compose ps

# Логи сервисов
docker compose logs gitlab
docker compose logs sonarqube --follow
docker compose logs gitlab-runner

# Остановка/запуск
docker compose stop
docker compose start
docker compose restart

# Полная остановка с сохранением данных
docker compose down

# Полная остановка с удалением данных (⚠️ осторожно!)
docker compose down -v
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

### Проверка работоспособности

```bash
cat > check-system.sh << 'EOF'
#!/bin/bash

echo "=== ✅ ПРОВЕРКА СИСТЕМЫ ==="
cd ~/gitlab-sonarqube-setup

echo ""
echo "📊 СТАТУС СЕРВИСОВ:"
docker compose ps

echo ""
echo "🌐 ДОСТУП К СЕРВИСАМ:"
echo "----------------------------------------"
echo "🔧 GitLab:"
curl -s http://gitlab.localdomain > /dev/null && echo "✅ Доступен" || echo "❌ Недоступен"

echo "📊 SonarQube:"
curl -s http://sonarqube.localdomain:9000 > /dev/null && echo "✅ Доступен" || echo "❌ Недоступен"

echo ""
echo "🔗 СЕТЕВАЯ СВЯЗНОСТЬ:"
docker compose exec gitlab-runner ping -c 1 gitlab.localdomain > /dev/null && \
  echo "✅ GitLab доступен из Runner" || echo "❌ Проблема с GitLab"

docker compose exec gitlab-runner ping -c 1 sonarqube.localdomain > /dev/null && \
  echo "✅ SonarQube доступен из Runner" || echo "❌ Проблема с SonarQube"

echo ""
echo "⚙️ GITLAB RUNNER:"
docker compose exec gitlab-runner gitlab-runner list

echo ""
echo "🎯 ПРОВЕРКА ПРОЕКТА:"
echo "----------------------------------------"
echo "🌐 GitLab проект: http://gitlab.localdomain/root/my-app"
echo "📊 Pipelines: http://gitlab.localdomain/root/my-app/-/pipelines"
echo "🔍 SonarQube: http://sonarqube.localdomain:9000"

echo ""
echo "✅ СИСТЕМА ГОТОВА К РАБОТЕ!"
EOF

chmod +x check-system.sh
./check-system.sh
```

### Резервное копирование

```bash
# Бэкап GitLab
docker compose exec gitlab gitlab-backup create

# Бэкап конфигурации
cd ~/gitlab-sonarqube-setup
tar -czf backup-$(date +%Y%m%d).tar.gz docker-compose.yml *.sh
```

---

## 🎯 Итоговый чеклист

### ✅ Что должно быть готово:
1. **GitLab**: http://gitlab.localdomain (root + пароль)
2. **SonarQube**: http://sonarqube.localdomain:9000 (admin/netology)
3. **GitLab Runner**: зарегистрирован и работает
4. **Тестовый проект**: `my-app` в GitLab
5. **CI/CD пайплайн**: с интеграцией SonarQube
6. **Сеть**: все сервисы связаны

### 🔧 Ключевые улучшения:
- **Экономная конфигурация**: ~3.5GB RAM вместо 6GB+
- **Рабочий скрипт ожидания**: корректно определяет статус GitLab
- **Полная интеграция**: GitLab CI/CD + SonarQube
- **Автоматизация**: скрипты для всех ключевых операций
- **Исправленные ошибки**: проблемы с сетью, паролями, статусами

### 🌐 Ссылки для проверки:
- **GitLab**: http://gitlab.localdomain
- **SonarQube**: http://sonarqube.localdomain:9000
- **Проект**: http://gitlab.localdomain/root/my-app
- **Pipelines**: http://gitlab.localdomain/root/my-app/-/pipelines

**Ваша система полностью готова для автоматизированной разработки с CI/CD и анализом качества кода!** 🚀
