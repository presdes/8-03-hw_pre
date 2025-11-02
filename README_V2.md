Отличное замечание! Проанализировал Vagrantfile и внесу соответствующие изменения, чтобы наше руководство было максимально похоже по структуре и функциональности.

## Исправленное руководство по развертыванию GitLab с GitLab Runner и SonarQube в Docker

```bash
# Практическое руководство: Развертывание GitLab с GitLab Runner и SonarQube в Docker
# Аналог Vagrantfile: https://github.com/netology-code/sdvps-materials/blob/main/gitlab/Vagrantfile
```

### Архитектура решения после исправлений:

```
+-----------------------------------------------------------------------+
| Docker Host (аналог Vagrant VM)                                       |
|                                                                       |
|  +----------------+      +------------------+      +---------------+  |
|  | GitLab         |      | GitLab Runner    |      | SonarQube     |  |
|  | (gitlab)       |      | (gitlab-runner)  |      | (sonarqube)   |  |
|  | Ports: 80, 443 |      |                  |      | Port: 9000    |  |
|  +----------------+      +------------------+      +---------------+  |
|                                                                       |
|                     Docker Network: gitlab-network                    |
+-----------------------------------------------------------------------+
```

---

## Шаг 1: Подготовка окружения и создание docker-compose.yml

```bash
mkdir gitlab-sonarqube-setup
cd gitlab-sonarqube-setup
```

Создайте `docker-compose.yml`:

```yaml
version: '3.8'

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    hostname: gitlab
    restart: always
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.localdomain'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
        # Отключаем встроенный мониторинг для экономии ресурсов (аналог Vagrantfile)
        prometheus_monitoring['enable'] = false
    ports:
      - "80:80"
      - "443:443"
      - "2224:22"
    volumes:
      - gitlab_config:/etc/gitlab
      - gitlab_logs:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab
    networks:
      - gitlab-network
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
      - gitlab
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab-runner-config:/etc/gitlab-runner
    networks:
      - gitlab-network
    environment:
      - GODEBUG=netdns=go

  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    hostname: sonarqube.localdomain
    restart: always
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
      # Оптимизации для ограниченных ресурсов (аналог настроек Vagrant)
      SONAR_WEB_JAVAOPTS: "-Xmx512m -Xms128m"
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    networks:
      - gitlab-network
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
```

---

## Шаг 2: Запуск и инициализация инфраструктуры

```bash
# Запуск всех сервисов (аналог vagrant up)
docker compose up -d

# Мониторинг запуска
docker compose logs -f gitlab &

# Ожидание полной инициализации GitLab (аналог ожидания в Vagrantfile)
echo "Ожидаем инициализацию GitLab..."
while true; do
    if docker compose logs gitlab 2>/dev/null | grep -q "gitlab Reconfigured!"; then
        echo "GitLab успешно запущен!"
        break
    fi
    sleep 10
    echo "Ожидаем... (проверка через 10 секунд)"
done

# Получение пароля root (аналог вывода пароля в Vagrant)
echo "=== ПАРОЛЬ ROOT ДЛЯ GITLAB ==="
docker compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password || echo "Пароль уже использован, установите новый через интерфейс"
echo "=============================="
```

---

## Шаг 3: Настройка GitLab Runner (аналог provisioning в Vagrant)

```bash
# Ожидаем полную готовность GitLab API
sleep 30

# Регистрация Runner с настройками аналогичными Vagrantfile
docker compose exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://gitlab/" \
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

echo "GitLab Runner зарегистрирован и готов к работе"
```

---

## Шаг 4: Настройка SonarQube (аналог ручной настройки в Vagrant)

После запуска выполните:

1. **Откройте SonarQube:** http://localhost:9000
   - Логин: `admin`
   - Пароль: `admin`

2. **Смена пароля** (обязательно, как в инструкции Vagrant):
   - При первом входе смените пароль на `netology`

