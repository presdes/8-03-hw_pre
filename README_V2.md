# Практическое руководство: Развертывание GitLab с GitLab Runner и SonarQube в Docker

Полный аналог Vagrantfile: https://github.com/netology-code/sdvps-materials/blob/main/gitlab/Vagrantfile

## Архитектура решения

```
+-----------------------------------------------------------------------+
| Docker Host                                                           |
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
Особенности для WSL2:
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
---

## Предварительные требования

### Для всех систем:
- Установленные Docker и Docker Compose
- Минимум 6 ГБ оперативной памяти (4 ГБ для GitLab + 2 ГБ для SonarQube)
- Права sudo для настройки hosts файла
- Достаточное дисковое пространство

### Особые требования для WSL2:
- Windows 10 версии 2004 или выше / Windows 11
- Включенная функция WSL2
- Установленный дистрибутив Ubuntu из Microsoft Store
- Docker Desktop с включенной опцией WSL2 integration

---
## Шаг 1: Подготовка окружения
Очистка окружения (опционально)
```bash
# ⚠️ ВНИМАНИЕ: Удаляет все Docker данные!
echo "=== 🔥 ОЧИСТКА DOCKER ==="
read -p "Вы уверены? (yes/NO): " confirm
if [ "$confirm" = "yes" ]; then
    docker stop $(docker ps -aq) 2>/dev/null
    docker rm $(docker ps -aq) 2>/dev/null
    docker volume rm $(docker volume ls -q) 2>/dev/null
    docker system prune -a -f --volumes
fi
```
Создание рабочей директории
```bash
# Выполняем в домашней директории или в выбранной вами location
cd ~
mkdir gitlab-sonarqube-setup
cd gitlab-sonarqube-setup
```
Проверка и настройка WSL2 (только для Windows)
```bash
# В PowerShell от администратора проверяем версию WSL
wsl --list --verbose

# Если WSL1, обновляем до WSL2
wsl --set-version Ubuntu 2

# Убеждаемся, что в Docker Desktop включена интеграция с WSL2
# Settings -> Resources -> WSL Integration -> Enable integration with Ubuntu
```
---
## Шаг 2: Настройка hosts файла

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
echo "192.168.56.30    gitlab-runner.localdomain" | sudo tee -a $TEMP_FILE

# Заменяем оригинальный файл
sudo cp $TEMP_FILE $HOSTS_FILE
sudo rm $TEMP_FILE

echo "Hosts файл успешно настроен!"
echo "============================"
grep "192.168.56" $HOSTS_FILE
```
Выполните настройку hosts файла:
```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup
chmod +x setup-hosts.sh
sudo ./setup-hosts.sh
```
Для Windows (если используете WSL2)
Дополнительно настройте hosts файл Windows (в PowerShell от администратора):
```bash
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "192.168.56.10    gitlab.localdomain"
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "192.168.56.20    sonarqube.localdomain"
```
---

## Шаг 3: Создание docker-compose.yml с фиксированными IP

Создайте `docker-compose.yml`:

```yaml
version: '3.8'

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    hostname: gitlab.localdomain
    restart: always
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.localdomain'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
        prometheus_monitoring['enable'] = false
        # Настройки для работы с фиксированным IP
        gitlab_rails['gitlab_ssh_host'] = '192.168.56.10'
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
      retries: 3

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    depends_on:
      gitlab:
        condition: service_healthy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab-runner-config:/etc/gitlab-runner
    networks:
      gitlab-network:
        ipv4_address: 192.168.56.30
    environment:
      - GODEBUG=netdns=go
    # Дополнительные hosts записи внутри контейнера
    extra_hosts:
      - "gitlab.localdomain:192.168.56.10"
      - "sonarqube.localdomain:192.168.56.20"

  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    hostname: sonarqube.localdomain
    restart: always
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
      # Оптимизации для ограниченных ресурсов
      SONAR_WEB_JAVAOPTS: "-Xmx512m -Xms128m"
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
      retries: 3

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
```

---

## Шаг 4: Запуск и инициализация инфраструктуры

