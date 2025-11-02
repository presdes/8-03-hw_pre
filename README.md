# Руководство по развертыванию GitLab с GitLab Runner и SonarQube в Docker
`Домашнее задание к занятию "GitLab" - Моторин Алексей`

Мы выбрали **WSL2 + Docker** как современную и эффективную альтернативу Vagrant по ключевым причинам:

- Производительность**: Прямая работа с контейнерами без накладных расходов виртуальной машины
- Быстрый запуск**: Контейнеры запускаются за секунды вместо минут у Vagrant
- Актуальные практики**: Docker стал industry standard для CI/CD и контейнеризации
- Простота настройки**: Единый docker-compose.yml вместо многоуровневой конфигурации
- Сетевая интеграция**: Лёгкая настройка доступа к GitLab и SonarQube по доменным именам

Этот подход отражает реальные production-среды 2025 года и обеспечивает лучший опыт для обучения современным DevOps-практикам.

[ⓘ Подробное обоснование почему мы выбрали WSL2 + Docker вместо Vagrant](https://github.com/presdes/8-03-hw_pre/blob/main/WSL2_Docker_VS_Vagrant.md "Включает сравнение по производительности, безопасности и проблеме двойной виртуализации")

---

## Содержание
1. [Предварительные требования](#предварительные-требования)
2. [Развертывание GitLab и SonarQube](#развертывание-gitlab-и-sonarqube)
3. [Создание проекта и настройка GitLab Runner](#создание-проекта-и-настройка-gitlab-runner)
4. [Создание тестового проекта](#создание-тестового-проекта)
5. [Настройка CI/CD пайплайнов](#настройка-cicd-пайплайнов)
6. [Проверка работоспособности](#проверка-работоспособности)
7. [Управление и мониторинг Runner](#управление-и-мониторинг-runner)
8. [Основные определения и концепции](#основные-определения-и-концепции)
9. [Устранение неисправностей](#устранение-неисправностей)

## Предварительные требования

### Системные требования
- **WSL2** с Ubuntu 20.04+
- **Docker** и **Docker Compose**
- Минимум **4GB RAM** для WSL2
- Доступ к терминалу WSL2 с правами администратора

### Проверка окружения
```bash
# Проверка версии WSL
wsl --list --verbose

# Проверка Docker
docker --version
docker-compose --version

# Проверка доступной памяти
free -h
```

## Развертывание GitLab и SonarQube

### Шаг 1.1: Создание рабочей директории
```bash
mkdir -p ~/gitlab-setup
cd ~/gitlab-setup
```

### Шаг 1.2: Создание docker-compose.yml
```bash
cat > docker-compose.yml << 'EOF'
version: '3.6'

services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab
    hostname: 'gitlab.localdomain'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.localdomain'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    ports:
      - '80:80'
      - '443:443'
      - '2222:22'
    volumes:
      - 'gitlab_config:/etc/gitlab'
      - 'gitlab_logs:/var/log/gitlab'
      - 'gitlab_data:/var/opt/gitlab'
    restart: always
    networks:
      - gitlab-network

  sonarqube:
    image: sonarqube:9.9.1-community
    container_name: sonarqube
    hostname: 'sonarqube'
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: 'true'
    ports:
      - '9000:9000'
    volumes:
      - 'sonarqube_data:/opt/sonarqube/data'
      - 'sonarqube_logs:/opt/sonarqube/logs'
      - 'sonarqube_extensions:/opt/sonarqube/extensions'
    restart: always
    networks:
      - gitlab-network

volumes:
  gitlab_config:
  gitlab_logs:
  gitlab_data:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_extensions:

networks:
  gitlab-network:
    driver: bridge
EOF
```

### Шаг 1.3: Запуск сервисов
```bash
# Запуск контейнеров
docker-compose up -d

# Проверка статуса
docker-compose ps

# Мониторинг логов инициализации
docker-compose logs -f gitlab
```

### Шаг 1.4: Настройка hosts файла
```bash
# В WSL2
echo "127.0.0.1 gitlab.localdomain" | sudo tee -a /etc/hosts
echo "127.0.0.1 sonarqube.localdomain" | sudo tee -a /etc/hosts

# В Windows (PowerShell от администратора)
# Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 gitlab.localdomain"
# Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 sonarqube.localdomain"
```

### Шаг 1.5: Получение пароля root и настройка SonarQube
```bash
# Ожидание полной инициализации GitLab (5-10 минут)
while true; do
    if curl -s http://gitlab.localdomain > /dev/null; then
        echo "GitLab готов к работе!"
        break
    else
        echo "Ожидаем инициализации GitLab..."
        sleep 30
    fi
done

# Получение пароля root
docker exec gitlab grep 'Password:' /etc/gitlab/initial_root_password

# Ожидание запуска SonarQube (3-5 минут)
while true; do
    if curl -s http://sonarqube.localdomain:9000 > /dev/null; then
        echo "SonarQube готов к работе!"
        break
    else
        echo "Ожидаем инициализации SonarQube..."
        sleep 30
    fi
done
```

### Шаг 1.6: Настройка SonarQube
1. **Откройте в браузере:** `http://sonarqube.localdomain:9000`
2. **Войдите с учетными данными:**
   - Логин: `admin`
   - Пароль: `admin`
3. **Смените пароль** и сохраните его
4. **Создайте токен:**
   - My Account → Security → Generate Tokens
   - Название: `gitlab-ci-token`
   - **Скопируйте токен** (он покажется только один раз!)

## Создание проекта и настройка GitLab Runner

### Шаг 2.1: Создание проекта в GitLab
1. **Откройте:** `http://gitlab.localdomain`
2. **Войдите с:**
   - Логин: `root`
   - Пароль: (из шага 1.5)
3. **Создайте проект:**
   - Нажмите "Create a project"
   - Выберите "Create blank project"
   - Project name: `netology-gitlab`
   - Visibility Level: `Private`
   - Нажмите "Create project"

### Шаг 2.2: Настройка переменных в GitLab
1. **В проекте перейдите:** Settings → CI/CD → Variables
2. **Добавьте новую переменную:**
   - Key: `SONAR_TOKEN`
   - Value: (ваш токен из SonarQube)
   - Type: `Variable`
   - Flags: ✅ `Mask variable` ✅ `Protect variable`

### Шаг 2.3: Установка GitLab Runner в Docker
```bash
# Создаем директорию для конфигурации
sudo mkdir -p /srv/gitlab-runner/config

**Важно:** Эта команда создает директорию в корневой файловой системе (`/srv/gitlab-runner/config`), а не в домашней директории.

# Получаем регистрационный токен из GitLab:
# Project → Settings → CI/CD → Runners → Registration token

# Регистрируем Runner
docker run -ti --rm --name gitlab-runner \
  --network host \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest register
```

**Ответьте на вопросы при регистрации:**
```
Enter the GitLab instance URL: http://gitlab.localdomain/
Enter the registration token: (ваш токен из GitLab)
Enter a description for the runner: netology-runner
Enter tags for the runner: docker,linux
Enter optional maintenance note: (нажмите Enter)
Enter an executor: docker
Enter the default Docker image: alpine:latest
```

### Шаг 2.4: Настройка конфигурации Runner
```bash
# Редактируем конфигурационный файл
sudo nano /srv/gitlab-runner/config/config.toml
```

**Убедитесь, что конфигурация содержит:**
```toml
concurrent = 1
check_interval = 0

[session_server]
  listen_address = ":8093"

[[runners]]
  name = "netology-runner"
  url = "http://gitlab.localdomain/"
  token = "YOUR_RUNNER_TOKEN"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
    extra_hosts = ["gitlab.localdomain:host-gateway", "sonarqube.localdomain:host-gateway"]
```

### Шаг 2.5: Запуск Runner
```bash
docker run -d --name gitlab-runner --restart always \
  --network host \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

## Создание тестового проекта

### Шаг 3.1: Клонирование и настройка проекта
```bash
# Создаем рабочую директорию
mkdir -p ~/projects/netology
cd ~/projects/netology

# Клонируем проект
git clone http://gitlab.localdomain/root/netology-gitlab.git
cd netology-gitlab

# Настраиваем Git
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### Шаг 3.2: Создание тестового Go приложения
**Примечание:** Выполняйте команды в каталоге склонированного проекта `netology-gitlab/`
```bash
# Создаем структуру проекта
mkdir -p src

# Основной файл Go приложения
cat > src/main.go << 'EOF'
package main

import "fmt"

func main() {
    fmt.Println("Hello Netology!")
    result := add(2, 3)
    fmt.Printf("2 + 3 = %d\n", result)
}

func add(a, b int) int {
    return a + b
}
EOF

# Тесты для приложения
cat > src/main_test.go << 'EOF'
package main

import "testing"

func TestAdd(t *testing.T) {
    result := add(2, 3)
    expected := 5
    if result != expected {
        t.Errorf("add(2, 3) = %d; want %d", result, expected)
    }
}
EOF

# Файл зависимостей Go
cat > go.mod << 'EOF'
module netology-gitlab

go 1.16
EOF
```

### Шаг 3.3: Создание Dockerfile
```bash
cat > Dockerfile << 'EOF'
FROM golang:1.16-alpine

WORKDIR /app

COPY go.mod ./
COPY src/ ./src/

RUN go build -o /netology-app ./src

CMD ["/netology-app"]
EOF
```

### Шаг 3.4: Создание конфигурации SonarQube
```bash
cat > sonar-project.properties << 'EOF'
sonar.projectKey=netology-gitlab
sonar.projectName=Netology GitLab Project
sonar.projectVersion=1.0
sonar.sources=src
sonar.sourceEncoding=UTF-8
sonar.host.url=http://sonarqube.localdomain:9000
sonar.login=${SONAR_TOKEN}
EOF
```

## Настройка CI/CD пайплайнов

### Шаг 4.1: Pipeline с Sonarqube-check
```bash
cat > .gitlab-ci.yml << 'EOF'
stages:
  - test
  - static-analysis
  - build

test:
  stage: test
  image: golang:1.16
  script: 
    - cd src
    - go test .
  tags:
    - docker

static-analysis:
  stage: static-analysis
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  script:
    - sonar-scanner 
      -Dsonar.projectKey=netology-gitlab 
      -Dsonar.sources=src 
      -Dsonar.host.url=http://sonarqube.localdomain:9000 
      -Dsonar.login=$SONAR_TOKEN
  allow_failure: true
  tags:
    - docker

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t netology-app .
    - docker images
  tags:
    - docker
EOF
```

### Шаг 4.2: Pipeline с ручным запуском (альтернатива)
```bash
# Для использования ручного запуска создайте альтернативный файл
cat > .gitlab-ci-manual.yml << 'EOF'
stages:
  - test
  - build

test:
  stage: test
  image: golang:1.16
  script: 
    - cd src
    - go test .
  tags:
    - docker

sonarqube-check:
  stage: test
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  script:
    - sonar-scanner 
      -Dsonar.projectKey=netology-gitlab 
      -Dsonar.sources=src 
      -Dsonar.host.url=http://sonarqube.localdomain:9000
      -Dsonar.login=$SONAR_TOKEN
  allow_failure: true
  tags:
    - docker

# Автоматическая сборка только для main ветки
build-auto:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  only:
    - main
  script:
    - docker build -t netology-app:latest .
    - echo "Автоматическая сборка завершена"
  tags:
    - docker

# Ручная сборка для всех веток кроме main
build-manual:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  when: manual
  except:
    - main
  script:
    - docker build -t netology-app:manual .
    - echo "Ручная сборка завершена"
  tags:
    - docker
EOF
```

### Шаг 4.3: Запуск пайплайна
```bash
# Добавляем файлы в репозиторий
git add .

# Создаем первый коммит
git commit -m "Initial commit with CI/CD pipelines and SonarQube integration"

# Отправляем изменения в GitLab
git push -u origin main
```

## Проверка работоспособности

### Шаг 5.1: Проверка в веб-интерфейсе GitLab
1. **Откройте проект:** `http://gitlab.localdomain/root/netology-gitlab`
2. **Перейдите:** CI/CD → Pipelines
3. **Убедитесь, что:**
   - Пайплайн запустился автоматически
   - Все stages (test, static-analysis, build) выполнены успешно
   - Статус пайплайна: **passed**

### Шаг 5.2: Проверка SonarQube анализа
1. **Откройте:** `http://sonarqube.localdomain:9000`
2. **Найдите проект:** "Netology GitLab Project"
3. **Убедитесь, что:**
   - Анализ кода выполнен
   - Показаны метрики качества кода
   - Нет критических нарушений

### Шаг 5.3: Проверка настроек Runner
1. **В GitLab перейдите:** Settings → CI/CD → Runners
2. **Убедитесь, что:**
   - Runner отображается в списке
   - Статус: **Online** (зеленая точка)
   - Тэги: `docker, linux`
   - Видны последние задачи

### Шаг 5.4: Команды проверки через терминал
```bash
# Проверка статуса Runner контейнера
docker ps -f name=gitlab-runner

# Проверка регистрации Runner
docker exec gitlab-runner gitlab-runner list

# Проверка доступности сервисов из Runner
docker exec gitlab-runner ping -c 2 gitlab.localdomain
docker exec gitlab-runner ping -c 2 sonarqube.localdomain

# Просмотр логов Runner
docker logs gitlab-runner --tail 20
```

## Управление и мониторинг Runner

### Команды для просмотра настроек Runner
```bash
# Просмотр полной конфигурации
sudo cat /srv/gitlab-runner/config/config.toml

# Просмотр только настроек проекта
sudo grep -A 20 "netology-runner" /srv/gitlab-runner/config/config.toml

# Проверка валидности конфигурации
docker exec gitlab-runner gitlab-runner verify

# Просмотр зарегистрированных runners
docker exec gitlab-runner gitlab-runner list
```

### Мониторинг работы Runner
```bash
# Просмотр логов в реальном времени
docker logs gitlab-runner -f

# Проверка использования ресурсов
docker stats gitlab-runner

# Просмотр выполненных задач
docker logs gitlab-runner | grep -i "job" | tail -10
```

### Диагностика сети Runner
```bash
# Проверка разрешения имен
docker exec gitlab-runner nslookup gitlab.localdomain
docker exec gitlab-runner nslookup sonarqube.localdomain

# Проверка сетевых подключений
docker exec gitlab-runner netstat -tulpn

# Проверка доступности Docker
docker exec gitlab-runner docker version
```

### Управление Runner контейнером
```bash
# Перезапуск Runner
docker restart gitlab-runner

# Остановка Runner
docker stop gitlab-runner

# Запуск Runner
docker start gitlab-runner

# Просмотр детальной информации
docker inspect gitlab-runner
```

## Основные определения и концепции

### GitLab
**GitLab** - это полнофункциональная платформа DevOps, которая предоставляет:
- **Git-репозитории** - хранение и управление исходным кодом
- **CI/CD** - встроенные инструменты непрерывной интеграции и доставки
- **Issue Tracking** - система отслеживания задач
- **Container Registry** - встроенный registry для Docker-образов
- **Code Review** - инструменты для проверки кода

### GitLab CI (Continuous Integration)
**GitLab CI** - встроенная система непрерывной интеграции, состоящая из:

- **Pipelines** - цепочки этапов, выполняемых при изменениях кода
- **Stages** - группы jobs (test, build, deploy)
- **Jobs** - отдельные задачи (unit-test, docker-build)
- **Runners** - сервисы, выполняющие jobs

```yaml
# Пример структуры пайплайна
stages:
  - test          # Stage 1
  - build         # Stage 2
  - deploy        # Stage 3

unit-test:        # Job 1
  stage: test
  script: go test .

docker-build:     # Job 2  
  stage: build
  script: docker build .
```

### Dind (Docker-in-Docker)
**Docker-in-Docker** - технология запуска Docker внутри Docker-контейнеров:

**Применение в GitLab CI:**
- Сборка Docker-образов внутри CI/CD пайплайнов
- Запуск контейнеров для тестирования
- Создание многоэтапных сборок

**Архитектура:**
```
GitLab Runner Container
    │
    └── Job Container (dind)
        │
        └── Docker Daemon
            │
            └── Build/test containers
```

### Настройка пайплайна (.gitlab-ci.yml)
**.gitlab-ci.yml** - YAML-файл конфигурации CI/CD пайплайнов:

**Основные секции:**
```yaml
# 1. Stages - последовательность выполнения
stages:
  - test
  - build
  - deploy

# 2. Variables - глобальные переменные
variables:
  DOCKER_TLS_CERTDIR: ""

# 3. Jobs - задачи с условиями выполнения
unit-test:
  stage: test
  image: golang:1.16
  script:
    - go test ./...
  only:
    - main
  tags:
    - docker
```

**Ключевые директивы:**
- `script` - выполняемые shell-команды
- `image` - Docker-образ для job
- `services` - дополнительные сервисы (базы данных, dind)
- `artifacts` - файлы, сохраняемые между jobs
- `cache` - кэширование зависимостей
- `rules` - условное выполнение jobs
- `when` - когда запускать (on_success, manual, always)

### SonarQube и SonarScanner
**SonarQube** - платформа для непрерывного анализа качества кода:

- **Статический анализ** - проверка кода без выполнения
- **Метрики качества** - покрытие тестами, сложность, дублирование
- **Выявление уязвимостей** - security issues
- **Поддержка языков** - Java, Go, Python, JavaScript и др.

**SonarScanner** - инструмент для запуска анализа из CI/CD:

```yaml
sonarqube-check:
  image: sonarsource/sonar-scanner-cli
  variables:
    SONAR_HOST_URL: "http://sonarqube.localdomain:9000"
  script:
    - sonar-scanner -Dsonar.host.url=http://sonarqube.localdomain:9000
      -Dsonar.projectKey=my-project
      -Dsonar.sources=src
      -Dsonar.host.url=http://sonarqube.localdomain:9000
      -Dsonar.login=$SONAR_TOKEN
```

**Интеграция с GitLab:**
1. **SonarQube Server** - отдельный сервер анализа
2. **GitLab CI Job** - запускает SonarScanner
3. **Security Token** - аутентификация между системами
4. **Quality Gates** - автоматическая проверка качества

## Устранение неисправностей

### Проблема: SonarQube недоступен из job-контейнеров
**Решение:**
```bash
# Проверка и обновление extra_hosts в конфиге Runner
sudo nano /srv/gitlab-runner/config/config.toml

# Убедитесь, что есть строка:
extra_hosts = ["gitlab.localdomain:host-gateway", "sonarqube.localdomain:host-gateway"]

# Перезапуск Runner
docker restart gitlab-runner
```

### Проблема: Docker build fails in pipeline
**Решение:**
```yaml
# Убедитесь, что в job есть services: docker:dind
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind  # Эта строка обязательна
  script:
    - docker build .
```

### Проблема: Runner не принимает jobs
**Решение:**
- Проверьте тэги в .gitlab-ci.yml и настройках Runner
- Убедитесь, что Runner имеет статус "Online" в GitLab
- Проверьте регистрацию Runner:
  ```bash
  docker exec gitlab-runner gitlab-runner list
  ```

### Проблема: SonarScanner не находит файлы
**Решение:**
- Проверьте путь в `sonar.sources` (должен быть относительным)
- Убедитесь, что файлы существуют в репозитории
- Проверьте структуру проекта:
  ```bash
  tree ~/projects/netology/netology-gitlab/
  ```

### Проблема: Недостаточно памяти
**Решение:**
```bash
# Увеличьте лимиты памяти в WSL2
# Создайте файл: C:\Users\[Username]\.wslconfig
```
```ini
[wsl2]
memory=4GB
processors=2
swap=2GB
```

### Полезные команды диагностики
```bash
# Функция для быстрой проверки состояния
runner-status() {
    echo "=== Runner Container Status ==="
    docker ps -f name=gitlab-runner --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
    
    echo -e "\n=== Runner Registration ==="
    docker exec gitlab-runner gitlab-runner list 2>/dev/null || echo "Runner not responding"
    
    echo -e "\n=== Service Access ==="
    echo "GitLab: $(curl -s -o /dev/null -w "%{http_code}" http://gitlab.localdomain)"
    echo "SonarQube: $(curl -s -o /dev/null -w "%{http_code}" http://sonarqube.localdomain:9000)"
}

# Добавьте в ~/.bashrc для постоянного использования
```

### Финальная проверка интеграции
```bash
#!/bin/bash
echo "=== Final Integration Check ==="
echo "1. GitLab Access: $(curl -s -o /dev/null -w "%{http_code}" http://gitlab.localdomain)"
echo "2. SonarQube Access: $(curl -s -o /dev/null -w "%{http_code}" http://sonarqube.localdomain:9000)"
echo "3. Runner Status: $(docker ps | grep gitlab-runner | wc -l)"
echo "4. Pipeline Status: Check in GitLab UI"
echo "5. SonarQube Analysis: Check in SonarQube UI"
echo "=== Check Complete ==="
```

---

## ✅ Чек-лист успешной настройки

После выполнения всех шагов убедитесь, что:

- [ ] **GitLab** доступен по `http://gitlab.localdomain`
- [ ] **SonarQube** доступен по `http://sonarqube.localdomain:9000`
- [ ] **Runner** отображается в проекте со статусом **Online**
- [ ] **Пайплайн** запускается при пуше в репозиторий
- [ ] **SonarQube анализ** выполняется и результаты видны в SonarQube
- [ ] **Все stages** (test, static-analysis, build) выполняются успешно
- [ ] **Docker образ** успешно собирается

---

## 🎯 Готовые решения для распространенных проблем

### Если ничего не работает:
1. Проверьте, что все контейнеры запущены: `docker-compose ps`
2. Проверьте hosts файл: `cat /etc/hosts | grep localdomain`
3. Перезапустите все сервисы: `docker-compose restart && docker restart gitlab-runner`

### Если Runner не регистрируется:
1. Проверьте регистрационный токен в GitLab
2. Убедитесь, что URL GitLab указан правильно
3. Проверьте сетевую доступность из контейнера Runner

### Если пайплайн зависает:
1. Проверьте логи Runner: `docker logs gitlab-runner -f`
2. Убедитесь, что достаточно памяти и CPU
3. Проверьте, что Docker images могут скачиваться

---

Теперь у нас полностью рабочая среда GitLab с интеграцией SonarQube и автоматическими CI/CD пайплайнами! 🚀
