# План форку aRDP

RDP-клієнт для Android-планшетів і телефонів з фізичною клавіатурою на основі aRDP від iiordanov.

**Цілі:**

1. Прокидати **Alt+Tab і Win-комбінації** з фізичної клавіатури у віддалену сесію.
2. Закрити **знайдені проблеми безпеки**.
3. **Вирізати все, що не стосується RDP**: VNC, SPICE, Opaque/Proxmox, Google Play, рекламу тощо.
4. Збирати APK **у GitHub Actions з вихідного коду**, без непрозорих готових бінарників.

**Статус на 2026-11-03:** лише план, код ще не змінено.
**База:** upstream `fd308d8` (2026-09-27, v6.4.9, build 116490).
Номери рядків нижче вказані для цього коміту.

> Технічні нотатки для Claude лежать у [`CLAUDE.md`](CLAUDE.md). Увага: спосіб збірки «Fastest setup» звідти зараз **не працює**, див. розділ 4.

---

## 1. Клавіатура: Alt+Tab і Win

### У чому проблема

Android забирає ці комбінації собі ще до того, як вони доходять до будь-якого застосунку:

- **Alt+Tab** перехоплює `PhoneWindowManager.interceptKeyBeforeDispatching` і відкриває перемикач застосунків Android.
- **Win/Meta** перехоплюється ще раніше, в `interceptKeyBeforeQueueing`, і відкриває асистента чи системні шорткати.

Це не баг aRDP: так само поводяться Windows App, Remote Desktop Manager, TeamViewer, Moonlight. Звичайний застосунок без обходу ці клавіші просто не отримує.

### Що вже є в aRDP

| Що | Де | Обмеження |
|---|---|---|
| Samsung-хак: приховане API One UI `SemWindowManager.requestMetaKeyEvent(componentName, true)`, викликається через рефлексію | `bVNC/.../iiordanov/util/SamsungDexUtils.kt`, виклик у `RemoteCanvasActivity.java:831` на кожен старт сесії (не лише в DeX) | Лише Samsung і лише Meta (Win). Alt+Tab, найімовірніше, ні. Новий One UI може тихо зламати його: у лозі буде `Could not call ...requestMetaKeyEvent` |
| Маппінг Win на віддалений бік | `RdpKeyboardMapper.java:352`: `KEYCODE_META_LEFT/RIGHT` → `VK_LWIN/VK_RWIN` | Працює, лише якщо подія взагалі дійшла до застосунку |
| Панель Extra Keys на екрані (кнопки SUPER, TAB) | `bVNC/.../extrakeys/` | Треба тицяти в екран |
| Меню → **Send Keys** («Send Special Keys»): комбінація з галочкою Super, яку можна зберегти; **Send Key Again** повторює останню | `dialogs/MetaKeyDialog.java` | Незручно для частого використання |

### Обхідні шляхи без коду

- **RDP-замінники:** `Alt+Home` = Win, `Alt+PgUp` / `Alt+PgDn` = Alt+Tab / Alt+Shift+Tab, `Ctrl+Alt+End` = Ctrl+Alt+Del. Win+літера так не працює.
- **PowerToys Keyboard Manager на Windows:** перепризначити клавішу, яку Android не чіпає, на потрібну комбінацію (наприклад, правий Alt → Alt+Tab).
- **Samsung DeX:** за старими звітами в DeX-режимі хоткеї доходять до RDP-клієнта. Не перевірено.

### План

0. **Перевірити офіційний aRDP на цільовому пристрої (зокрема Samsung):** Win, Win+E/D/R, Alt+Tab. Якщо проходить усе, патч не потрібен. Обсяг патча від кількості комбінацій майже не залежить: сервіс бачить усі апаратні клавіші, і вирішується лише те, які з них забирати в Android.
1. **Прототип-«детектор клавіш».** Окремий мінімальний APK з `AccessibilityService` (`flagRequestFilterKeyEvents`), який показує на екрані, які події доходять до `onKeyEvent`.
   - **Відкрите питання:** чи доходить до accessibility-фільтра «голий» Win, якщо Meta обробляється в `interceptKeyBeforeQueueing`. Автори shortcut-passthrough кажуть, що так. Прототип це покаже.
