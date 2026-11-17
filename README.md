# Minimalist Autonomous YouTube Player for Low-End PCs 🚀

Легковесный, автономный и полностью бесшовный плеер для воспроизведения YouTube-плейлистов (например, фоновой музыки и клипов) на слабых старых ноутбуках и мини-ПК (проверен на **Intel Celeron N4000C**, Windows 10/11).

Проект полностью избавлен от тяжелых зависимостей (нет Python, нет тяжелых GUI-оболочек) и оптимизирован под максимальную экономию оперативной памяти и процессорных ресурсов.

---

## 💡 Особенности конфигурации

*Без AV1 и тяжелых кодеков: Принудительное ограничение разрешения до 480p с использованием аппаратного ускорения (`hwdec=auto`) и выбора легких кодеков (`avc1` / `vp09`), так как старые процессоры не справляются с программным декодированием современных потоков.
* Встроенный SponsorBlock на чистом Lua: Автоматический пропуск ненужных вступлений (`intro`), концовок (`outro`) и разговорных пауз в клипах (`music_offtopic`) без использования Python — запросы к API выполняются напрямую через системный curl.
* Мгновенные переходы: Использование предварительной загрузки плейлиста (prefetch-playlist=yes) и буферизации (cache-secs=50), исключающее паузы между треками.
* Автообновление плейлиста: Плеер автоматически перезапрашивает свежий список треков у YouTube при пробуждении ноутбука (после закрытия/открытия крышки) или по нажатию клавиши [r].

---

## 🛠 Что необходимо скачать и почему

