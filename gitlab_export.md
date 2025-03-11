# 📌 Как экспортировать проекты GitLab с помощью gitlab-rails console и Bash-скрипта

В этой статье мы разберём процесс экспорта проектов в GitLab с использованием внутренней утилиты **gitlab-rails console** и автоматизации через **Bash-скрипт**. Этот метод полезен, если стандартные Rake-задачи (`gitlab:import_export:export`) не работают из-за ошибок или ограничений. Мы предоставим пошаговую инструкцию с проверкой и готовым скриптом.

---

## 🎯 Что мы делаем

Мы экспортируем проекты GitLab (например, `your_project/infrastructure/terraform` и `your_project/infrastructure/tfmodules`) в виде архивов `.tar.gz`, содержащих:
- Репозиторий
- Метаданные (issues, merge requests, pipelines и т. д.)
- Всё необходимое для импорта в другой GitLab-инстанс

Если Rake-задачи не работают, мы используем `gitlab-rails console` и Bash-скрипт.

---

## 🔹 Что такое gitlab-rails console?
`gitlab-rails console` — это интерактивная **Ruby-консоль**, встроенная в GitLab, которая позволяет работать с внутренними объектами приложения и базой данных.

### Почему использовать `gitlab-rails console`?
✅ **Обход ошибок**: если стандартные Rake-задачи не работают (например, с вложенными группами).  
✅ **Гибкость**: можно проверить доступ, структуру проекта и запустить экспорт в одном месте.  
✅ **Административный доступ**: консоль доступна только на self-managed GitLab и требует прав администратора.  
✅ **Асинхронность**: экспорт выполняется в фоновом режиме через Sidekiq.

---

## 🛠️ Пошаговая инструкция

### **1. Запуск `gitlab-rails console`**
Подключитесь к серверу GitLab через SSH и запустите консоль:
```bash
sudo gitlab-rails console
```
Вы увидите приглашение: `irb(main):001:0>` для ввода команд.

### **2. Проверка и запуск экспорта**

#### **Проверка пользователя и проекта**
```ruby
user = User.find_by_username('your_user')
project = Project.find_by_full_path('your_project/infrastructure/terraform')
```
Если `user` или `project` возвращают `nil`, проверьте:
- Имя пользователя (в профиле GitLab)
- Точный путь проекта (из URL проекта)

#### **Запуск экспорта**
```ruby
ProjectExportWorker.new.perform(user.id, project.id)
puts "Экспорт запущен для #{project.full_path}"
```
Вывод: `Экспорт запущен для your_project/infrastructure/terraform`.

#### **Поиск результата**
```bash
find /var/opt/gitlab/gitlab-rails/uploads/ -name "*.tar.gz" -mtime -1
```
📌 Архив создаётся в директории:
```
/var/opt/gitlab/gitlab-rails/uploads/-/system/import_export_upload/export_file/<number>/<timestamp>_your_project_infrastructure_terraform_export.tar.gz
```
Если архив не появился, подождите 5-30 минут и повторите команду.

---

## 🤖 Автоматизация с помощью Bash-скрипта
Для экспорта нескольких проектов используйте скрипт **export.sh**:

### **Скрипт `export.sh`**
```bash
#!/bin/bash

# === 🔹 Конфигурация ===
GITLAB_USER="your_user"
OUTPUT_DIR="archives"
REPO_LIST=("your_project/infrastructure/terraform" "your_project/infrastructure/tfmodules")

# === 🔹 Функции ===
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
}

export_project() {
    local repo_path="$1"
    local archive_path="${OUTPUT_DIR}/${repo_path}-export.tar.gz"
    local temp_dir="/var/opt/gitlab/gitlab-rails/uploads/-/system/import_export_upload/export_file"
    local log_file="${OUTPUT_DIR}/${repo_path}-export.log"

    mkdir -p "$(dirname "$archive_path")"
    if [[ -f "$archive_path" ]]; then
        log "ℹ️ Архив $archive_path уже существует, пропускаем."
        return 0
    fi

    log "📤 Начинаем экспорт проекта $repo_path..."
    echo "user = User.find_by_username('$GITLAB_USER'); project = Project.find_by_full_path('$repo_path'); ProjectExportWorker.new.perform(user.id, project.id)" | sudo gitlab-rails console > "$log_file" 2>&1

    local max_attempts=360
    local attempt=0
    while [[ $attempt -lt $max_attempts ]]; do
        local temp_archive=$(sudo find "$temp_dir" -name "*.tar.gz" -mtime -1)
        if [[ -n "$temp_archive" && -f "$temp_archive" ]]; then
            sudo mv "$temp_archive" "$archive_path"
            log "✅ Экспорт проекта $repo_path сохранён: $archive_path"
            return 0
        fi
        log "⏳ Ожидаем завершения экспорта $repo_path... (Попытка: $((attempt+1))/$max_attempts)"
        sleep 10
        ((attempt++))
    done

    log "❌ Ошибка экспорта $repo_path после 60 минут! Подробности в $log_file"
    cat "$log_file"
    return 1
}

log "🚀 Начинаем экспорт проектов..."
mkdir -p "$OUTPUT_DIR"

for repo in "${REPO_LIST[@]}"; do
    log "🔧 Обрабатываем репозиторий: $repo"
    export_project "$repo"
done

log "🎉 Все экспорты завершены!"
```

---

## 📌 Как использовать скрипт

1. **Создайте файл `export.sh`**:
```bash
nano export.sh
```
2. **Вставьте код, сохраните (`Ctrl+O`, Enter) и выйдите (`Ctrl+X`)**.
3. **Сделайте исполняемым:**
```bash
chmod +x export.sh
```
4. **Запустите:**
```bash
./export.sh
```

---

## 📜 Как работает скрипт
✅ **Конфигурация**: Определяются пользователь (`your_user`), выходная директория (`archives`) и список проектов (`REPO_LIST`).  
✅ **Функция `log`**: Логирует действия с временной меткой.  
✅ **Функция `export_project`**:
- Проверяет наличие архива.
- Запускает экспорт через `gitlab-rails console`.
- Ожидает до 60 минут появления архива.
- Перемещает архив в `archives/`.
  ✅ **Основной процесс**: Перебирает проекты и вызывает `export_project`.

---

🚀 Теперь вы можете легко экспортировать проекты GitLab для резервного копирования или миграции! 🔥