2. **Якщо прототип працює**, додати `AccessibilityService` у форк:
   - Сервіс активний **лише поки `RemoteCanvasActivity` на передньому плані.** Фільтр змушує кожну клавішу проходити синхронний round-trip через сервіс, тож поза сесією він має бути вимкнений.
   - Передає події в `keyboard.keyEvent(...)` поточного `RemoteConnection` і повертає `true`, щоб Android їх не обробив.
   - Можна забирати **всі** апаратні клавіші, поки сесія на екрані: один шлях для всього, без ризику, що клавіша прийде двічі. Або забирати лише ті, які краде Android. Коду в обох варіантах приблизно однаково.
   - На відміну від KeyRelay чи shortcut-passthrough, **не потрібні ні агент на Windows, ні мережевий канал**: клавіші йдуть прямо в RDP-сесію.
3. **Якщо accessibility не бачить потрібних клавіш**, читати `/dev/input` через бездротову налагодку (ADB shell), як KeyRelay. Мінус: після кожного перезавантаження планшета хелпер треба запускати заново.

### Куди вбудовуватися (ланцюжок обробки клавіш)

`RemoteCanvasActivity.java:1217` `canvas.setOnKeyListener(inputListener)` → `RemoteClientsInputListener` → `RemoteConnection.onKeyDownEvent / onKeyUpEvent` (`RemoteConnection.java:373`) → `keyboard.keyEvent(...)` → `RemoteRdpKeyboard.processLocalKeyEvent` → `RdpKeyboardMapper` → `RdpCommunicator.sendKeyEvent` → `LibFreeRDP`.

### Довідка