```bash
# Выполняем в директории gitlab-sonarqube-setup (где лежит docker-compose.yml)
cd ~/gitlab-sonarqube-setup

# Проверяем доступность памяти (особенно важно для WSL2)
free -h
docker system df

# Запуск всех сервисов (аналог vagrant up)
docker compose up -d

# Мониторинг запуска
echo "Ожидаем запуск сервисов..."
docker compose logs -f gitlab &
```

---

## Шаг 5: Проверка сетевой конфигурации

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
docker compose exec gitlab-runner ping -c 2 gitlab.localdomain

echo "Из Runner в SonarQube:"
docker compose exec gitlab-runner ping -c 2 sonarqube.localdomain

echo ""
echo "5. Проверка доступности сервисов:"
echo "GitLab: http://gitlab.localdomain"
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://gitlab.localdomain || echo "Недоступен"

echo "SonarQube: http://sonarqube.localdomain:9000"
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://sonarqube.localdomain:9000 || echo "Недоступен"
```

Выполните проверку:
```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup
chmod +x check-network.sh
./check-network.sh
```

---

## Шаг 6: Ожидание инициализации GitLab

```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup

# Ожидание полной инициализации GitLab (аналог ожидания в Vagrantfile)
echo "Ожидаем инициализацию GitLab..."
while true; do
    if docker compose logs gitlab 2>/dev/null | grep -q "gitlab Reconfigured!"; then
        echo "✅ GitLab успешно запущен и сконфигурирован!"
        break
    fi
    sleep 10
    echo "⏳ Ожидаем... (проверка через 10 секунд)"
done

# Получение пароля root (аналог вывода пароля в Vagrant)
echo ""
echo "=== ПАРОЛЬ ROOT ДЛЯ GITLAB ==="
docker compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password || echo "Пароль уже использован, установите новый через интерфейс"
echo "=============================="
```
Скрипт ожидания доступности GitLab wait-gitlab-working.sh
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
```
Быстрая роверка здоровья системы (GitLab)
```bash
echo "=== 🔍 ДИАГНОСТИКА СИСТЕМЫ ==="
cd ~/gitlab-sonarqube-setup

# Проверяем статус
docker compose ps

# Проверяем логи
echo "Последние логи ошибок:"
docker compose logs gitlab --tail=20 | grep -i error

# Проверяем доступность
curl -I http://gitlab.localdomain
```
---

## Шаг 7: Настройка GitLab Runner

```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup

# Ожидаем полную готовность GitLab API
sleep 30

echo "Регистрируем GitLab Runner..."

# Регистрация Runner с настройками аналогичными Vagrantfile
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

echo "✅ GitLab Runner успешно зарегистрирован"
```

---

## Шаг 8: Настройка SonarQube

После запуска выполните:

1. **Откройте SonarQube:** http://sonarqube.localdomain:9000
   - Логин: `admin`
   - Пароль: `admin`

2. **Смена пароля** (обязательно):
   - При первом входе смените пароль

3. **Создание токена для GitLab:**
Выполняем в любой директории - это просто инструкция для пользователя
Для настройки SonarQube:
1. Войдите в http://sonarqube.localdomain:9000 (admin/[ваш пароль])
2. Перейдите: My Account → Security → Generate Token
3. Название: 'gitlab-token', сохраните значение!
---

## Шаг 9: Настройка проекта в GitLab

### 1. Создайте проект в GitLab:
- Откройте http://gitlab.localdomain
- Войдите как root (пароль из шага 6)
- Создайте новый проект или импортируйте существующий

### 2. Добавьте переменные CI/CD в GitLab:
- Перейдите в **Settings → CI/CD → Variables → Expand**
- Добавьте переменные:
  - `SONAR_TOKEN` = [ваш токен из SonarQube]
  - `SONAR_HOST_URL` = `http://sonarqube.localdomain:9000`

### 3. Создайте файл .gitlab-ci.yml в корне вашего проекта:

