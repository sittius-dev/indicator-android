# Как работает android-build.yml — разбор по шагам

Этот документ объясняет `.github/workflows/android-build.yml` — CI-пайплайн,
который на GitHub-сервере (не на вашем компьютере) собирает `.apk` из
Capacitor-проекта каждый раз, когда вы делаете `git push` в ветку `master`.

## 0. Что такое GitHub Actions вообще

GitHub Actions — сервис, который по событию в репозитории (push, pull request,
таймер и т.д.) поднимает **временную виртуальную машину** ("runner"),
выполняет в ней список команд ("steps"), а затем машину уничтожает.
Никакого постоянного сервера у вас нет — каждый запуск получает чистую
Ubuntu-машину с нуля, как если бы вы только что достали новый компьютер
из коробки.

Файл лежит строго по пути `.github/workflows/<любое-имя>.yml` — это
единственное место, откуда GitHub считывает пайплайны. Можно держать
сколько угодно `.yml`-файлов в этой папке, каждый — отдельный независимый
workflow.

## 1. Заголовок и триггер запуска

```yaml
name: Android Build

on:
  push:
    branches: [master]
  workflow_dispatch: {}
```

- `name` — просто отображаемое имя, видно во вкладке Actions.
- `on.push.branches: [master]` — правило запуска: **каждый раз**, когда в ветку
  `master` попадает новый коммит (обычный push или merge), GitHub автоматически
  стартует этот workflow. Пуш в другие ветки его не триггерит.
- `on.workflow_dispatch: {}` — добавляет кнопку **"Run workflow"** во вкладке
  Actions, чтобы запустить пайплайн вручную, без нового коммита (например,
  чтобы пересобрать APK после того, как в другом репозитории обновили
  `index.html`, но здесь коммита ещё не делали).

## 2. Раннер

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```

`build` — имя job'а (можно было бы назвать иначе, GitHub его просто отображает
в UI как ярлык). `runs-on: ubuntu-latest` — заказываем чистую виртуалку с
последней Ubuntu LTS. GitHub бесплатно даёт лимит минут в месяц для приватных
репозиториев (публичные — без лимита).

Внутри одного `job` все `steps` выполняются **последовательно на одной и той
же машине** — файлы, скачанные на шаге 3, видны на шаге 8. Если бы был второй
`job`, он получил бы **отдельную** чистую машину без файлов первого.

## 3. Checkout — скачать репозиторий

```yaml
- name: Checkout
  uses: actions/checkout@v4
```

Свежая виртуалка изначально **пустая** — на ней нет даже вашего кода.
`actions/checkout` — официальный "экшен" (переиспользуемый готовый блок,
опубликованный на GitHub Marketplace), который делает `git clone` текущего
репозитория и переключается на тот коммит, который вызвал запуск. После
этого шага в `/home/runner/work/indicator-android/indicator-android/`
лежит точная копия того, что вы запушили.

## 4. JDK 21 — компилятор Java для Android-кода

```yaml
- name: Set up JDK 21
  uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '21'
```

Android-приложения компилируются из Java/Kotlin в байткод через
Gradle, а Gradle — сам JVM-инструмент, которому нужен установленный
**JDK** (Java Development Kit — компилятор + рантайм). `setup-java` скачивает
и разворачивает JDK нужной версии на виртуалке (`temurin` — конкретная
сборка OpenJDK от Eclipse Adoptium, самая распространённая бесплатная
сборка).

**Почему версия 21, а не 17 или 8**: библиотека `@capacitor/android` версии
8.x собрана с `sourceCompatibility/targetCompatibility = JavaVersion.VERSION_21`
в своём Gradle-модуле. Если поставить JDK младше — компиляция падает с
ошибкой `error: invalid source release: 21` (мы ловили это на прогоне
`#2`, см. раздел "Реальные ошибки" ниже). Требуемая версия JDK жёстко
привязана к версии зависимости `@capacitor/android` в `package.json` —
если когда-нибудь обновите Capacitor до другого мажора, эту версию, возможно,
придётся пересмотреть.

## 5. Node.js — для Capacitor CLI

```yaml
- name: Set up Node
  uses: actions/setup-node@v4
  with:
    node-version: '22'
```

Сам APK собирает Gradle (Java-инструмент), а не Node — но **до** сборки
нужно прогнать `npx cap sync` (шаг 7), а это уже node.js-инструмент
(Capacitor CLI написан на JS и распространяется через npm). Поэтому
раннеру ставится ещё и Node.

**Почему версия 22**: `@capacitor/cli` версии 8.x при запуске проверяет
версию Node и **отказывается работать** ниже 22 —
`[fatal] The Capacitor CLI requires NodeJS >=22.0.0` (поймали это на
прогоне `#1`, самом первом — упало за 15 секунд именно на этой проверке).

## 6. Установка npm-зависимостей

```yaml
- name: Install npm dependencies
  run: npm ci
```

