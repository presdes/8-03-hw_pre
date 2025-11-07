# Практическое руководство: Развертывание GitLab с GitLab Runner и SonarQube в Docker для веб-разработки

## Введение

Это руководство предоставляет пошаговые инструкции по развертыванию собственного экземпляра GitLab, настроенного для веб-разработки (CSS, HTML, PHP, JavaScript), вместе с GitLab Runner и SonarQube в среде Docker. Мы настроим автоматизированный пайплайн CI/CD, который будет проверять код, включая анализ синтаксиса LESS, с помощью SonarQube и развертывать его.

## Архитектура решения

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

## Предварительные требования

- Сервер с ОС Ubuntu 22.04 LTS (или аналогичной)
- Минимум 4 ГБ ОЗУ (рекомендуется 8 ГБ для комфортной работы)
- Docker и Docker Compose установлены

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

## Настройка Docker сети

### Создание пользовательской сети

```bash
docker network create --driver=bridge --subnet=192.168.56.0/24 gitlab-network
```

### Проверка сети

```bash
docker network ls
docker network inspect gitlab-network
```

## Развертывание сервисов

### Создание директории и docker-compose.yml

```bash
mkdir gitlab-setup && cd gitlab-setup
```

Создайте файл `docker-compose.yml`:

```yaml
version: '3.8'

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    hostname: gitlab.localdomain
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.localdomain'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
    ports:
      - "80:80"
      - "443:443"
      - "2224:22"
    volumes:
      - gitlab_config:/etc/gitlab
      - gitlab_logs:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab
    restart: unless-stopped
    networks:
      gitlab-network:
        ipv4_address: 192.168.56.10

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: unless-stopped
    depends_on:
      - gitlab
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab_runner_config:/etc/gitlab-runner
    networks:
      gitlab-network:
        ipv4_address: 192.168.56.30

  sonarqube:
    image: sonarqube:latest
    container_name: sonarqube
    restart: unless-stopped
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_conf:/opt/sonarqube/conf
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    networks:
      gitlab-network:
        ipv4_address: 192.168.56.20

volumes:
  gitlab_config:
  gitlab_logs:
  gitlab_data:
  gitlab_runner_config:
  sonarqube_conf:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:

networks:
  gitlab-network:
    external: true
    name: gitlab-network
```

### Запуск GitLab

```bash
docker-compose up -d gitlab
```

### Мониторинг запуска GitLab

```bash
docker logs -f gitlab
```

Процесс инициализации может занять 5-10 минут. Дождитесь сообщений о успешном запуске служб.

### Настройка hosts файла

Добавьте в `/etc/hosts` на вашей машине:

```bash
192.168.56.10 gitlab.localdomain
192.168.56.20 sonarqube.localdomain
```

## Начальная настройка GitLab

### Получение пароля root

```bash
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

### Вход в GitLab

1. Откройте в браузере: `http://gitlab.localdomain`
2. Войдите с логином `root` и паролем из предыдущего шага
3. Смените пароль при первом входе

## Настройка GitLab Runner

### Регистрация Runner

```bash
docker-compose exec gitlab-runner gitlab-runner register
```

В процессе регистрации укажите:

- GitLab coordinator URL: `http://gitlab.localdomain/`
- Token (найдите в GitLab: **Admin -> CI/CD -> Runners** -> "Set up a specific Runner manually")
- Description: `docker-runner`
- Tags: `docker,linux`
- Executor: `docker`
- Default Docker image: `alpine:latest`

### Запуск всех сервисов

```bash
docker-compose up -d
```

### Проверка статуса сервисов

```bash
docker-compose ps
```

### Проверка сетевой связности

```bash
docker exec gitlab ping -c 3 192.168.56.20
docker exec gitlab-runner ping -c 3 192.168.56.10
```

## Развертывание и настройка SonarQube

### Начальная настройка SonarQube

1. Откройте в браузере: `http://sonarqube.localdomain:9000` или `http://localhost:9000`
2. Войдите с логином `admin` и паролем `admin`
3. Смените пароль при первом входе

### Настройка SonarQube для веб-технологий (CSS, HTML, PHP, JavaScript, LESS)

После входа в аккаунт администратора, необходимо установить плагины для поддержки всех нужных технологий и создать токен для нашего проекта.