```yaml
stages:
  - test
  - sonarqube

sonarqube-check:
  stage: sonarqube
  image: 
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    keys:
      - "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  before_script:
    # Добавляем запись в hosts внутри контейнера для гарантии
    - echo "192.168.56.20 sonarqube.localdomain" >> /etc/hosts
  script:
    - sonar-scanner
      -Dsonar.projectKey=${CI_PROJECT_NAME}
      -Dsonar.projectName=${CI_PROJECT_NAME}
      -Dsonar.host.url=${SONAR_HOST_URL}
      -Dsonar.token=${SONAR_TOKEN}
      -Dsonar.projectVersion=${CI_COMMIT_SHORT_SHA}
      -Dsonar.sources=.
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```
---
## 🎯 Что делать дальше:
1. Войдите в GitLab и смените пароль root
2. Настройте SonarQube (смена пароля, создание токена)
3. Создайте проект в GitLab
4. Настройте CI/CD с переменными SonarQube
5. Протестируйте пайплайн с анализом кода

Поздравляю с успешным развертыванием! Ваша система полностью готова для разработки с CI/CD и анализом качества кода. 🚀

---

## Шаг 10: Полезные команды управления

```bash
# Все команды выполняются в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup

# Просмотр статуса всех сервисов
docker compose ps

# Просмотр логов
docker compose logs gitlab
docker compose logs sonarqube
docker compose logs gitlab-runner

# Остановка всех сервисов
docker compose stop

# Полная остановка с удалением контейнеров
docker compose down

# Перезапуск
docker compose restart

# Обновление и перезапуск
docker compose pull && docker compose up -d

# Проверка здоровья сервисов
docker compose ps --filter "status=running"
```

---

## Шаг 11: Проверка работоспособности

```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup

# Финальная проверка
echo "=== ФИНАЛЬНАЯ ПРОВЕРКА РАБОТОСПОСОБНОСТИ ==="

echo "1. Сервисы доступны по доменным именам:"
echo "   GitLab:       http://gitlab.localdomain"
echo "   SonarQube:    http://sonarqube.localdomain:9000"

echo "2. Сеть настроена корректно:"
docker compose exec gitlab-runner curl -s http://sonarqube.localdomain:9000/api/system/status | grep -q "UP" && echo "   ✅ SonarQube доступен из Runner" || echo "   ❌ SonarQube недоступен из Runner"

echo "3. GitLab Runner зарегистрирован:"
docker compose exec gitlab-runner gitlab-runner list && echo "   ✅ Runner активен" || echo "   ❌ Проблемы с Runner"

echo "=== НАСТРОЙКА ЗАВЕРШЕНА ==="
```

---

## Решение проблем

### Если сервисы недоступны по доменным именам:
```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup

# Перезапускаем сервисы
docker compose restart

# Проверяем hosts файл
sudo ./setup-hosts.sh

# Проверяем сеть
./check-network.sh
```

### Если Runner не может подключиться к SonarQube:
```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup

# Проверяем связность из контейнера Runner
docker compose exec gitlab-runner ping sonarqube.localdomain
docker compose exec gitlab-runner curl -v http://sonarqube.localdomain:9000
```

### Проблемы с памятью:
```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup

# Останавливаем сервисы
docker compose stop

# Очищаем ресурсы Docker
docker system prune -f

# Запускаем заново
docker compose up -d
```

### Для полного удаления и переустановки:
```bash
# Выполняем в директории gitlab-sonarqube-setup
cd ~/gitlab-sonarqube-setup

# Полное удаление
docker compose down -v
sudo rm -rf ./*

# Возвращаемся на шаг 1
cd ~
rm -rf gitlab-sonarqube-setup
```

---

## Важные примечания

- **Все Docker команды** выполняются в директории `~/gitlab-sonarqube-setup` (где лежит `docker-compose.yml`)
- **Скрипты настройки** (`setup-hosts.sh`, `check-network.sh`) выполняются из той же директории
- **Команды GitLab CI/CD** настраиваются в интерфейсе GitLab или в файлах проекта
- **Первоначальная настройка** выполняется из любой директории, но основная работа - из `~/gitlab-sonarqube-setup`

Теперь ваша инфраструктура полностью развернута с фиксированными IP-адресами и доменными именами, полностью соответствующая оригинальной конфигурации Vagrantfile!