`run:` (в отличие от `uses:`) — это не готовый экшен, а произвольная shell-
команда, выполняется как есть в bash на раннере.

`npm ci` (не `npm install`) — специально для CI: ставит **ровно** те версии
пакетов, что зафиксированы в `package-lock.json`, ничего не пересчитывая и
не обновляя. Если `package.json` и `package-lock.json` разъехались — `npm ci`
упадёт с ошибкой (это фича, а не баг: CI должен быть воспроизводим,
а не "как повезёт с последними версиями пакетов").

## 7. Синхронизация веб-ассетов в Android-проект

```yaml
- name: Sync Capacitor web assets
  run: npx cap sync android
```

`npx` — запускает CLI-инструмент из `node_modules/.bin` (тот, что только что
поставил `npm ci`) без глобальной установки. `cap sync android` — команда
Capacitor CLI, которая копирует `www/index.html` (уже закоммиченный в этом
репозитории файл — сам HTML/CSS/JS индикатора) в
`android/app/src/main/assets/public/index.html` и обновляет служебные
конфиги внутри Android-проекта (`capacitor.config.json`,
`capacitor.plugins.json`). Это тот же самый шаг, что вы руками запускаете
локально командой `npm run sync:android`.

## 8. Права на исполнение gradlew

```yaml
- name: Grant execute permission for gradlew
  run: chmod +x android/gradlew
```

`gradlew` ("Gradle wrapper") — shell-скрипт внутри репозитория, который сам
скачивает нужную версию Gradle и запускает сборку — за счёт этого не нужно
ставить Gradle на раннер отдельным шагом. Файл уже был закоммичен с правом
на исполнение (`git ls-files -s` показывает режим `100755`), но `chmod +x`
на всякий случай ставится явно — некоторые способы получения архива
репозитория (например, скачивание zip вместо git checkout) могут не сохранить
бит исполняемости, и без этой страховки шаг 9 упал бы с
`Permission denied`.

## 9. Сама сборка APK

```yaml
- name: Build debug APK
  run: |
    cd android
    ./gradlew assembleDebug --no-daemon
```

`|` в YAML — блок из нескольких команд построчно (эквивалент `cd android && ./gradlew ...`).

`./gradlew assembleDebug` — Gradle-таск, который проходит по графу зависимых
задач (компиляция Java, упаковка ресурсов, генерация `AndroidManifest.xml` в
бинарном виде, упаковка ассетов, подпись **debug**-ключом) и в итоге кладёт
файл в `android/app/build/outputs/apk/debug/app-debug.apk`.

`--no-daemon` — Gradle обычно держит фоновый процесс ("daemon") между
запусками, чтобы ускорить повторные локальные сборки за счёт прогретого
JVM/кэша. На CI-раннере, который живёт ровно один прогон и потом уничтожается,
это не даёт выигрыша, зато лишний фоновый процесс может помешать раннеру
корректно завершиться — поэтому его отключают.

**Debug vs release**: `assembleDebug` подписывает APK автоматически
генерируемым debug-ключом — годится только для локальной установки на
телефон (то, что мы и сделали). Для загрузки в Google Play нужен
`bundleRelease` с **вашим собственным** keystore-файлом и паролем,
переданными через GitHub Secrets — это отдельный, ещё не настроенный шаг.

## 10. Публикация результата

```yaml
- name: Upload debug APK
  uses: actions/upload-artifact@v4
  with:
    name: whale-watch-debug-apk
    path: android/app/build/outputs/apk/debug/app-debug.apk
```

Виртуалка после прогона **уничтожается вместе со всеми файлами** — если не
сохранить APK явно, он пропадёт бесследно. `actions/upload-artifact` берёт
файл по указанному `path` и заливает его как "Artifact", привязанный к
конкретному прогону — именно оттуда вы скачивали `whale-watch-debug-apk.zip`
на вкладке Actions. Артефакты по умолчанию хранятся 90 дней, потом
автоматически удаляются GitHub.

## Реальные ошибки, которые мы поймали на этом пайплайне

Полезно видеть не только "как правильно", но и как выглядят типичные
CI-ошибки для этого стека:

| Прогон | Упавший шаг | Ошибка | Причина | Фикс |
|---|---|---|---|---|
| #1 | Sync Capacitor web assets | `[fatal] The Capacitor CLI requires NodeJS >=22.0.0` | В workflow стоял Node 20 | `node-version: '20'` → `'22'` |
| #2 | Build debug APK | `error: invalid source release: 21` (в `compileDebugJavaWithJavac`) | В workflow стоял JDK 17, а Capacitor 8.x Android-модуль требует Java 21 | `java-version: '17'` → `'21'` |
| #3 | — | — | — | **Success**, артефакт 3.66 МБ |

Оба фикса — однострочные правки версии в `.yml`, каждая закоммичена и
запушена отдельно, что и видно в истории `git log` этого репозитория.