1. **Установка необходимых плагинов:**
   - В верхнем меню перейдите в **Administration** -> **Marketplace**.
   - В поиске найдите и установите следующие плагины:
     - **SonarCSS** (для анализа CSS и LESS)
     - **SonarHTML** (для анализа HTML)
     - **SonarPHP** (для анализа PHP)
   - *Примечание: Плагины для JavaScript (SonarJS) и для общего анализа кода (SonarWay) обычно предустановлены.*
   - После установки плагинов потребуется перезагрузка SonarQube (система предложит это сделать).

2. **Создание токена для GitLab проекта:**
   - Нажмите на аватар вашего профиля в правом верхнем углу и выберите **My Account**.
   - Перейдите на вкладку **Security**.
   - Введите название токена, например, `gitlab-web-project-token`.
   - Скопируйте сгенерированный токен и сохраните его в надежном месте. **Он понадобится для настройки переменных в GitLab.**

## Создание и настройка тестового Веб-Проекта

Мы создадим простой веб-проект с файлом `index.html` и стилями, написанными на **LESS**. Наш пайплайн будет сначала проверять синтаксис LESS-файла с помощью SonarQube, и только в случае успеха компилировать его в CSS.

### Создание проекта в GitLab

1. Войдите в ваш GitLab и создайте новый проект `my-web-app`.

### Создание структуры файлов

В корне проекта создайте следующие файлы и папки:

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

**Файл: `sonar-project.properties`**
```properties
sonar.projectKey=my-web-app
sonar.projectName=My Web App

sonar.sources=.
sonar.exclusions=node_modules/**, css/**

# Для LESS анализа через плагин SonarCSS
sonar.css.less.parser.less=style.less
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

**Важно:** Создайте пустую папку `css/` в корне проекта. Git требует, чтобы папка существовала, прежде чем коммитить в нее файлы.

### Настройка переменных в GitLab

1. Перейдите в вашем проекте `my-web-app` в **Settings -> CI/CD -> Variables**.
2. Разверните секцию и добавьте две переменные:
   - **Key:** `SONAR_HOST_URL` | **Value:** `http://sonarqube.localdomain:9000` | **Не защищать, не маскировать**
   - **Key:** `SONAR_TOKEN` | **Value:** `[Токен, сгенерированный в SonarQube]` | **Защитить, замаскировать**

### Запуск пайплайна

1. Закоммитьте и запушьте все созданные файлы в ветку `main` вашего GitLab-репозитория.
2. Перейдите в **CI/CD -> Pipelines** и наблюдайте за выполнением.
3. **Этап `sonarqube-check`:** Сканер проверит код, включая синтаксис вашего `style.less`. Результат будет доступен в интерфейсе SonarQube (`http://sonarqube.localdomain:9000`).
4. **Этап `compile-less`:** Если анализ прошел успешно, этот этап скомпилирует `style.less` в `css/style.css`. Сгенерированный CSS-файл можно будет скачать из интерфейса пайплана в разделе "Артефакты".

## Проверка работы пайплайна

### В GitLab

- Перейдите в **CI/CD -> Pipelines** для просмотра статуса пайплайна
- Убедитесь, что оба этапа (`sonarqube-check` и `compile-less`) выполнены успешно
- Проверьте артефакты этапа `compile-less` - скачайте и проверьте сгенерированный CSS-файл

### В SonarQube

- Откройте `http://sonarqube.localdomain:9000`
- Найдите проект `My Web App` в списке проектов
- Убедитесь, что анализ выполнен и отображаются метрики качества кода
- Проверьте, что LESS-файл был проанализирован корректно

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

## Заключение

Вы успешно развернули полнофункциональную среду CI/CD для веб-разработки с правильно настроенной сетевой архитектурой, включающую:

- **GitLab** (192.168.56.10) для управления исходным кодом
- **GitLab Runner** (192.168.56.30) для выполнения пайплайнов
- **SonarQube** (192.168.56.20) для анализа качества кода веб-технологий
- **Автоматизированный пайплайн** для проверки LESS-синтаксиса и компиляции в CSS

Эта система обеспечивает автоматическую проверку качества кода и сборку ваших веб-проектов при каждом коммите в репозиторий.

## Дальнейшие шаги

- Настройка SSL-сертификатов для GitLab
- Настройка резервного копирования
- Добавление этапа деплоя в пайплайн
- Интеграция с другими инструментами качества кода
- Настройка уведомлений о результатах пайплайнов
- Масштабирование Runner для обработки нескольких проектов

---

*Версия V5 - Обновлено с исправленной сетевой конфигурацией и статическими IP-адресами*