1. mpv (версия 0.39+ / Portable-сборка):
Выбирается портативная версия плеера. Она не засоряет реестр Windows, легко настраивается через папку [portable_config] и потребляет минимальное количество оперативной памяти.
к примеру mpv-x86_64-20261111-git-bcde6b93cc.7z [https://github.com/zhongfly/mpv-winbuild/releases]

2. yt-dlp.exe:
Основной движок для парсинга прямых ссылок на потоки и плейлисты с YouTube. Должен находиться в одной папке с `mpv.exe`.
к примеру тут [https://github.com/yt-dlp/yt-dlp]

3. curl.exe:
Используется легковесным Lua-скриптом SponsorBlock для быстрой отправки HTTP-запросов к серверу без установки тяжелых сред выполнения.
[В современных версиях Windows 10/11 встроен по умолчанию]

---

## 📁 Структура папок

```text
C:\mpv\
│   mpv.com
│   mpv.exe
│   run.bat
│   yt-dlp.exe
└───portable_config
    │   input.conf
    │   mpv.conf
    └───scripts
            sponsorblock_minimal.lua
```

---

## ⚙️ Конфигурационные файлы

### 1. mpv.conf

Основной файл настроек производительности, декодирования и кэша:

```text
# Аппаратное ускорение видео (использует графику Intel Celeron для H.264 / VP9)
hwdec=auto

# Фиксация качества через yt-dlp (ограничение до 480p для экономии ресурсов и стабильной работы на кухне)
ytdl-format="bestvideo[vcodec^=avc1][height<=?480]/bestvideo[vcodec^=vp09][height<=?480]+bestaudio/best[height<=?480]"
aid=1
# Сохранение позиции при закрытии и удержание плеера открытым при окончании трека
save-position-on-quit=yes
keep-open=yes
idle=yes

# Буферизация для минимизации задержек старта
cache=yes
demuxer-max-bytes=300M
cache-secs=50
tls-verify=no
fs=yes
terminal=no
prefetch-playlist=yes
```

### 2. `input.conf`

Привязка горячих клавиш:

```text
# Управление громкостью колесиком мыши или стрелками
UP add volume 5
DOWN add volume -5

# Пауза / Воспроизведение
SPACE cycle pause

# Переход на следующий / предыдущий трек в плейлисте
ENTER playlist-next
BS playlist-prev

# Выход
q quit

```

---

## `scripts/sponsorblock_minimal.lua`

Оптимизированный скрипт автоматического пропуска ненужных сегментов (работает через системный `curl` без внешних зависимостей):

```lua
local mp = require('mp')
local utils = require('mp.utils')

-- НАСТРОЙКИ КАТЕГОРИЙ (true - пропускать, false - игнорировать)
local categories_config = {
    intro = true,          -- Начальные титры / вступление
    outro = true,          -- Конечные титры / заставки
    music_offtopic = true  -- Болтовня / вставки без музыки в клипах
}

local current_segments = {}
local fetched_video_id = nil

-- Функция получения ID видео
local function get_video_id()
    local path = mp.get_property("path", "")
    local purl = mp.get_property("metadata/by-key/PURL", "")
    local ytdl_id = mp.get_property("ytdl-metadata/id", "")
    
    if ytdl_id ~= "" then return ytdl_id end

    local urls = {
        "youtu%.be/([%w-_]+)",
        "v=([%w-_]+)",
        "embed/([%w-_]+)"
    }
    for _, u in ipairs(urls) do
        local id = path:match(u) or purl:match(u)
        if id then return id end
    end
    return nil
end

-- Проверка позиции во время воспроизведения (вызывается только при смене времени)
local function skip_ads(name, pos)
    if not pos or #current_segments == 0 then return end
    for _, seg in ipairs(current_segments) do
        if pos >= seg.start and pos < seg.ending - 0.01 then
            mp.set_property("time-pos", seg.ending + 0.01)
            mp.osd_message(string.format("⏩ Пропущено (%s)", seg.category), 2)
            break
        end
    end
end

-- Запрос к API SponsorBlock через прямой вызов curl
local function fetch_segments(video_id)
    if not video_id or video_id == fetched_video_id then return end
    fetched_video_id = video_id
    current_segments = {}

    -- Собираем только те категории, которые включены в настройках выше
    local active_cats = {}
    for cat, enabled in pairs(categories_config) do
        if enabled then table.insert(active_cats, cat) end
    end
    if #active_cats == 0 then return end

    local cats_json = utils.format_json(active_cats)

    -- Вызываем curl напрямую (без PowerShell, мгновенно и без нагрузки)
    local res = utils.subprocess({
        args = {
            "curl", "-s", "-G",
            "https://sponsor.ajay.app/api/skipSegments",
            "--data-urlencode", "videoID=" .. video_id,
            "--data-urlencode", "categories=" .. cats_json
        }
    })

    if res.status ~= 0 or not res.stdout then return end

    -- Надежный парсинг JSON с помощью встроенных средств mpv
    local json = utils.parse_json(res.stdout)
    if type(json) == "table" then
        for _, item in ipairs(json) do
            if item.segment and item.category and categories_config[item.category] then
                table.insert(current_segments, {
                    start = item.segment[1],
                    ending = item.segment[2],
                    category = item.category
                })
            end
        end
    end

    if #current_segments > 0 then
        mp.osd_message("SponsorBlock: загружено сегментов: " .. #current_segments, 2)
        -- Включаем отслеживание таймлайна только если есть что пропускать
        mp.observe_property("time-pos", "native", skip_ads)
    end
end

-- Сброс и инициализация при загрузке нового видео
mp.register_event("file-loaded", function()
    fetched_video_id = nil
    current_segments = {}
    mp.unobserve_property(skip_ads)
    
    mp.add_timeout(1.0, function()
        local id = get_video_id()
        if id then
            fetch_segments(id)
        end
    end)
end)
```

---

## 🚀 Быстрый запуск [run.bat]

Для удобного запуска плейлиста в фоновом режиме без висящих окон консоли создайте в корне папки `C:\mpv\` файл [run.bat]:

```bat
@echo off
REM Запуск mpv с открытием вашего YouTube-плейлиста или ссылки на трансляцию
start /min "" C:\mpv\mpv.com "https://www.youtube.com/playlist?list=PL2BlfuqKJ9uyGgbUQ_1OdX_4e_trfdzEe"
timeout /t 22
exit

```
