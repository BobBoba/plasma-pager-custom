# plasma-pager-custom

Локальная пересборка пакета `plasma-desktop` с одной добавленной возможностью: **настраиваемый шрифт (семейство + размер) и цвет подписей** у виджета **Pager** (и **Activity Pager**).

Зачем так сложно: в Plasma 6 виджет Pager — это скомпилированный C++/QML апплет (`/usr/lib/qt6/plugins/plasma/applets/org.kde.plasma.pager.so`); его `main.qml` вкомпилирован в бинарник как AOT QML-кэш, поэтому редактируемого QML на диске нет. Единственный способ изменить интерфейс — пропатчить исходники и пересобрать пакет.

## Что внутри

| Файл | Назначение |
|---|---|
| `PKGBUILD` | копия официального Arch-пакета `plasma-desktop` 6.7.1-1 + подключение патча в `prepare()` |
| `0001-pager-custom-label-font-and-color.patch` | сам патч (3 файла, ~109 строк) |
| `arch-pkg/` | нетронутый клон официального пакета для сверки/обновления |
| `README.md` | этот файл |

Патч трогает только апплет Pager:

- `applets/pager/main.xml` — восемь новых ключей конфигурации:
  - шрифт/цвет: `customFontEnabled`, `fontFamily`, `fontSize`, `customColorEnabled`, `textColor`;
  - читаемость подписи: `labelEffect` (enum None/Outline/Shadow/Scrim), `effectColorAuto` (авто-контраст цвета эффекта), `effectColor`.
- `applets/pager/qml/configGeneral.qml` — раздел «Label appearance» в настройках виджета: переключатели, выбор семейства шрифта (`Button` + всплывающий `Popup` с **превью каждого начертания в его же шрифте** и **полем поиска** `Kirigami.SearchField`; реализовано через `Button`, а не `ComboBox`, потому что стиль `org.kde.desktop` рисует текущий текст `ComboBox` нативным `StyleItem` и завязывает фон на `popup.exit` — переопределение `contentItem`+`popup` там ломает отрисовку), размер (0 = авто), кнопка цвета текста, выпадающий список эффекта читаемости, авто-цвет эффекта и кнопка цвета эффекта.
- `applets/pager/qml/main.qml` — применение настроек в делегате подписи (обёрнут в `Item`, чтобы scrim рисовался позади текста). Эффекты:
  - **Outline** — встроенный `Text.style = Outline` + `styleColor` (без шейдера);
  - **Shadow** — `MultiEffect` (`QtQuick.Effects`) как `layer.effect`, drop shadow на GPU;
  - **Scrim** — полупрозрачная подложка `Rectangle` под текстом;
  - цвет эффекта в режиме Auto выбирается по контрасту через `Kirigami.ColorUtils.brightnessForColor`.
- **Поведение по умолчанию не меняется**: пока переключатели выключены и `labelEffect = None`, рендеринг байт-в-байт совпадает со стоковым (тема задаёт шрифт и цвет).

## Сборка и установка

Требуется `base-devel`. Зависимости сборки `makepkg` доустановит сам (нужен `sudo`).

```bash
cd /code/github/plasma-pager-custom

# Собрать и установить (makepkg -s доустановит makedepends, -i установит пакет)
makepkg -si
```

Альтернатива — собрать, затем установить вручную:

```bash
makepkg -s
sudo pacman -U plasma-desktop-6.7.1-1-x86_64.pkg.tar.zst
```

> ⚠️ Это пересборка **всего** пакета `plasma-desktop`, а не одного апплета — первая сборка занимает несколько минут и подтягивает dev-зависимости KDE. Последующие пересборки быстрее (кэш ccache, если включён).

## Применение изменений

Полный выход из сессии **не требуется** — достаточно перезапустить оболочку:

```bash
systemctl --user restart plasma-plasmashell.service
# fallback, если это не systemd-сервис:
# kquitapp6 plasmashell; kstart plasmashell
```

Затем: правый клик по виджету Pager → «Настроить…» → раздел **Label appearance** → включить «Use a custom font» / «Use a custom text color» и задать значения.

## После обновления plasma-desktop

Стоковый `plasma-desktop` из репозитория при `pacman -Syu` заменит этот пакет и откатит патч (это ожидаемо — версии совпадают, пока репозиторий не обновится). Порядок действий:

1. Обновить эталон и сверить версию:
   ```bash
   cd arch-pkg && git pull && grep -E '^pkgver|^pkgrel|sha256sums' PKGBUILD
   ```
2. Перенести новые `pkgver`/`pkgrel`/`sha256sums` в корневой `PKGBUILD`.
3. Перепроверить, что патч применяется к новой версии (если апстрим переписал `main.qml`/`configGeneral.qml` — поправить патч):
   ```bash
   makepkg -od --skippgpcheck            # распаковать новый исходник
   cd src/plasma-desktop-*/ && patch -Np1 --dry-run < ../../0001-pager-custom-label-font-and-color.patch
   ```
4. Пересобрать: `makepkg -si`.

Чтобы случайное `pacman -Syu` не откатило сборку между вашими пересборками, можно временно держать пакет в `IgnorePkg` (`/etc/pacman.conf`), не забывая снимать его перед намеренным обновлением.

## Регенерация патча (если правите QML дальше)

Исходники извлекаются в `src/`. Удобный цикл с git:

```bash
makepkg -od --skippgpcheck
cd src/plasma-desktop-*/
git init -q && git add -A && git commit -qm baseline
# ...правки в applets/pager/...
git diff > ../../0001-pager-custom-label-font-and-color.patch
```

## Заметки для upstream MR

Изменение оформлено как полноценная фича (конфиг-ключи + GUI + применение), а не хардкод, поэтому годится как основа merge request в `invent.kde.org/plasma/plasma-desktop` (апплет `applets/pager/`).

Честная оговорка: KDE исторически сдержанно относится к per-widget переопределению шрифта/цвета (философия единообразия через тему/цветовую схему, KDE HIG). У «размера шрифта» шансов на приёмку больше, чем у произвольного «цвета». Перед отправкой MR стоит обсудить идею в issue/Matrix-канале Plasma, чтобы не делать работу впустую.
