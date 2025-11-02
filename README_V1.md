Отличная работа по созданию руководства! Проанализировал ваш README и выделил ключевые проблемы, которые мешают интеграции с SonarQube. Вот исправленная и улучшенная версия руководства с пояснениями.

### Основные проблемы в исходном руководстве:

1.  **Сеть Docker:** Контейнеры по умолчанию находятся в разных сетях и не могут разрешить имена друг друга (например, `gitlab` не мог найти `sonarqube.localdomain`).
2.  **Анализ SonarQube в CI/CD:** Ключевой момент — анализ должен запускаться **изнутри** контейнера с GitLab Runner, а этот контейнер не имел доступа к SonarQube.
3.  **Настройки SonarQube:** Не было шага по созданию токена и настройки проекта в самом SonarQube.
4.  **Переменные CI/CD:** Токен SonarQube нужно хранить безопасно в переменных GitLab, а не в коде `.gitlab-ci.yml`.

---

### Исправленное и дополненное руководство

Замените содержимое вашего `README.md` на следующее:

# Практическое руководство: Развертывание GitLab с GitLab Runner и SonarQube в Docker

Это руководство описывает процесс развертывания связки GitLab + GitLab Runner + SonarQube с использованием Docker Compose. Особое внимание уделено корректной настройке сети для обеспечения взаимодействия между сервисами, особенно для доступа GitLab Runner к SonarQube по адресу `http://sonarqube.localdomain:9000`.

---

## Предварительные требования

- Сервер с установленными Docker и Docker Compose.
- Минимум 6 ГБ оперативной памяти (4 ГБ для GitLab + 2 ГБ для SonarQube).
- Достаточное дисковое пространство.
- Доступ к серверу с правами на выполнение Docker команд.

---

## Архитектура и Сетевая модель

Все сервисы будут размещены в пользовательской сети Docker `gitlab-network`. Это гарантирует, что контейнеры смогут находить друг друга по имени, и GitLab Runner сможет обращаться к SonarQube по имени `sonarqube`.

```
+-----------------------------------------------------------------------+
| Docker Host                                                           |
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

## Шаг 1: Подготовка окружения

1.  Создайте директорию для проекта и перейдите в нее:

    ```bash
    mkdir gitlab-sonarqube-setup && cd gitlab-sonarqube-setup
    ```

2.  Создайте файл `docker-compose.yml`.

---

## Шаг 2: Настройка Docker Compose

Скопируйте следующую конфигурацию в файл `docker-compose.yml`:

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

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab-runner-config:/etc/gitlab-runner
    networks:
      - gitlab-network

  sonarqube:
    image: sonarqube:latest
    container_name: sonarqube
    hostname: sonarqube.localdomain
    restart: always
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    networks:
      - gitlab-network

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
```

**Ключевые моменты конфигурации:**
- Все сервисы подключены к одной сети `gitlab-network`.
- Для SonarQube явно задан `hostname: sonarqube.localdomain`, что позволяет другим контейнерам обращаться к нему по этому имени.
- GitLab Runner монтирует Docker сокет (`/var/run/docker.sock`), что позволяет ему запускать контейнеры (режим Docker-in-Docker).

---

## Шаг 3: Запуск сервисов

1.  Запустите все сервисы в фоновом режиме:

    ```bash
    docker compose up -d
    ```

2.  Дождитесь полного запуска GitLab и SonarQube. Это может занять несколько минут. Проверяйте статус командой:

    ```bash
    docker compose logs -f gitlab
    # И для SonarQube
    docker compose logs -f sonarqube
    ```

    О готовности GitLab свидетельствует сообщение `gitlab Reconfigured!` в логах. SonarQube будет доступен на порту 9000.

---

## Шаг 4: Настройка SonarQube

1.  Откройте SonarQube в браузере: `http://<IP_ВАШЕГО_СЕРВЕРА>:9000`.
    *Логин: admin, Пароль: admin.*

2.  Создайте токен для доступа из GitLab CI:
    - Перейдите в свой профиль (верхний правый угол) -> **My Account** -> **Security**.
    - Введите имя токена (например, `gitlab-token`) и нажмите **Generate**.
    - **Скопируйте и сохраните этот токен!** Он понадобится на следующем шаге.

3.  Создайте проект вручную (например, `my-test-project`). Мы будем использовать его в CI-пайплайне.

---

## Шаг 5: Настройка GitLab Runner

1.  Получите регистрационный токен GitLab:
    - Войдите в GitLab (`http://<IP_ВАШЕГО_СЕРВЕРА>`) как root. Пароль можно посмотреть в логах:
      ```bash
      docker compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password
      ```
    - Перейдите **Admin Area** -> **CI/CD** -> **Runners**.
    - Скопируйте **Registration Token** из секции **Set up a specific Runner manually**.