- [shortcut-passthrough](https://github.com/OverDustD7/shortcut-passthrough): accessibility-фільтр → UDP → скрипт на хості
- [KeyRelay](https://github.com/furkanisikay/KeyRelay): `/dev/input` через wireless debugging → TLS → агент на ПК
- [skypc-issues #103](https://github.com/theskyPC/skypc-issues/issues/103): той самий опис проблеми на Android 16

---

## 2. Безпека: що знайдено і що виправити

Аудит від 2026-11-03 по коміту `fd308d8`.

**Чого не знайдено:** телеметрії, аналітики, реклами, crash-репортерів. Даних на сервери автора застосунок не відправляє.

**Куди застосунок ходить у мережу:**

- Лише на вказаний сервер (і SSH-сервер, якщо налаштовано тунель).
- Кнопка «+» запускає пошук ПК у локальній мережі: перебір /24 на порт RDP (`PortScanDiscovery.java`) плюс mDNS.
- Якщо замість IP вписати ім'я ПК, відправляється NetBIOS-запит у LAN.
- Google Play — лише кнопка «Оцінити».
- Посилання (Morpheusly, GitHub, email підтримки) відкриваються тільки натисканням.

**Дозволи безкоштовного aRDP** (`tools:node="removeAll"`): лише `INTERNET`, `ACCESS_NETWORK_STATE`, `VIBRATE`.

### Проблеми

| # | Проблема | Де | Що робити | Важливість |
|---|---|---|---|---|
| 1 | **FreeRDP 2.11.7.** Гілка 2.x закрита: 2.11.8 (2026-02-25) — останній реліз 2.x, і aRDP навіть до нього не оновився (наприклад, CVE-2026-25941: шкідливий сервер читає неініціалізовану пам'ять клієнта) | `remoteClientLib/jni/libs/build-deps.conf` (`freerdp_ver`) | Мінімум — оновити до **2.11.8**. Перехід на 3.x — це великий порт: патчі aRDP написані під 2.x, а Java-обгортка FreeRDP 3 інша | Висока |
| 2 | **OpenSSL 1.1.1w**, підтримка закінчилася 2023-09-11 | `build-deps.conf` (`openssl_ver`), FreeRDP `scripts/android-build.conf` | Спробувати OpenSSL 3.x. Ризик: NTLM у NLA потребує MD4/RC4, яких в OpenSSL 3 за замовчуванням немає. Треба або `WITH_INTERNAL_MD4/RC4`, або legacy provider, і обов'язково перевірити вхід з NLA | Висока |
| 3 | **Змінений сертифікат RDP виглядає як перше підключення.** `OnVerifyChangedCertificate` викликає той самий діалог, прапорець mismatch у `Bundle` не передається, `validateCert(save=false)` ніколи не показує `warning_cert_does_not_match` | `RdpCommunicator.java:524–571`, `RemoteCanvasHandler.java:239–282` | Передавати mismatch і старий відбиток, показувати явне попередження, а на mismatch робити кнопкою за замовчуванням «Ні». Плюс: патч `03` прибирає `/cert-ignore` з офіційного Android-клієнта FreeRDP, тож сертифікат узагалі перевіряється. **Не повертати cert-ignore** | Середня |
| 4 | **Посилання `rdp://`** (intent-filter з `BROWSABLE`). Посилання з браузера чи месенджера одразу запускає підключення без підтвердження. Якщо host:port збігається зі збереженим підключенням, підставляється збережений пароль. Параметри посилання (`SshHost`, `SshPassword`, `IdHash`, `VpnUriScheme`…) за замовчуванням **записуються у збережене підключення** (`SaveConnection=true`) | маніфест `freeaRDP-app`, `UriIntentParser.java`, `AbstractConnectionBean.java:2256–2330`, `RemoteCanvasActivity.java:336` | Прибрати `BROWSABLE` або показувати діалог підтвердження. Не застосовувати параметри з посилання до збереженого підключення. Зробити `SaveConnection=false` за замовчуванням. **Тимчасово:** увімкнути майстер-пароль, він блокує такі посилання | Середня |
| 5 | **Паролі відкритим текстом без майстер-пароля.** База — SQLCipher, але з порожнім ключем (`Database.password = ""`). Захищає лише пісочниця Android, бекап вимкнено (`allowBackup=false`) | `Database.java:137–255` | Шифрувати ключем з Android Keystore за замовчуванням, а майстер-пароль залишити як додатковий рівень | Середня |
| 6 | **Експорт налаштувань** — XML з усіма паролями відкритим текстом | `Utils.exportSettingsToXml` → `SqliteElement.exportDbAsXmlToStream` | Показувати попередження або не включати паролі в експорт | Низька |
| 7 | Дрібниці: `PasswordManager.encrypt` пише пароль у лог (мертвий код, ніде не викликається); `Log.d("Got data: " + data)` — URI може містити пароль (`AbstractConnectionBean.java` ≈2104); `PRAGMA key = '%s'` через `format` — апостроф у майстер-паролі все зламає (`Database.java:202–210`); у патчі `16` `"/sec:nla /sec:ext"` передається **одним** аргументом, схоже на баг | — | Видалити або виправити заодно | Низька |
| 8 | **Ланцюг постачання.** Готовий архів `libs-17` качається без контрольної суми і вже застарілий (див. розділ 4). У `build-deps.conf` є `http://` URL (майже всі для SPICE/GStreamer). NDK і sqlcipher теж без перевірки хешу. Скрипт aRDP вмикає `WITH_FFMPEG=1` з **ffmpeg n4.4.1 (2021)** | `download-prebuilt-dependencies.sh`, `build-deps.conf`, `bVNC/prepare_project.sh`, `build-deps.sh:553–554` | Збирати FreeRDP з вихідного коду (власний скрипт FreeRDP перевіряє sha256 OpenSSL, openh264 і ffmpeg). **FFMPEG вимкнути**: є MediaCodec і openh264. Додати хеш для sqlcipher | Середня |

Із проблемою 1–2 ризик реальний лише при підключенні до шкідливого сервера або коли хтось стоїть посередині з'єднання. Для власного ПК у домашній мережі або через Tailscale практичний ризик низький, але це найсерйозніше з усього списку.

### Як перевірити самостійно

- **Код, 2–4 години:**
  - маніфест `freeaRDP-app`
  - `UriIntentParser.java`
  - `AbstractConnectionBean.java` (≈2090–2330)
  - `RdpCommunicator.java` (≈360–600)
  - `RemoteCanvasHandler.java` (≈200–290)
  - `Database.java`
  - `ClipboardMonitor.java`
  - `PortScanDiscovery.java`
  - `Utils.java` (≈780–850)
  - 22 файли `remoteClientLib/jni/libs/*_freerdp_*.patch` (≈600 рядків разом)
- **Під час роботи:** PCAPdroid на планшеті (без root) показує всі з'єднання конкретного застосунку.
- **Що не перевірялося:** нативний код самого FreeRDP (офіційний upstream, дуже великий) і поведінка застосунку під час роботи.

---

## 3. Вирізання непотрібного

Головний компроміс: **кожен вирізаний шматок ускладнює підтягування оновлень з upstream.** Тому ріжемо етапами, від дешевого до дорогого. Етапи A–B майже не заважають merge, етап C — це вже фактично окремий проєкт.

### Етап A: без змін Java-коду

- **`settings.gradle`:** залишити `:freeaRDP-app`, `:bVNC`, `:remoteClientLib`, `freeRDPCore`, `:common`, `:pubkeyGenerator`. Прибрати `bVNC-app`, `freebVNC-app`, `aSPICE-app`, `freeaSPICE-app`, `CustomVnc-app`, `Opaque-app`, `aRDP-app` (Pro).
- **ABI:** лише `arm64-v8a` (сучасні планшети й телефони). `armeabi-v7a` додавати, тільки якщо знадобиться старий 32-бітний пристрій. Місця: `splits` у `freeaRDP-app/build.gradle`, `abiFilters` у `remoteClientLib/build.gradle`, `BUILD_ARCH` для FreeRDP. FreeRDP збиратиметься в ~4 рази швидше, APK стане меншим.
- **Нативна збірка:** тільки `build_freerdp`, без cerbero/GStreamer, spice-gtk, libgovirt, libsoup, librest, libusb, usbredir, gmp, nettle. Разом із ними зникають майже всі `http://`-завантаження і старі бібліотеки з `build-deps.conf`. Для aRDP вони й так не потрапляють в APK: `prepare_project.sh` видаляє їх `.so`.
- `externalNativeBuild` у `remoteClientLib/build.gradle` уже закоментований (SPICE JNI не збирається). NDK потрібен лише для FreeRDP.

### Етап B: дрібні правки коду

| Що | Навіщо воно в upstream | Дія |
|---|---|---|
| `com.google.android.gms:play-services-base` 18.0.1, `com.google.android.play:review` і `review-ktx` 2.0.1 | Лише діалог «Оцінити» (`Utils.showRateAppDialog`, `ConnectionGridActivity.rateApp`) | Видалити разом із пунктом меню. Застосунок стає **без Google-бібліотек** |
| `MorpheuslyBottomSheet`, `rateOrShareFragment` | Реклама іншого продукту автора і донати | Видалити |
| `jcifs` 1.3.17 (2011 рік) | NetBIOS: ім'я ПК → IP (`Utils.resolveNetbiosAddress`) | Видалити, вводити IP або ім'я з Tailscale/DNS |
| `androidx.legacy:legacy-support-v4`, `com.android.support:multidex:1.0.3` | Легасі: ніде не імпортуються, а при `minSdk 21` multidex і так вбудований | Видалити й перевірити збірку |
| Три бібліотеки ed25519: `net.vrallev.ecc:ecc-25519-java`, `io.moatwel.crypto:eddsa`, `net.i2p.crypto:eddsa` | Напряму ніде не імпортуються, можливо, потрібні транзитивно для `sshlib` | Видаляти по одній і перевіряти SSH-тунель з ключем ed25519 |
| `pubkeyGenerator/.../net/sourceforge/jsocks/` (вбудований SOCKS-**сервер**) | Не використовується | Видалити |
| `pubkeyGenerator/libs/*.jar` | У Gradle не підключені, замість них беруться Maven-версії | Видалити |
| Assets: `bVNC/src/main/assets/icudt46l.zip` (2,2 МБ, з Java не використовується), `layouts/` (keymap для SPICE), `blackberry/` | — | Видалити |

### Етап C: вирізати код VNC, SPICE і Opaque (великий рефакторинг)

**Що потрібне лише їм:**

- `bctls`: TLS для VNC (`TLSTunnelBase`, `X509Tunnel`, `SecureVNCPlugin*`)
- `zstd-jni`: декодер VNC Tight-Zstd (`Decoder.java`), тягне за собою власні `.so`
- `snakeyaml`: конфіг Custom VNC (`CustomClientConfigFileReader`)
- `httpcore`: Proxmox/oVirt
- SPICE JNI: `remoteClientLib/jni/android`, `virt-viewer`
- увесь `com.undatech.opaque.proxmox`, `ConnectionSettings` тощо

**Що залишити:**

- `bcprov` і `bcpkix`: SSH-ключі для тунелю (`PubkeyUtils`)
- `sshlib`: SSH-тунель
- `sqlcipher`
- `commons-validator`: використовується лише `InetAddressValidator` у `RdpCommunicator`, можна замінити кількома рядками
- AndroidX, Material, Kotlin

**Нюанс:** код по всьому модулю `:bVNC` перемішаний через перевірки `Utils.isRdp()` / `isVnc()` / `isSpice()` / `isOpaque()`. Різати варто лише після того, як з'явиться CI, який збирає APK, і його можна поставити на планшет і перевірити.

**Альтернатива:** зупинитися на етапах A–B. APK і так стане суттєво меншим, а оновлення з upstream підтягуватимуться без болю.

---

## 4. Збірка в GitHub Actions

### Вимоги

- JDK 21
- Android SDK 36
- Gradle 8.13 через `./gradlew`
- NDK **r25c** і CMake **3.23** для FreeRDP (`build-deps.conf`)
- heap ~6 ГБ (`gradle.properties`)

Android Studio не потрібна.

### Чому не годиться готовий архів `libs-17` (і «Fastest setup» з `CLAUDE.md`)

Перевірено 2026-11-03. `remote-desktop-clients-libs-17.tar.gz`, sha256 `ec916db7e3d89a112ef4b9167e6a3ebf82d52f918de2e80dc9a6c8e56e51010e`:

1. **З актуальним кодом не скомпілюється.** `RdpCommunicator.java:374` викликає `performanceFlags.setGlyphCache(...)`, а в `BookmarkBase` з архіву цього методу немає.
2. **Архів старший за патчі.** Файли датовані січнем–березнем 2025 року. Патчів `05`, `16`, `21` немає, а glyph-cache (`19`/`20`) є лише в ранішому варіанті (`addFlag("glyph-cache", false)`). Є незадокументована зміна `/log-level:DEBUG`, якої немає в жодному патчі. В іншому Java-частина FreeRDP **збігається** з upstream FreeRDP 2.11.7 плюс патчі: нічого шкідливого в різниці немає.
3. **Зібраний з іншими прапорцями,** ніж задає нинішній скрипт: в архіві `FFMPEG=OFF`, `OPENH264=ON`, а в скрипті навпаки.
4. Чи зібрані `.so` саме з цих джерел, перевірити неможливо: збірка не відтворювана. У рядках бінарників сторонніх URL немає. Версії: OpenSSL 1.1.1w, FreeRDP 2.11.7.

### План workflow

1. Checkout і встановлення NDK r25c та CMake 3.23.
2. Клонувати FreeRDP за тегом (2.11.8, див. розділ 2) і застосувати `remoteClientLib/jni/libs/*_freerdp_*.patch` через `patch -N -p1`, як у `build-deps.sh`.
   - Через `git apply` патч `14_freerdp_add_ldflags_to_freerdp_build.patch` на чистий 2.11.7 не накладається. Треба перевірити, чи пройде `patch` з fuzz, і перебазувати патчі на 2.11.8.
3. `scripts/android-build-freerdp.sh` лише для `arm64-v8a`, з `WITH_FFMPEG=0`.
4. `bVNC/prepare_project.sh --skip-build freeaRDP …`, а sqlcipher завантажувати **з перевіркою sha256**.
5. `./gradlew :freeaRDP-app:assembleDebug` → завантажити APK як artifact.
6. Кешувати збірку FreeRDP за ключем «версія FreeRDP + хеш патчів», щоб нативна частина не перезбиралася щоразу.

### Інші нюанси

- **Package name.** Щоб ставити форк поряд з офіційним free aRDP, потрібен інший `applicationId`. Але тип клієнта визначається **за підрядками в імені пакета** (`Utils.isRdp` шукає `rdp`, `Utils.isFree` шукає `free`), тож новий id має містити `rdp` (і `free`, якщо потрібна поведінка безкоштовної версії). Наприклад, `com.example.freeardp`. `namespace` можна лишити як є.
- **Підпис:** поки debug-ключ. Release-ключ, якщо знадобиться, класти в GitHub Secrets.
- **Ліцензія GPL-3.0:** якщо колись публікувати APK, разом з ним має бути відкритий код. Форк і так публічний.

---

## 5. Нотатки з налаштування RDP (не про код)

- Приймати RDP-підключення може лише Windows **Pro / Enterprise / Education**, Home не може.
- **PIN не підходить**, потрібен пароль. Для Microsoft-акаунта це email плюс пароль від акаунта.
  - Якщо в Windows завжди входив лише PIN-ом, один раз виконай `runas /user:MicrosoftAccount\email@example.com cmd`, щоб Windows закешувала пароль.
  - Також вимкни «Дозволити лише вхід через Windows Hello».
- **Ім'я користувача з пробілом** вводиться як є, без лапок. Якщо не пускає: `.\Ім'я Прізвище` або `ІМ'Я-ПК\Ім'я Прізвище`. Назва папки в `C:\Users\` — не логін.
- Домашня мережа має бути позначена як **приватна**. Поза домом підключатися через **Tailscale**, а порт 3389 на роутері не відкривати.
- При першому підключенні звірити відбиток сертифіката. Повторний запит сертифіката для вже знайомого ПК — тривожний знак (див. проблему 3).

---

## 6. Відкриті питання

- [ ] Чи працює Samsung `requestMetaKeyEvent` на поточному One UI? Чи проходять Win і Alt+Tab в офіційному aRDP?
- [ ] Чи бачить `AccessibilityService` «голий» Win і Alt+Tab? (прототип-детектор)
- [ ] FreeRDP 2.11.8 зараз, чи одразу порт на 3.x?
- [ ] OpenSSL 3.x: чи працює NLA/NTLM з `WITH_INTERNAL_MD4/RC4`?
- [ ] Наскільки глибоко різати (етап C) і як це поєднується з merge з upstream?

## Джерела

- [CVE-2026-25941 — Debian Security Tracker](https://security-tracker.debian.org/tracker/CVE-2026-25941)
- [OpenSSL 1.1.1 End Of Life](https://mirror.openssl-library.org/post/2023-09-11-eol-111/)
- [FreeRDP ChangeLog (2.11.8 — останній реліз 2.x)](https://github.com/FreeRDP/FreeRDP/blob/2.11.8/ChangeLog)
- [Windows App: клавіатурні скорочення](https://learn.microsoft.com/en-us/windows-app/input-keyboard-mouse-touch-pen)
- [upstream: iiordanov/remote-desktop-clients](https://github.com/iiordanov/remote-desktop-clients)
