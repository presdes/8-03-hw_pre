# Создание проекта GitLab CI

## Подготовка CI/CD пайплайна с оптимизацией выполнения

### Шаг 1: Создание структуры проекта

```bash
# Переходим в директорию проекта
cd ~/projects/netology/netology-gitlab

# Создаем структуру Go-проекта
mkdir -p src
```

### Шаг 2: Создание исходного кода Go

**src/main.go:**
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello Netology CI/CD!")
    result := add(5, 3)
    fmt.Printf("5 + 3 = %d\n", result)
}

func add(a, b int) int {
    return a + b
}

func multiply(a, b int) int {
    return a * b
}
```

**src/main_test.go:**
```go
package main

import "testing"

func TestAdd(t *testing.T) {
    result := add(2, 3)
    expected := 5
    if result != expected {
        t.Errorf("add(2, 3) = %d; want %d", result, expected)
    }
}

func TestMultiply(t *testing.T) {
    result := multiply(2, 3)
    expected := 6
    if result != expected {
        t.Errorf("multiply(2, 3) = %d; want %d", result, expected)
    }
}
```

**go.mod:**
```mod
module netology-gitlab

go 1.16
```

### Шаг 3: Создание Dockerfile

```dockerfile
FROM golang:1.16-alpine AS builder

WORKDIR /app
COPY go.mod ./
COPY src/ ./src/

RUN go build -o /netology-app ./src

FROM alpine:latest
WORKDIR /root/
COPY --from=builder /netology-app .
CMD ["./netology-app"]
```

### Шаг 4: Создание оптимизированного .gitlab-ci.yml

```yaml
stages:
  - build_and_test
  - static-analysis

variables:
  DOCKER_TLS_CERTDIR: ""

# Сборка Docker образа - запускается всегда
build:
  stage: build_and_test
  image: docker:latest
  services:
    - docker:dind
  script:
    - echo "🚀 Начинаем сборку Docker образа"
    - docker build -t netology-app:$CI_COMMIT_SHORT_SHA .
    - docker tag netology-app:$CI_COMMIT_SHORT_SHA netology-app:latest
    - echo "✅ Сборка завершена успешно"
  artifacts:
    paths:
      - Dockerfile
    expire_in: 1 hour
  tags:
    - docker
  only:
    changes:
      - Dockerfile
      - src/**/*
      - go.mod

# Тесты Go - запускаются только при изменении Go файлов
test:
  stage: build_and_test
  image: golang:1.16
  script:
    - echo "🧪 Запуск Go тестов"
    - cd src
    - go test -v .
    - echo "✅ Тесты выполнены успешно"
  tags:
    - docker
  rules:
    - changes:
        - "**/*.go"
      when: on_success
    - when: manual
      allow_failure: true

# Статический анализ кода
static-analysis:
  stage: static-analysis
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  script:
    - echo "🔍 Запуск статического анализа SonarQube"
    - sonar-scanner 
      -Dsonar.projectKey=netology-gitlab 
      -Dsonar.sources=src 
      -Dsonar.host.url=http://sonarqube:9000 
      -Dsonar.login=$SONAR_TOKEN
    - echo "✅ Анализ завершен"
  allow_failure: true
  tags:
    - docker
  only:
    changes:
      - "**/*.go"
      - sonar-project.properties

# Деплой (ручной запуск)
deploy:
  stage: static-analysis
  image: docker:latest
  services:
    - docker:dind
  script:
    - echo "🚀 Запуск деплоя"
    - docker run --rm netology-app:latest
    - echo "✅ Деплой завершен"
  when: manual
  tags:
    - docker
```

### Шаг 5: Создание sonar-project.properties

```properties
sonar.projectKey=netology-gitlab
sonar.projectName=Netology GitLab Project
sonar.projectVersion=1.0
sonar.sources=src
sonar.sourceEncoding=UTF-8
sonar.host.url=http://sonarqube:9000
sonar.login=${SONAR_TOKEN}
sonar.tests=src
sonar.test.inclusions=**/*_test.go
sonar.go.coverage.reportPaths=coverage.out
```

### Шаг 6: Запуск проекта в GitLab

```bash
# Добавляем все файлы в Git
git add .

# Создаем коммит
git commit -m "Добавлен оптимизированный CI/CD пайплайн с параллельным выполнением"

# Отправляем изменения в GitLab
git push origin main
```

### Шаг 7: Наблюдение за выполнением пайплайна

1. **Откройте GitLab:** `http://gitlab.localdomain/root/netology-gitlab`
2. **Перейдите:** CI/CD → Pipelines
3. **Наблюдайте за выполнением:**

**Ожидаемое поведение:**
- ✅ **Сборка** запускается сразу при любом изменении кода
- ✅ **Тесты** запускаются только при изменении `.go` файлов
- ✅ **Статический анализ** выполняется параллельно с другими задачами
- ✅ **Деплой** доступен для ручного запуска

### Шаг 8: Тестирование различных сценариев

#### Сценарий 1: Изменение Go файлов
```bash
# Вносим изменение в Go файл
echo "// Test comment" >> src/main.go

git add .
git commit -m "Обновление main.go"
git push origin main
```

**Результат:** Запустятся все этапы - сборка, тесты и статический анализ.

#### Сценарий 2: Изменение не-Go файлов
```bash
# Изменяем README.md
echo "### Documentation update" >> README.md

git add .
git commit -m "Обновление документации"
git push origin main
```

**Результат:** Запустится только сборка, тесты и анализ пропустятся.

#### Сценарий 3: Ручной запуск деплоя
1. В GitLab: CI/CD → Pipelines
2. Найдите выполненный пайплайн
3. Нажмите кнопку ▶️ у job "deploy"

### Шаг 9: Мониторинг и логи

**Просмотр логов выполнения:**
```bash
# Логи GitLab Runner
docker logs gitlab-runner --tail 50

# Проверка выполнения jobs
docker exec gitlab-runner gitlab-runner list
```

**Проверка в SonarQube:**
1. Откройте: `http://sonarqube.localdomain:9000`
2. Найдите проект "Netology GitLab Project"
3. Убедитесь, что анализ выполнен после изменения Go-файлов

## Ключевые особенности оптимизированного пайплайна

### 1. **Параллельное выполнение**
```yaml
stage: build_and_test  # Сборка и тесты в одной стадии
```
- Сборка не ждет завершения тестов
- Максимальное использование ресурсов Runner

### 2. **Условный запуск тестов**
```yaml
rules:
  - changes:
      - "**/*.go"
```
- Тесты запускаются только при изменении Go-файлов
- Экономия времени и ресурсов при изменении документации

### 3. **Гибкие правила выполнения**
```yaml
only:
  changes:
    - Dockerfile
    - src/**/*
    - go.mod
```
- Сборка запускается только при изменении relevant файлов
- Избегание ненужных сборок

### 4. **Артефакты и кэширование**
```yaml
artifacts:
  paths:
    - Dockerfile
  expire_in: 1 hour
```
- Сохранение результатов между stages
- Ускорение последующих выполнений

### 5. **Ручное управление**
```yaml
when: manual
```
- Контроль над запуском деплоя
- Возможность предварительной проверки

## Результат выполнения

После настройки вы получите:
- ✅ **Ускоренные пайплайны** за счет параллельного выполнения
- ✅ **Экономное использование ресурсов** благодаря условному запуску
- ✅ **Гибкую настройку** под различные сценарии разработки
- ✅ **Прозрачный процесс** с понятными этапами выполнения

Такой подход особенно эффективен в больших проектах, где время выполнения пайплайнов критически важно для скорости разработки.