2.  Зарегистрируйте Runner внутри его контейнера:

    ```bash
    docker compose exec gitlab-runner gitlab-runner register
    ```

    - **Enter the GitLab instance URL:** `http://gitlab/`
    - **Enter the registration token:** (вставьте ваш токен)
    - **Enter a description for the runner:** `docker-runner`
    - **Enter the tags for the runner:** (можно оставить пустым, нажать Enter)
    - **Enter optional maintenance note for the runner:** (можно оставить пустым, нажать Enter)
    - **Enter an executor:** `docker`
    - **Enter the default Docker image:** `alpine:latest`
    - **Enter the privileged mode (true/false):** `true` *[Важно! Нужен для запуска Docker-in-Docker]*

---

## Шаг 6: Настройка проекта в GitLab и переменных CI/CD

1.  Создайте новый проект в GitLab (или используйте существующий).
2.  В вашем проекте перейдите в **Settings** -> **CI/CD** -> **Variables**.
3.  Разверните секцию и добавьте две переменные:
    - **Key:** `SONAR_TOKEN` | **Value:** (токен, сгенерированный в SonarQube) | **Flags:** [x] Mask, [x] Protect
    - **Key:** `SONAR_HOST_URL` | **Value:** `http://sonarqube.localdomain:9000` | **Flags:** [ ] Mask, [ ] Protect

    *`SONAR_HOST_URL` указывает Runner'у, где находится SonarQube. Так как они в одной сети, он сможет разрешить это имя.*

---

## Шаг 7: Настройка `.gitlab-ci.yml` для анализа SonarQube

Создайте файл `.gitlab-ci.yml` в корне вашего репозитория. Этот пример для Java Maven проекта:

```yaml
stages:
  - test

sonarqube-check:
  stage: test
  image: maven:3.8.6-openjdk-11
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  script:
    - mvn verify sonar:sonar
  allow_failure: true
  only:
    - merge_requests
    - main
    - develop
```

**Для других языков (например, JavaScript/TypeScript) шаг `script` будет другим:**

```yaml
sonarqube-check:
  stage: test
  image: node:lts
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  before_script:
    - npm install -g typescript
    - npm ci
  script:
    - npx sonar-scanner
  allow_failure: true
  only:
    - merge_requests
    - main
    - develop
```

**Важно:** Для работы `npx sonar-scanner` в вашем проекте должен быть файл `sonar-project.properties`. Пример для Node.js проекта:

```properties
sonar.projectKey=my-test-project
sonar.projectName=My Test Project
sonar.sources=src
sonar.exclusions=node_modules/**
sonar.tests=test
sonar.test.inclusions=test/**
sonar.javascript.lcov.reportPaths=coverage/lcov.info
```

---

## Шаг 8: Проверка работоспособности

1.  Сделайте push в вашу ветку `main` или создайте Merge Request.
2.  В GitLab перейдите в **CI/CD** -> **Pipelines** и убедитесь, что пайплайн запустился.
3.  Если этап `sonarqube-check` проходит успешно (зеленый значок), перейдите в ваш проект в SonarQube (`http://<IP_ВАШЕГО_СЕРВЕРА>:9000`). Вы должны увидеть результаты анализа.

---

## Возможные проблемы и решения

1.  **Runner не может подключиться к SonarQube (`sonarqube.localdomain:9000`):**
    - Убедитесь, что все контейнеры в одной сети: `docker network inspect gitlab-network`.
    - Попробуйте изнутри контейнера Runner'а выполнить `ping sonarqube.localdomain`.

2.  **Ошибка "Failed to create task in SonarQube" или "Not authorized":**
    - Проверьте, что переменная `SONAR_TOKEN` создана корректно (без пробелов, правильный регистр).
    - Убедитесь, что в файле `sonar-project.properties` указан правильный `sonar.projectKey`, совпадающий с ключом проекта в SonarQube.

3.  **SonarQube падает с ошибкой памяти:**
    - Увеличьте RAM на сервере.
    - В сервисе `sonarqube` в `docker-compose.yml` добавьте ограничение памяти:
      ```yaml
      mem_limit: 2g
      ```

4.  **GitLab недоступен после перезапуска:**
    - Запустите команду `docker compose restart gitlab` и подождите 5-10 минут.

Это исправленное руководство решает ключевую проблему с сетью и предоставляет полный путь для успешной интеграции SonarQube в CI/CD пайплайн GitLab.