3. **Создание токена для GitLab:**
   ```bash
   # Инструкция для создания токена (выполняется в UI SonarQube)
   echo "Для настройки SonarQube:"
   echo "1. Войдите в http://localhost:9000 (admin/netology)"
   echo "2. Перейдите: My Account -> Security -> Generate Token"
   echo "3. Название: 'gitlab-token', скопируйте значение"
   echo "4. Добавьте токен как переменную SONAR_TOKEN в GitLab CI/CD"
   ```

---

## Шаг 5: Создание тестового проекта и настройка CI/CD

### 1. Создайте новый проект в GitLab:
```bash
echo "Создайте проект в GitLab: http://localhost"
echo "Или используйте существующий репозиторий"
```

### 2. Добавьте переменные CI/CD в GitLab:
- **Settings → CI/CD → Variables → Expand**
- `SONAR_TOKEN` = [ваш токен из SonarQube]
- `SONAR_HOST_URL` = `http://sonarqube.localdomain:9000`

### 3. Пример .gitlab-ci.yml для Java проекта:
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
  script:
    - sonar-scanner
      -Dsonar.projectKey=${CI_PROJECT_NAME}
      -Dsonar.projectName=${CI_PROJECT_NAME}
      -Dsonar.host.url=${SONAR_HOST_URL}
      -Dsonar.token=${SONAR_TOKEN}
      -Dsonar.projectVersion=${CI_COMMIT_SHORT_SHA}
      -Dsonar.sources=.
      -Dsonar.java.binaries=target/classes
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

---

## Шаг 6: Полезные команды для управления (аналог vagrant commands)

```bash
# Просмотр статуса всех сервисов
docker compose ps

# Просмотр логов конкретного сервиса
docker compose logs gitlab
docker compose logs sonarqube
docker compose logs gitlab-runner

# Остановка всех сервисов (аналог vagrant halt)
docker compose stop

# Полная остановка с удалением контейнеров (аналог vagrant destroy)
docker compose down

# Перезапуск конкретного сервиса
docker compose restart gitlab

# Вход в контейнер для отладки
docker compose exec gitlab bash
docker compose exec sonarqube bash
```

---

## Шаг 7: Проверка работоспособности

```bash
# Проверка доступности сервисов
echo "Проверка доступности сервисов:"

echo -n "GitLab: "
curl -s -o /dev/null -w "%{http_code}" http://localhost || echo " недоступен"

echo -n "SonarQube: "
curl -s -o /dev/null -w "%{http_code}" http://localhost:9000 || echo " недоступен"

# Проверка сети между контейнерами
echo "Проверка связи между контейнерами:"
docker compose exec gitlab-runner ping -c 2 sonarqube.localdomain

echo "=== СЕТЕВЫЕ НАСТРОЙКИ ==="
docker network inspect gitlab-network --format '{{range .Containers}}{{.Name}} - {{.IPv4Address}}{{"\n"}}{{end}}'
```

---

## Решение распространенных проблем

### 1. Проблемы с памятью (аналогично настройкам в Vagrantfile):
```bash
# Если контейнеры падают из-за нехватки памяти:
docker compose stop
sudo systemctl restart docker
docker compose up -d
```

### 2. Проблемы с регистрацией Runner:
```bash
# Получить токен вручную:
docker compose exec gitlab gitlab-rails runner -e production "puts Gitlab::CurrentSettings.current_application_settings.runners_registration_token"
```

### 3. SonarQube не запускается:
```bash
# Проверить логи
docker compose logs sonarqube

# Частая проблема - недостаточно памяти для Elasticsearch
export SONARQUBE_JAVA_OPTS="-Xmx512m -Xms128m"
docker compose up -d sonarqube
```

Это руководство теперь максимально приближено к логике оригинального Vagrantfile, но реализовано на Docker Compose с учетом всех особенностей сетевого взаимодействия и интеграции между компонентами.
