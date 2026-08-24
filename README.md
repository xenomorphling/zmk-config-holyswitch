# zmk-config

Конфигурация прошивки клавиатуры **Mriya** на ZMK. Проект собирает две половины: левую (`mriya_left`) и правую (`mriya_right`).

## Требования

- **Docker** (используется официальный образ `zmkfirmware/zmk-build-arm:stable`).
- В `config/west.yml` закреплена версия `revision: v0.3.0` (Zephyr 3.5). Кастомные борды `mriya_*` несовместимы с `main` (Zephyr 4.1), поэтому **не меняйте** ревизию на `main`.

## Сборка локально

### Шаг 1: подготовка копии для сборки

`west init` создаёт workspace прямо в клонированном конфиге (папки `.west`, `zmk`, `zephyr`, `modules`, `build-*`), поэтому удобно работать в отдельной рабочей копии. Выполните из корня проекта:

```bash
# пересоздать чистую директорию сборки с последними правками
rm -rf /tmp/zmk-modules && mkdir -p /tmp/zmk-modules
git clone "$PWD" /tmp/zmk-modules/zmk-config
```

### Шаг 2: предзагрузка модулей (один раз, долго)

Скачивает ZMK v0.3.0, Zephyr 3.5 и все модули, затем экспортирует CMake-пакеты:

```bash
docker run --rm -v /tmp/zmk-modules:/workspace -w /workspace \
  zmkfirmware/zmk-build-arm:stable \
  /bin/bash -lc "west init -l zmk-config/config && cd zmk-config && west update && west zephyr-export"
```

### Шаг 3: сборка прошивок

Сборка каждой половины занимает несколько минут. `zephyr-export` нужно выполнять в том же контейнере, что и сборку (иначе Zephyr не найдётся):

```bash
# левая половина (central)
docker run --rm -v /tmp/zmk-modules:/workspace -w /workspace/zmk-config \
  zmkfirmware/zmk-build-arm:stable \
  /bin/bash -lc "west zephyr-export && west build -s zmk/app -d build-left -b mriya_left \
    -- -DZMK_CONFIG=/workspace/zmk-config/config"

# правая половина (peripheral)
docker run --rm -v /tmp/zmk-modules:/workspace -w /workspace/zmk-config \
  zmkfirmware/zmk-build-arm:stable \
  /bin/bash -lc "west zephyr-export && west build -s zmk/app -d build-right -b mriya_right \
    -- -DZMK_CONFIG=/workspace/zmk-config/config"
```

Результат:

```
/tmp/zmk-modules/zmk-config/build-left/zephyr/zmk.uf2   (левая)
/tmp/zmk-modules/zmk-config/build-right/zephyr/zmk.uf2  (правая)
```

### Шаг 4: скопируйте прошивки в проект

```bash
cp /tmp/zmk-modules/zmk-config/build-left/zephyr/zmk.uf2  ./mriya_left-zmk.uf2
cp /tmp/zmk-modules/zmk-config/build-right/zephyr/zmk.uf2 ./mriya_right-zmk.uf2
```

## Прошивка на клавиатуру

Контроллеры — **nice!nano v2**. Чтобы прошить:

1. Отключите USB-кабель.
2. Дважды быстро нажмите кнопку **RESET** на прошиваемой половине — появится USB-диск `NICENANO`.
3. Перетащите `.uf2` на диск:
   - `mriya_left-zmk.uf2` → на **левую** половину (central)
   - `mriya_right-zmk.uf2` → на **правую** половину (peripheral)
4. Прошивайте сначала левую, затем правую — BLE-связка между половинами устанавливается автоматически.

## GitHub Actions

Сборка также автоматически выполняется через GitHub Actions (см. `.github/workflows/`). Список собираемых плат задаётся в `build.yaml` (по умолчанию `mriya_left` и `mriya_right`).