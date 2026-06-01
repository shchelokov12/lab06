# Лабораторная работа №06

Студент: Щелоков Александр ИУ8-25

GitHub: shchelokov12

Gmail: aesch8877@gmail.com

## Цель работы
Ознакомиться с инструментом генерации пакетов CPack, входящим в состав CMake. Настроить локальную генерацию архивов проекта, а также разработать сценарий непрерывной интеграции (CI) на базе GitHub Actions для автоматической сборки кроссплатформенных пакетов (`.deb`, `.rpm`, `.dmg`, `.tar.gz`) приложения `solver` из лабораторной работы №3 при создании новых релизов (тегов) в Git.

## Ход работы

### 1. Подготовка репозитория
Задаем базовые переменные окружения, переходим в рабочую область
```
export GITHUB_USERNAME=shchelokov12
export GITHUB_EMAIL=aesch8877@gmail.com
alias edit=nano
alias gsed=sed

cd ${GITHUB_USERNAME}/workspace
pushd .
source scripts/activate
```
Клонируем репозиторий предыдущей лабораторной работы в новую директорию. Произведена перенастройка удаленного репозитория Git на новый URL.
```
git clone https://github.com{GITHUB_USERNAME}/lab05 projects/lab06
cd projects/lab06
git remote remove origin
git remote add origin https://github.com{GITHUB_USERNAME}/lab06
```

### 2. Настройка версионирования в CMakeLists.txt

```
gsed -i '/project(print)/a\
set(PRINT_VERSION_STRING "v\${PRINT_VERSION}")
' CMakeLists.txt
gsed -i '/project(print)/a\
set(PRINT_VERSION\
  \${PRINT_VERSION_MAJOR}.\${PRINT_VERSION_MINOR}.\${PRINT_VERSION_PATCH}.\${PRINT_VERSION_TWEAK})
' CMakeLists.txt
gsed -i '/project(print)/a\
set(PRINT_VERSION_TWEAK 0)
' CMakeLists.txt
gsed -i '/project(print)/a\
set(PRINT_VERSION_PATCH 0)
' CMakeLists.txt
gsed -i '/project(print)/a\
set(PRINT_VERSION_MINOR 1)
' CMakeLists.txt
gsed -i '/project(print)/a\
set(PRINT_VERSION_MAJOR 0)
' CMakeLists.txt
git diff
```

### 3. Создание сопроводительных файлов пакета
Генерируем текстовый файл описания пакета `DESCRIPTION` и историю изменений пакета `ChangeLog.md`

```
touch DESCRIPTION && edit DESCRIPTION
touch ChangeLog.md
export DATE="`LANG=en_US date +'%a %b %d %Y'`"
cat > ChangeLog.md <<EOF
* ${DATE} ${GITHUB_USERNAME} <${GITHUB_EMAIL}> 0.1.0.0
- Initial RPM release
EOF
```

### 4. Настройка базовой конфигурации CPack
Создаем конфигурационный файл `CPackConfig.cmake`, описывающий общие метаданные и настройки для генераторов пакетов систем Debian (`.deb`) и RedHat (`.rpm`).

```
cat > CPackConfig.cmake <<EOF
include(InstallRequiredSystemLibraries)
EOF

cat >> CPackConfig.cmake <<EOF
set(CPACK_PACKAGE_CONTACT ${GITHUB_EMAIL})
set(CPACK_PACKAGE_VERSION_MAJOR \${PRINT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR \${PRINT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH \${PRINT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK \${PRINT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION \${PRINT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_FILE \${CMAKE_CURRENT_SOURCE_DIR}/DESCRIPTION)
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "static C++ library for printing")
EOF

cat >> CPackConfig.cmake <<EOF
set(CPACK_RESOURCE_FILE_LICENSE \${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)
set(CPACK_RESOURCE_FILE_README \${CMAKE_CURRENT_SOURCE_DIR}/README.md)
EOF

cat >> CPackConfig.cmake <<EOF
set(CPACK_RPM_PACKAGE_NAME "print-devel")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "print")
set(CPACK_RPM_CHANGELOG_FILE \${CMAKE_CURRENT_SOURCE_DIR}/ChangeLog.md)
set(CPACK_RPM_PACKAGE_RELEASE 1)
EOF

cat >> CPackConfig.cmake <<EOF
set(CPACK_DEBIAN_PACKAGE_NAME "libprint-dev")
set(CPACK_DEBIAN_PACKAGE_PREDEPENDS "cmake >= 3.0")
set(CPACK_DEBIAN_PACKAGE_RELEASE 1)
EOF

cat >> CPackConfig.cmake <<EOF
include(CPack)
EOF
```

### 5. Подключение конфигурации CPack
Созданный файл `CPackConfig.cmake` подключаем в конец `CMakeLists.txt`. Адапатируем файл `README.md` для 6 лабораторной. Добавляем файлы в индекс Git, делаеим коммит, присваиваем проекту тег версии `v0.1.0.0` и отправляем на GitHub изменения.

```
cat >> CMakeLists.txt <<EOF
include(CPackConfig.cmake)
EOF

gsed -i 's/lab05/lab06/g' README.md

git add .
git commit -m"added cpack config"
git tag v0.1.0.0
git push origin master --tags
```

### 6. Локальная сборка и упаковка проекта
Были протестированы два способа вызова CPack. В первом случае генератор CPack запущен вручную из папки сборки. Во втором упаковка произведена через запуск специальной цели `package` средствами CMake. Из полученных архивов была сформирована директория артефактов `artifacts`.

```
cmake -H. -B_build
cmake --build _build
cd _build
cpack -G "TGZ"
cd ..

cmake -H. -B_build -DCPACK_GENERATOR="TGZ"
cmake --build _build --target package

mkdir artifacts
mv _build/*.tar.gz artifacts
tree artifacts
```

---

## Homework
Задача заключается в том, чтобы система сборки автоматически генерировала не только архив .tar.gz, а полный набор бинарных пакетов под разные операционные системы (.deb, .rpm, .msi, .dmg) при создании тега, а затем загружала их в релизы GitHub. Вместо устаревшего сервиса Travis CI конфигурация была реализована на GitHub Actions.

### 1. Используем матрицу сборки (matrix) прямо в файле release.yml. Это запустит параллельно виртуальные машины на Linux и macOS, чтобы собрать нужные пакеты.
```
name: CMake Pack and Multi-OS Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  build-and-pack:
    # Использование матрицы для запуска сборки одновременно на Linux и macOS
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            generator: "DEB;RPM"  # Собираем .deb и .rpm пакеты на Linux
          - os: macos-latest
            generator: "DragNDrop" # Собираем .dmg пакет на macOS

    runs-on: ${{ matrix.os }}

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    # Для сборки RPM-пакетов на Ubuntu утилите CPack нужен инструмент rpm
    - name: Install RPM tools (Linux only)
      if: matrix.os == 'ubuntu-latest'
      run: |
        sudo apt-get update
        sudo apt-get install -y rpm

    - name: Configure CMake
      run: cmake -H. -B_build -DCPACK_GENERATOR="${{ matrix.generator }}"

    - name: Build and Pack
      run: cmake --build _build --target package

    - name: Prepare Artifacts
      run: |
        mkdir artifacts
        # Переносим все сгенерированные пакеты (.deb, .rpm, .dmg) в папку artifacts
        mv _build/*.deb artifacts/ 2>/dev/null || true
        mv _build/*.rpm artifacts/ 2>/dev/null || true
        mv _build/*.dmg artifacts/ 2>/dev/null || true
        mv _build/*.tar.gz artifacts/ 2>/dev/null || true

    # Скрипт автоматически объединит файлы с обеих ОС и прикрепит их к одному релизу
    - name: Upload Artifacts to GitHub Release
      uses: softprops/action-gh-release@v2
      with:
        files: artifacts/*
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 2. Исправляем ошибку с версиями CMake
Исправляем первую строчку CMakeLists.txt
```
cmake_minimum_required(VERSION 3.5)
```

Оптимизируем блок strategy  в файле release.yml
```
strategy:
      fail-fast: false
      matrix:
        include:
          - os: ubuntu-latest
            generator: "DEB;RPM"
          - os: macos-latest
            generator: "DragNDrop"
```

### 3. Запускаем сборку
```
$ git add CMakeLists.txt .github/workflows/release.yml
$ git commit -m "fix: upgrade cmake minimum required version and disable fail-fast"
$ git push origin main

# Поднимаем тег для домашнего задания
$ git tag v0.1.0.4
$ git push origin --tags
```

## Вывод
В ходе лабораторной работы была настроена утилита CPack для создания установочных пакетов проекта и запустили в GitHub Actions автоматическую сборку, которая при создании тега сама создаёт готовые релизы с инсталляторами под Linux и macOS.
