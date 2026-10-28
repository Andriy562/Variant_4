# Практична робота 8 — асинхронний Dart

> Формат: Markdown (.md), кодування UTF-8. Можна відкрити у VS Code або іншому редакторі Markdown та завантажити як документ. Сам файл .md не запускає Flutter-застосунок.
>
> Для запуску використовуйте повний проєкт з архіву `Practical_8_Variant_4_Flutter.zip`. Нижче наведено опис і вихідний код з розподілом за файлами. Платформні файли Android/Web, шрифти та зображення залишаються в архіві. За умовою викладача здається посилання на повний GitHub-репозиторій, а не лише цей документ.

Виконав(ла): ____________________  
Група: ____________________

**Варіант 4: менеджер завантажень.** Дисципліна: «Програмування для мобільних платформ».

Flutter-застосунок імітує завантаження шести файлів із різною швидкістю. Є окремий і пакетний запуск, прогрес у відсотках, скасування, повтор після помилки та підсумок пакета. Усі дані створюються локально: справжніх HTTP-запитів і запису отриманих файлів на диск немає.

## Запуск

Перевірена версія: **Flutter 3.35.5 / Dart 3.9.2**. Проєкт містить платформи Android та Web. Потрібні Flutter SDK і Git; для Android також Android Studio/SDK та емулятор або телефон.

```bash
flutter pub get
flutter analyze
flutter test
flutter run -d chrome
```

Для телефона/емулятора виконайте `flutter devices`, потім `flutter run -d ID_ПРИСТРОЮ`. Для вебзбірки:

```bash
flutter build web --release
```

На головному екрані натисніть **«Відкрити менеджер»**. Мережеві умови задаються у панелі **«Умови мережі»**. Після повернення назад активні операції припиняються; повторний вхід починає новий сеанс.

## Відповідність вимогам

| Вимога | Реалізація |
|---|---|
| Список із 5–7 файлів | 6 файлів: назва, формат, умовний розмір |
| Імітація мережі | `FakeDownloadApi.fetchFiles()` використовує `Future.delayed` |
| Керована частота збоїв | `failureRate`: 0%, 50%, 100%; змінюється в інтерфейсі |
| Разовий запит | `FutureBuilder<List<RemoteFile>>`, поле `_catalog` створюється в `initState`, повтор — у `_reload` |
| Оновлення наживо | `StreamBuilder<DownloadSnapshot>`; прогрес окремої задачі — `Stream<double>` |
| Власні контролери | `DownloadJob` володіє `StreamController<double>`, менеджер — контролером загального стану |
| Паралельний запуск | `DownloadManager.downloadAll()` використовує `Future.wait<DownloadOutcome>` |
| Незалежність помилок | Кожна задача завершується значенням `success`, `failed` або `cancelled` |
| Тайм-аут | Каталог: `Future.timeout(3 с)`; прогрес: `Stream.timeout(2 с)` |
| Скасування | `cancel()` зупиняє таймер, закриває джерело, скасовує підписку; картка повертається до 0% |
| Повтор | Файл із помилкою можна запустити знову з початку |
| Підсумок | Успішні, помилки, скасовані та загальний час через `Stopwatch` |
| Очищення | `DownloadScreen.dispose()` → `DownloadManager.dispose()` → закриття контролерів, скасування таймерів і підписок |
| Безпека після `await` | `_load()` і `_downloadAll()` перевіряють `mounted` |

У пакет включаються лише ще не завершені файли; уже активна задача використовується повторно, не дублюється. Уже успішно завантажені файли не входять у новий підсумок. Під час пакетного запуску можна скасувати окремий файл. Завершення інших файлів продовжується.

## Структура та ключові місця коду

- `lib/main.dart`: тема та головний екран із переходом у менеджер.
- `lib/models/download_models.dart`: моделі файлу, прогресу й результату пакета.
- `lib/services/fake_download_api.dart`: список, `Future.delayed`, частота збоїв; клас `DownloadJob` із таймером і `StreamController<double>`.
- `lib/services/download_manager.dart`: `loadFiles()` — тайм-аут каталогу; `start()` — `Stream.timeout`; `_finish()` — очищення задачі; `downloadAll()` — `Future.wait`; `dispose()` — завершення роботи.
- `lib/screens/download_screen.dart`: кешований Future, обидва builders, стани інтерфейсу, `mounted` і `dispose()`.
- `test/manager_test.dart`: тести сервісів, помилок, повтору та скасування.
- `test/widget_test.dart`: поведінка екрана й вихід під час незавершених операцій.
- `test/screenshots_test.dart`: відтворення реальних Flutter-зображень станів у `docs/`.

`StreamBuilder` сам скасовує свою підписку після видалення віджета. Явні підписки на прогрес кожного файлу належать менеджеру та скасовуються в `_finish()`. `dispose()` синхронно позначає менеджер закритим і зупиняє джерела; асинхронне завершення контролера викликається через `unawaited`.

`Future.timeout` не скасовує початковий `Future.delayed`. Пізня відповідь каталогу не змінює віджет: використовуються перевірка `mounted` і покоління запиту. Після тайм-ауту прогресу додатково закривається сама задача, тому її таймер не продовжує працювати.

## Сценарії захисту

1. **Завантаження й дані.** Установіть 0%, відкрийте менеджер: спільний індикатор зміниться списком шести файлів.
2. **Помилка каталогу.** Установіть 100%, натисніть «Оновити каталог». Після помилки поверніть 0% і натисніть «Повторити».
3. **Порожній каталог.** За частоти 0% увімкніть «Порожній каталог» та оновіть. Вимкніть режим і повторіть запит.
4. **Паралельний прогрес.** Натисніть «Завантажити всі». Різні швидкості видно одночасно; після завершення з’являється підсумок.
5. **Скасування.** Натисніть хрестик на активному файлі. Він повернеться до початкового стану, інші продовжать роботу.
6. **Помилка окремого файлу.** Спочатку відкрийте каталог за 0%, потім установіть 100% та запустіть файл. Поверніть 0% і натисніть «Повторити».
7. **Тайм-аут.** Увімкніть «Зависання першого файлу» й запустіть перший файл. Через 2 с без прогресу з’явиться помилка. Вимкніть зависання й повторіть.
8. **Частковий успіх.** Запустіть пакет за 50%. Збої випадкові: окремий запуск може дати різне співвідношення результатів, що є нормальною поведінкою.
9. **Збій потоку стану.** Натисніть «Збій потоку», потім «Повторити». Каталог зберігається; нова подія також автоматично прибирає помилку.
10. **Вихід.** Під час завантаження поверніться на головний екран. Усі активні задачі скасовуються; помилок після `dispose` бути не повинно.

Частота збоїв перевіряється один раз для кожного запиту/нового завантаження. У режимі 50% це не ймовірність збою на кожному тіку. Нове налаштування застосовується до наступних операцій. «Оновити каталог» скидає поточний сеанс; під час пакета ця кнопка вимкнена.

## Скриншоти

Вісім скриншотів станів містяться в каталозі `docs/` повного проєкту в архіві `Practical_8_Variant_4_Flutter.zip`. У цей текстовий файл зображення не вбудовано.

## Перевірка

Результати фінальної перевірки наведено в `docs/verification.md`.
Android-проєкт включено, але Android SDK у середовищі перевірки відсутній, тому APK тут не збирався. Вебзбірка й Flutter-тести перевіряються окремо; їх не слід видавати за запуск на фізичному телефоні.

## Публікація й здача

Для кожної практичної достатньо окремого репозиторію у вашому наявному GitHub-акаунті. Другий профіль і повторне встановлення GitHub Desktop не потрібні.

1. Створіть **порожній** репозиторій `practical-8-download-manager` у своєму акаунті, без автоматичного README та .gitignore.
2. Розпакуйте комплект зі збереженою папкою `.git`. У GitHub Desktop виберіть **File → Add local repository**, укажіть папку проєкту та опублікуйте її. Альтернатива — команди нижче.
3. Після публікації перевірте README, `docs/` та історію комітів.
4. У Classroom здайте URL репозиторію. ZIP не замінює посилання за умовою завдання.

```bash
git remote add origin https://github.com/YOUR_USERNAME/practical-8-download-manager.git
git push -u origin main
```

Замініть `YOUR_USERNAME` своїм логіном. Якщо `origin` уже заданий, перевірте його через `git remote -v`; не перезаписуйте чужий проєкт. Завантаження всіх файлів одним кліком через сайт **не переносить** підготовлену історію комітів. Не запускайте `git init` повторно поверх комплекту.

Якщо розпаковувач пропустив `.git`, використайте доданий поруч `download_manager.bundle`:

```bash
git clone download_manager.bundle practical-8-download-manager
cd practical-8-download-manager
git remote remove origin
git remote add origin https://github.com/YOUR_USERNAME/practical-8-download-manager.git
git push -u origin main
```

Автор локальних комітів — нейтральний `Coursework Author`; особистий email користувача не вигадувався. Для наступних власних комітів налаштуйте `git config user.name` та `git config user.email`.

## Джерела

- [Dart: асинхронне програмування](https://dart.dev/libraries/async/async-await)
- [Dart: створення потоків](https://dart.dev/libraries/async/creating-streams)
- [FutureBuilder](https://api.flutter.dev/flutter/widgets/FutureBuilder-class.html)
- [StreamBuilder](https://api.flutter.dev/flutter/widgets/StreamBuilder-class.html)
- [Stream.timeout](https://api.dart.dev/dart-async/Stream/timeout.html)
- [Future.wait](https://api.dart.dev/dart-async/Future/wait.html)

Шрифт DejaVu розповсюджується з ліцензією в `assets/FONT_LICENSE.txt`.


## Вихідний код проєкту

Кожний блок потрібно зберігати в окремому файлі за вказаним шляхом від кореня проєкту.

### `pubspec.yaml`

````yaml
name: download_manager
description: Practical work 8, variant 4. Simulated asynchronous downloads.
publish_to: none
version: 1.0.0+1
environment:
  sdk: '>=3.9.0 <4.0.0'
dependencies:
  flutter:
    sdk: flutter
dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0
flutter:
  uses-material-design: true
  fonts:
    - family: DejaVu
      fonts:
        - asset: assets/DejaVuSans.ttf
        - asset: assets/DejaVuSans-Bold.ttf
          weight: 700
````

### `analysis_options.yaml`

````yaml
include: package:flutter_lints/flutter.yaml
linter:
  rules:
    use_build_context_synchronously: true
    unawaited_futures: true
````

### `lib/main.dart`

````dart
import 'package:flutter/material.dart';
import 'screens/download_screen.dart';

void main() => runApp(const DownloadApp());

class DownloadApp extends StatelessWidget {
  const DownloadApp({super.key, this.home});
  final Widget? home;
  @override
  Widget build(BuildContext context) => MaterialApp(
    title: 'Завантаження',
    debugShowCheckedModeBanner: false,
    theme: ThemeData(
      useMaterial3: true,
      fontFamily: 'DejaVu',
      colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xff147d70)),
      scaffoldBackgroundColor: const Color(0xfff3f6f5),
    ),
    home: home ?? const HomeScreen(),
  );
}

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
    body: Center(
      child: Padding(
        padding: const EdgeInsets.all(28),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Icon(
              Icons.cloud_download_outlined,
              size: 76,
              color: Color(0xff147d70),
            ),
            const SizedBox(height: 24),
            Text(
              'Усі матеріали — в одному місці',
              textAlign: TextAlign.center,
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 12),
            const Text(
              'Практична 8 • Варіант 4\nСимуляція асинхронних завантажень',
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 24),
            FilledButton.icon(
              icon: const Icon(Icons.arrow_forward),
              onPressed: () => Navigator.of(context).push<void>(
                MaterialPageRoute(builder: (_) => const DownloadScreen()),
              ),
              label: const Text('Відкрити менеджер'),
            ),
          ],
        ),
      ),
    ),
  );
}
````

### `lib/models/download_models.dart`

````dart
enum DownloadStatus { idle, running, completed, failed }

enum DownloadOutcome { success, failed, cancelled }

class RemoteFile {
  const RemoteFile(this.id, this.name, this.megabytes, this.kind);
  final String id;
  final String name;
  final double megabytes;
  final String kind;
}

class FileProgress {
  const FileProgress({
    this.status = DownloadStatus.idle,
    this.fraction = 0,
    this.error,
  });
  final DownloadStatus status;
  final double fraction;
  final String? error;
}

class DownloadSnapshot {
  const DownloadSnapshot(this.files);
  final Map<String, FileProgress> files;
  FileProgress progress(String id) => files[id] ?? const FileProgress();
}

class BatchSummary {
  const BatchSummary({
    required this.success,
    required this.failed,
    required this.cancelled,
    required this.elapsed,
  });
  final int success;
  final int failed;
  final int cancelled;
  final Duration elapsed;
}
````

### `lib/screens/download_screen.dart`

````dart
import 'dart:async';
import 'package:flutter/material.dart';
import '../models/download_models.dart';
import '../services/download_manager.dart';

class DownloadScreen extends StatefulWidget {
  const DownloadScreen({super.key, this.manager});
  final DownloadManager? manager;
  @override
  State<DownloadScreen> createState() => _DownloadScreenState();
}

class _DownloadScreenState extends State<DownloadScreen> {
  late final DownloadManager _manager;
  late final Stream<DownloadSnapshot> _stream;
  late Future<List<RemoteFile>> _catalog;
  bool _batchRunning = false;
  BatchSummary? _summary;
  int _generation = 0;

  @override
  void initState() {
    super.initState();
    _manager = widget.manager ?? DownloadManager();
    _stream = _manager.stream;
    _catalog = _load();
  }

  Future<List<RemoteFile>> _load() async {
    final generation = ++_generation;
    final files = await _manager.loadFiles();
    if (!mounted || generation != _generation) return files;
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (mounted && generation == _generation) _manager.retryUpdates();
    });
    return files;
  }

  void _reload() => setState(() {
    _summary = null;
    _catalog = _load();
  });

  Future<void> _downloadAll(List<RemoteFile> files) async {
    if (_batchRunning) return;
    setState(() {
      _batchRunning = true;
      _summary = null;
    });
    final summary = await _manager.downloadAll(files);
    if (!mounted) return;
    setState(() {
      _batchRunning = false;
      _summary = summary;
    });
  }

  @override
  void dispose() {
    _generation++;
    // Cancels every generating timer and progress subscription immediately.
    // StreamBuilder owns and cancels its UI subscription automatically.
    unawaited(_manager.dispose());
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Завантаження')),
    body: SafeArea(
      child: Center(
        child: ConstrainedBox(
          constraints: const BoxConstraints(maxWidth: 760),
          child: Column(
            children: [
              _settings(),
              Expanded(
                child: FutureBuilder<List<RemoteFile>>(
                  future: _catalog,
                  builder: (context, snapshot) {
                    if (snapshot.connectionState != ConnectionState.done) {
                      return const StatePanel(
                        icon: Icons.cloud_download_outlined,
                        title: 'Отримуємо каталог',
                        detail: 'Готуємо список доступних матеріалів…',
                        loading: true,
                      );
                    }
                    if (snapshot.hasError) {
                      return StatePanel(
                        icon: Icons.cloud_off_outlined,
                        title: 'Каталог недоступний',
                        detail: snapshot.error is TimeoutException
                            ? 'Сервер не відповів за 3 секунди.'
                            : 'Зменште частоту збоїв і спробуйте ще раз.',
                        action: _reload,
                      );
                    }
                    final files = snapshot.data ?? [];
                    if (files.isEmpty) {
                      return StatePanel(
                        icon: Icons.folder_open,
                        title: 'Поки що немає файлів',
                        detail:
                            'Каталог порожній. Вимкніть порожній режим і повторіть запит.',
                        action: _reload,
                      );
                    }
                    return _downloads(files);
                  },
                ),
              ),
            ],
          ),
        ),
      ),
    ),
  );

  Widget _settings() => Padding(
    padding: const EdgeInsets.fromLTRB(16, 4, 16, 8),
    child: Card(
      margin: EdgeInsets.zero,
      clipBehavior: Clip.antiAlias,
      child: ExpansionTile(
        leading: const Icon(Icons.tune),
        title: const Text(
          'Умови мережі',
          style: TextStyle(fontSize: 14, fontWeight: FontWeight.w700),
        ),
        subtitle: Text(
          'Симуляція • частота збоїв ${(_manager.api.failureRate * 100).round()}%',
          style: const TextStyle(fontSize: 12),
        ),
        childrenPadding: const EdgeInsets.fromLTRB(16, 0, 16, 12),
        children: [
          Wrap(
            spacing: 8,
            children: [
              for (final value in [0.0, 0.5, 1.0])
                ChoiceChip(
                  label: Text('${(value * 100).round()}%'),
                  selected: _manager.api.failureRate == value,
                  onSelected: (_) => setState(() {
                    _manager.api.failureRate = value;
                  }),
                ),
            ],
          ),
          SwitchListTile(
            dense: true,
            contentPadding: EdgeInsets.zero,
            title: const Text('Порожній каталог'),
            value: _manager.api.emptyCatalog,
            onChanged: (v) => setState(() {
              _manager.api.emptyCatalog = v;
            }),
          ),
          SwitchListTile(
            dense: true,
            contentPadding: EdgeInsets.zero,
            title: const Text('Зависання першого файлу'),
            value: _manager.api.stallFirstFile,
            onChanged: (v) => setState(() {
              _manager.api.stallFirstFile = v;
            }),
          ),
          Wrap(
            spacing: 8,
            children: [
              OutlinedButton(
                onPressed: _batchRunning ? null : _reload,
                child: const Text('Оновити каталог'),
              ),
              TextButton(
                onPressed: _manager.simulateStreamError,
                child: const Text('Збій потоку'),
              ),
            ],
          ),
          const Text(
            'Нові умови діють на наступний запит. Оновлення каталогу скасовує активні завантаження.',
            style: TextStyle(fontSize: 11),
          ),
        ],
      ),
    ),
  );

  Widget _downloads(List<RemoteFile> files) => StreamBuilder<DownloadSnapshot>(
    stream: _stream,
    builder: (context, snapshot) {
      if (!snapshot.hasData && !snapshot.hasError) {
        return const StatePanel(
          icon: Icons.sync,
          title: 'Підключаємо прогрес',
          detail: 'Очікуємо першу подію потоку…',
          loading: true,
        );
      }
      final state = snapshot.data ?? _manager.snapshot;
      final completed = files
          .where((f) => state.progress(f.id).status == DownloadStatus.completed)
          .length;
      return Column(
        children: [
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 20, vertical: 10),
            child: Row(
              children: [
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      const Text(
                        'Ваші матеріали',
                        style: TextStyle(
                          fontSize: 20,
                          fontWeight: FontWeight.w700,
                        ),
                      ),
                      const SizedBox(height: 4),
                      Text(
                        '${files.length} файлів · готово $completed',
                        style: const TextStyle(
                          fontSize: 12,
                          color: Color(0xff657670),
                        ),
                      ),
                    ],
                  ),
                ),
                FilledButton.icon(
                  onPressed: _batchRunning || completed == files.length
                      ? null
                      : () => unawaited(_downloadAll(files)),
                  icon: Icon(
                    _batchRunning ? Icons.sync : Icons.download,
                    size: 18,
                  ),
                  label: Text(
                    _batchRunning ? 'Триває…' : 'Завантажити всі',
                    style: const TextStyle(fontSize: 12),
                  ),
                ),
              ],
            ),
          ),
          if (snapshot.hasError)
            Container(
              margin: const EdgeInsets.symmetric(horizontal: 16),
              padding: const EdgeInsets.all(10),
              color: const Color(0xffffe9dd),
              child: Row(
                children: [
                  const Expanded(
                    child: Text(
                      'Збій оновлень. Завантаження не втрачені.',
                      style: TextStyle(fontSize: 12),
                    ),
                  ),
                  TextButton(
                    onPressed: _manager.retryUpdates,
                    child: const Text('Повторити'),
                  ),
                ],
              ),
            ),
          if (_summary != null) _summaryCard(_summary!),
          Expanded(
            child: ListView.builder(
              padding: const EdgeInsets.fromLTRB(16, 8, 16, 16),
              itemCount: files.length,
              itemBuilder: (_, index) =>
                  _fileCard(files[index], state.progress(files[index].id)),
            ),
          ),
        ],
      );
    },
  );

  Widget _summaryCard(BatchSummary summary) => Container(
    key: const Key('batchSummary'),
    margin: const EdgeInsets.fromLTRB(16, 0, 16, 8),
    padding: const EdgeInsets.all(14),
    decoration: BoxDecoration(
      color: const Color(0xffe2efe9),
      borderRadius: BorderRadius.circular(14),
    ),
    child: Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const Text(
          'Підсумок завантаження',
          style: TextStyle(fontWeight: FontWeight.w700),
        ),
        const SizedBox(height: 6),
        Text(
          'Успішно: ${summary.success} · Помилки: ${summary.failed}\n'
          'Скасовано: ${summary.cancelled} · Час: ${(summary.elapsed.inMilliseconds / 1000).toStringAsFixed(1)} с',
          style: const TextStyle(fontSize: 12, height: 1.5),
        ),
      ],
    ),
  );

  Widget _fileCard(RemoteFile file, FileProgress progress) {
    final running = progress.status == DownloadStatus.running;
    final done = progress.status == DownloadStatus.completed;
    final failed = progress.status == DownloadStatus.failed;
    final color = failed ? const Color(0xffb24936) : const Color(0xff147d70);
    final label = switch (progress.status) {
      DownloadStatus.idle => 'Готовий до завантаження',
      DownloadStatus.running => 'Завантаження…',
      DownloadStatus.completed => 'Завантажено',
      DownloadStatus.failed => progress.error ?? 'Помилка завантаження',
    };
    return Container(
      key: ValueKey('file-${file.id}'),
      margin: const EdgeInsets.only(bottom: 12),
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Colors.white,
        borderRadius: BorderRadius.circular(18),
        border: Border.all(
          color: failed ? const Color(0xffe8b5a8) : const Color(0xffdfe8e3),
        ),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            children: [
              Container(
                width: 45,
                height: 45,
                alignment: Alignment.center,
                decoration: BoxDecoration(
                  color: const Color(0xffedf3ef),
                  borderRadius: BorderRadius.circular(12),
                ),
                child: Text(
                  file.kind,
                  style: TextStyle(
                    fontSize: 11,
                    color: color,
                    fontWeight: FontWeight.w700,
                  ),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      file.name,
                      style: const TextStyle(
                        fontWeight: FontWeight.w700,
                        fontSize: 13,
                      ),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      '${file.megabytes.toStringAsFixed(1)} МБ',
                      style: const TextStyle(
                        fontSize: 12,
                        color: Color(0xff708079),
                      ),
                    ),
                  ],
                ),
              ),
              if (done)
                const Icon(Icons.check_circle, color: Color(0xff147d70))
              else
                IconButton(
                  key: ValueKey('${running ? 'cancel' : 'start'}-${file.id}'),
                  tooltip: running
                      ? 'Скасувати'
                      : failed
                      ? 'Повторити'
                      : 'Завантажити',
                  onPressed: running
                      ? () => _manager.cancel(file.id)
                      : () => unawaited(_manager.start(file)),
                  icon: Icon(
                    running
                        ? Icons.close
                        : failed
                        ? Icons.refresh
                        : Icons.download,
                    color: color,
                  ),
                ),
            ],
          ),
          const SizedBox(height: 14),
          LinearProgressIndicator(
            value: progress.fraction,
            minHeight: 5,
            color: color,
            backgroundColor: const Color(0xffedf1ee),
            borderRadius: BorderRadius.circular(4),
          ),
          const SizedBox(height: 8),
          Row(
            children: [
              Expanded(
                child: Text(
                  label,
                  style: TextStyle(fontSize: 11, color: color),
                ),
              ),
              Text(
                '${(progress.fraction * 100).round()}%',
                style: TextStyle(fontSize: 12, color: color),
              ),
            ],
          ),
          if (failed)
            TextButton.icon(
              onPressed: () => unawaited(_manager.start(file)),
              icon: const Icon(Icons.refresh, size: 16),
              label: const Text('Повторити'),
            ),
        ],
      ),
    );
  }
}

class StatePanel extends StatelessWidget {
  const StatePanel({
    super.key,
    required this.icon,
    required this.title,
    required this.detail,
    this.loading = false,
    this.action,
  });
  final IconData icon;
  final String title;
  final String detail;
  final bool loading;
  final VoidCallback? action;
  @override
  Widget build(BuildContext context) => Center(
    child: SingleChildScrollView(
      padding: const EdgeInsets.all(28),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(icon, size: 56, color: const Color(0xff6c9989)),
          const SizedBox(height: 20),
          Text(
            title,
            textAlign: TextAlign.center,
            style: const TextStyle(fontSize: 22, fontWeight: FontWeight.w700),
          ),
          const SizedBox(height: 12),
          Text(
            detail,
            textAlign: TextAlign.center,
            style: const TextStyle(height: 1.5, color: Color(0xff6b7b73)),
          ),
          const SizedBox(height: 24),
          if (loading)
            const SizedBox(
              width: 28,
              height: 28,
              child: CircularProgressIndicator(),
            ),
          if (action != null)
            FilledButton(onPressed: action, child: const Text('Повторити')),
        ],
      ),
    ),
  );
}
````

### `lib/services/download_manager.dart`

````dart
import 'dart:async';
import '../models/download_models.dart';
import 'fake_download_api.dart';

class DownloadManager {
  DownloadManager({
    FakeDownloadApi? api,
    this.idleTimeout = const Duration(seconds: 2),
    this.catalogTimeout = const Duration(seconds: 3),
  }) : api = api ?? FakeDownloadApi();
  final FakeDownloadApi api;
  final Duration idleTimeout;
  final Duration catalogTimeout;
  final StreamController<DownloadSnapshot> _controller =
      StreamController<DownloadSnapshot>.broadcast();
  final Map<String, FileProgress> _states = {};
  final Map<String, _ActiveDownload> _active = {};
  bool _disposed = false;
  int _generation = 0;
  Stream<DownloadSnapshot> get stream => _controller.stream;
  DownloadSnapshot get snapshot => DownloadSnapshot(Map.unmodifiable(_states));
  int get activeCount => _active.length;
  bool get isDisposed => _disposed;

  Future<List<RemoteFile>> loadFiles() async {
    final generation = ++_generation;
    for (final id in _active.keys.toList()) {
      cancel(id);
    }
    _states.clear();
    _emit();
    final files = await api.fetchFiles().timeout(catalogTimeout);
    if (_disposed || generation != _generation) return files;
    for (final file in files) {
      _states[file.id] = const FileProgress();
    }
    _emit();
    return files;
  }

  Future<DownloadOutcome> start(RemoteFile file) {
    if (_disposed) return Future.value(DownloadOutcome.cancelled);
    final existing = _active[file.id];
    if (existing != null) return existing.result.future;
    if (_states[file.id]?.status == DownloadStatus.completed) {
      return Future.value(DownloadOutcome.success);
    }
    final job = api.createJob(file);
    final active = _ActiveDownload(job);
    _active[file.id] = active;
    _states[file.id] = const FileProgress(status: DownloadStatus.running);
    _emit();
    active.subscription = job.progress
        .timeout(idleTimeout)
        .listen(
          (value) {
            if (_disposed || _active[file.id] != active) return;
            _states[file.id] = FileProgress(
              status: DownloadStatus.running,
              fraction: value,
            );
            _emit();
            if (value >= 1) _finish(file.id, DownloadOutcome.success);
          },
          onError: (Object error, StackTrace stack) {
            _finish(
              file.id,
              DownloadOutcome.failed,
              error is TimeoutException
                  ? 'Немає прогресу протягом ${idleTimeout.inSeconds} с'
                  : 'Перервано з’єднання з сервером',
            );
          },
          onDone: () {
            if (_active[file.id] == active) {
              _finish(
                file.id,
                DownloadOutcome.failed,
                'Потік завершився до отримання файлу',
              );
            }
          },
          cancelOnError: false,
        );
    return active.result.future;
  }

  void _finish(String id, DownloadOutcome outcome, [String? error]) {
    final active = _active.remove(id);
    if (active == null) return;
    // close() stops the generator synchronously, before awaiting close completion.
    unawaited(active.job.close());
    unawaited(active.subscription?.cancel() ?? Future<void>.value());
    switch (outcome) {
      case DownloadOutcome.success:
        _states[id] = const FileProgress(
          status: DownloadStatus.completed,
          fraction: 1,
        );
      case DownloadOutcome.failed:
        _states[id] = FileProgress(
          status: DownloadStatus.failed,
          fraction: _states[id]?.fraction ?? 0,
          error: error,
        );
      case DownloadOutcome.cancelled:
        _states[id] = const FileProgress();
    }
    _emit();
    active.result.complete(outcome);
  }

  void cancel(String id) => _finish(id, DownloadOutcome.cancelled);

  Future<BatchSummary> downloadAll(List<RemoteFile> files) async {
    final stopwatch = Stopwatch()..start();
    // Each task resolves to an outcome, so one error cannot discard other results.
    final results = await Future.wait<DownloadOutcome>([
      for (final file in files)
        if (_states[file.id]?.status != DownloadStatus.completed) start(file),
    ]);
    stopwatch.stop();
    return BatchSummary(
      success: results.where((r) => r == DownloadOutcome.success).length,
      failed: results.where((r) => r == DownloadOutcome.failed).length,
      cancelled: results.where((r) => r == DownloadOutcome.cancelled).length,
      elapsed: stopwatch.elapsed,
    );
  }

  void _emit() {
    if (!_disposed) _controller.add(snapshot);
  }

  void simulateStreamError() {
    if (!_disposed) _controller.addError(StateError('Збій каналу оновлень'));
  }

  void retryUpdates() => _emit();

  Future<void> dispose() async {
    if (_disposed) return;
    _disposed = true;
    _generation++;
    for (final id in _active.keys.toList()) {
      cancel(id);
    }
    await _controller.close();
  }
}

class _ActiveDownload {
  _ActiveDownload(this.job);
  final DownloadJob job;
  final Completer<DownloadOutcome> result = Completer<DownloadOutcome>();
  StreamSubscription<double>? subscription;
}
````

### `lib/services/fake_download_api.dart`

````dart
import 'dart:async';
import 'dart:math';
import '../models/download_models.dart';

/// No actual HTTP requests or files. All data is generated in memory.
class FakeDownloadApi {
  FakeDownloadApi({
    this.failureRate = 0,
    Random? random,
    this.catalogDelay = const Duration(milliseconds: 900),
    this.tick = const Duration(milliseconds: 200),
  }) : _random = random ?? Random();
  final Random _random;
  final Duration catalogDelay;
  final Duration tick;
  double failureRate;
  bool emptyCatalog = false;
  bool stallFirstFile = false;

  Future<List<RemoteFile>> fetchFiles() async {
    final fail = _random.nextDouble() < failureRate;
    final empty = emptyCatalog;
    await Future<void>.delayed(catalogDelay);
    if (fail) throw StateError('Каталог тимчасово недоступний');
    if (empty) return [];
    return const [
      RemoteFile('1', 'Навчальний посібник.pdf', 12.4, 'PDF'),
      RemoteFile('2', 'Приклади коду.zip', 28.8, 'ZIP'),
      RemoteFile('3', 'Відеолекція.mp4', 164.0, 'MP4'),
      RemoteFile('4', 'Презентація.pptx', 8.6, 'PPTX'),
      RemoteFile('5', 'Макети інтерфейсу.fig', 42.1, 'FIG'),
      RemoteFile('6', 'Матеріали практики.zip', 19.7, 'ZIP'),
    ];
  }

  DownloadJob createJob(RemoteFile file) => DownloadJob(
    tick: tick,
    totalTicks: 10 + int.parse(file.id) * 3,
    fail: _random.nextDouble() < failureRate,
    stall: stallFirstFile && file.id == '1',
  );
}

/// Owns a single-subscription progress stream and the generating timer.
class DownloadJob {
  DownloadJob({
    required this.tick,
    required this.totalTicks,
    this.fail = false,
    this.stall = false,
  }) {
    _controller = StreamController<double>(
      onListen: _start,
      onCancel: _stopTimer,
    );
  }
  final Duration tick;
  final int totalTicks;
  final bool fail;
  final bool stall;
  late final StreamController<double> _controller;
  Timer? _timer;
  int _step = 0;
  bool _closed = false;
  int get emittedTicks => _step;
  bool get isStopped => _closed && !(_timer?.isActive ?? false);
  Stream<double> get progress => _controller.stream;

  void _start() {
    if (_closed) return;
    _controller.add(0);
    _timer = Timer.periodic(tick, (_) {
      if (_closed || stall) return;
      _step++;
      if (fail && _step == 4) {
        _controller.addError(StateError('З’єднання з сервером перервано'));
        unawaited(close());
        return;
      }
      _controller.add((_step / totalTicks).clamp(0, 1).toDouble());
      if (_step >= totalTicks) unawaited(close());
    });
  }

  void _stopTimer() {
    _timer?.cancel();
  }

  Future<void> close() async {
    if (_closed) return;
    _closed = true;
    _stopTimer();
    await _controller.close();
  }
}
````

### `test/manager_test.dart`

````dart
import 'dart:async';
import 'package:flutter_test/flutter_test.dart';
import 'package:download_manager/models/download_models.dart';
import 'package:download_manager/services/download_manager.dart';
import 'package:download_manager/services/fake_download_api.dart';

class SelectiveApi extends FakeDownloadApi {
  SelectiveApi()
    : super(catalogDelay: Duration.zero, tick: const Duration(milliseconds: 2));
  @override
  DownloadJob createJob(RemoteFile file) => DownloadJob(
    tick: tick,
    totalTicks: 8 + int.parse(file.id),
    fail: file.id == '2',
  );
}

void main() {
  test('catalog has six files, empty mode and controlled failure', () async {
    final api = FakeDownloadApi(catalogDelay: Duration.zero);
    expect(await api.fetchFiles(), hasLength(6));
    api.emptyCatalog = true;
    expect(await api.fetchFiles(), isEmpty);
    api.failureRate = 1;
    await expectLater(api.fetchFiles(), throwsStateError);
  });

  test(
    'one failed file does not interrupt five successful downloads',
    () async {
      final manager = DownloadManager(api: SelectiveApi());
      final files = await manager.loadFiles();
      final result = await manager.downloadAll(files);
      expect(result.success, 5);
      expect(result.failed, 1);
      expect(result.cancelled, 0);
      expect(result.elapsed.inMicroseconds, greaterThan(0));
      expect(manager.activeCount, 0);
      await manager.dispose();
    },
  );

  test('cancellation really stops generator and resets progress', () async {
    final manager = DownloadManager(
      api: FakeDownloadApi(
        catalogDelay: Duration.zero,
        tick: const Duration(milliseconds: 10),
      ),
    );
    final files = await manager.loadFiles();
    final result = manager.start(files.first);
    await Future<void>.delayed(const Duration(milliseconds: 35));
    manager.cancel(files.first.id);
    expect(await result, DownloadOutcome.cancelled);
    expect(manager.snapshot.progress('1').status, DownloadStatus.idle);
    expect(manager.snapshot.progress('1').fraction, 0);
    await Future<void>.delayed(const Duration(milliseconds: 40));
    expect(manager.snapshot.progress('1').fraction, 0);
    expect(manager.activeCount, 0);
    await manager.dispose();
  });

  test('Stream.timeout fails stalled download and retry can succeed', () async {
    final api = FakeDownloadApi(
      catalogDelay: Duration.zero,
      tick: const Duration(milliseconds: 2),
    )..stallFirstFile = true;
    final manager = DownloadManager(
      api: api,
      idleTimeout: const Duration(milliseconds: 40),
    );
    final files = await manager.loadFiles();
    expect(await manager.start(files.first), DownloadOutcome.failed);
    expect(manager.snapshot.progress('1').error, contains('Немає прогресу'));
    expect(manager.activeCount, 0);
    api.stallFirstFile = false;
    expect(await manager.start(files.first), DownloadOutcome.success);
    expect(manager.snapshot.progress('1').fraction, 1);
    await manager.dispose();
  });

  test('dispose cancels all running tasks and resolves batch future', () async {
    final manager = DownloadManager(
      api: FakeDownloadApi(catalogDelay: Duration.zero),
    );
    final files = await manager.loadFiles();
    final batch = manager.downloadAll(files);
    await manager.dispose();
    final result = await batch;
    expect(result.cancelled, 6);
    expect(manager.activeCount, 0);
    expect(manager.isDisposed, isTrue);
  });

  test('job timer stops after cancelling its source', () async {
    final job = DownloadJob(
      tick: const Duration(milliseconds: 5),
      totalTicks: 100,
    );
    final subscription = job.progress.listen((_) {});
    await Future<void>.delayed(const Duration(milliseconds: 20));
    await subscription.cancel();
    await job.close();
    final emitted = job.emittedTicks;
    await Future<void>.delayed(const Duration(milliseconds: 20));
    expect(job.emittedTicks, emitted);
    expect(job.isStopped, isTrue);
  });

  test('catalog has a timeout', () async {
    final manager = DownloadManager(
      api: FakeDownloadApi(catalogDelay: const Duration(milliseconds: 50)),
      catalogTimeout: const Duration(milliseconds: 10),
    );
    await expectLater(manager.loadFiles(), throwsA(isA<TimeoutException>()));
    await manager.dispose();
    await Future<void>.delayed(const Duration(milliseconds: 60));
  });
}
````

### `test/screenshots_test.dart`

````dart
// Actual Flutter raster output, not mockups. Run separately to regenerate docs.
import 'dart:io';
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:download_manager/main.dart';
import 'package:download_manager/screens/download_screen.dart';
import 'package:download_manager/services/download_manager.dart';
import 'package:download_manager/services/fake_download_api.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();
  setUpAll(() async {
    final regular = FontLoader('DejaVu')
      ..addFont(rootBundle.load('assets/DejaVuSans.ttf'));
    await regular.load();
    // Material icons are bundled by Flutter, explicitly load for test rendering.
    final icons = FontLoader('MaterialIcons')
      ..addFont(rootBundle.load('fonts/MaterialIcons-Regular.otf'));
    await icons.load();
  });

  testWidgets('capture required application states', (tester) async {
    tester.view.physicalSize = const Size(500, 1280);
    tester.view.devicePixelRatio = 1;
    addTearDown(tester.view.resetPhysicalSize);
    addTearDown(tester.view.resetDevicePixelRatio);
    final boundaryKey = GlobalKey();
    final api = FakeDownloadApi();
    final manager = DownloadManager(api: api);
    await tester.pumpWidget(
      RepaintBoundary(
        key: boundaryKey,
        child: DownloadApp(home: DownloadScreen(manager: manager)),
      ),
    );

    Future<void> capture(String name) async {
      await tester.pump();
      final boundary =
          boundaryKey.currentContext!.findRenderObject()!
              as RenderRepaintBoundary;
      await tester.runAsync(() async {
        final image = await boundary.toImage(pixelRatio: 1);
        final bytes = await image.toByteData(format: ui.ImageByteFormat.png);
        await Directory('docs').create(recursive: true);
        await File('docs/$name.png').writeAsBytes(bytes!.buffer.asUint8List());
        image.dispose();
      });
    }

    await capture('01_loading');
    await tester.pump(const Duration(seconds: 1));
    await tester.pump();
    await tester.pump();
    expect(find.text('Ваші матеріали'), findsOneWidget);
    await capture('04_data');
    await tester.tap(find.text('Завантажити всі'));
    await tester.pump(const Duration(milliseconds: 800));
    await capture('05_realtime');
    await tester.pump(const Duration(milliseconds: 800));
    await capture('06_realtime_later');
    await tester.pump(const Duration(seconds: 5));
    await tester.pump();
    await capture('07_summary');
    // Reload through the same service controls exposed in the UI.
    await tester.tap(find.text('Умови мережі'));
    await tester.pump();
    await tester.pump(const Duration(milliseconds: 400));
    api.failureRate = 1;
    await tester.tap(find.text('Оновити каталог'));
    await tester.pump();
    await tester.pump(const Duration(seconds: 1));
    await tester.pump();
    await tester.tap(find.text('Умови мережі'));
    await tester.pump();
    await tester.pump(const Duration(milliseconds: 400));
    expect(find.text('Каталог недоступний'), findsOneWidget);
    await capture('02_error');
    api.failureRate = 0;
    api.emptyCatalog = true;
    await tester.tap(find.text('Повторити'));
    await tester.pump();
    await tester.pump(const Duration(seconds: 1));
    await tester.pump();
    await capture('03_empty');
    api.emptyCatalog = false;
    await tester.tap(find.text('Повторити'));
    await tester.pump();
    await tester.pump(const Duration(seconds: 1));
    await tester.pump();
    await tester.pump();
    api.stallFirstFile = true;
    await tester.tap(find.byKey(const ValueKey('start-1')));
    await tester.pump(const Duration(seconds: 3));
    await tester.pump();
    await capture('08_timeout');
    await tester.pumpWidget(const SizedBox.shrink());
    await tester.pump();
    expect(tester.takeException(), isNull);
  });
}
````

### `test/widget_test.dart`

````dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:download_manager/main.dart';
import 'package:download_manager/screens/download_screen.dart';
import 'package:download_manager/services/download_manager.dart';
import 'package:download_manager/services/fake_download_api.dart';

void main() {
  testWidgets('loading, catalog, progress, cancel and widget disposal', (
    tester,
  ) async {
    final manager = DownloadManager();
    await tester.pumpWidget(
      DownloadApp(home: DownloadScreen(manager: manager)),
    );
    expect(find.text('Отримуємо каталог'), findsOneWidget);
    await tester.pump(const Duration(seconds: 1));
    await tester.pump();
    await tester.pump();
    expect(find.text('Ваші матеріали'), findsOneWidget);
    await tester.tap(find.byKey(const ValueKey('start-1')));
    await tester.pump(const Duration(milliseconds: 500));
    expect(manager.activeCount, 1);
    await tester.tap(find.byKey(const ValueKey('cancel-1')));
    await tester.pump();
    expect(manager.activeCount, 0);
    await tester.tap(find.byKey(const ValueKey('start-1')));
    await tester.pump();
    await tester.pumpWidget(const SizedBox.shrink());
    await tester.pump(const Duration(seconds: 5));
    expect(manager.isDisposed, isTrue);
    expect(manager.activeCount, 0);
    expect(tester.takeException(), isNull);
  });

  testWidgets('catalog error can be retried as an empty catalog', (
    tester,
  ) async {
    final api = FakeDownloadApi(failureRate: 1);
    final manager = DownloadManager(api: api);
    await tester.pumpWidget(
      DownloadApp(home: DownloadScreen(manager: manager)),
    );
    await tester.pump(const Duration(seconds: 1));
    await tester.pump();
    expect(find.text('Каталог недоступний'), findsOneWidget);
    api.failureRate = 0;
    api.emptyCatalog = true;
    await tester.tap(find.text('Повторити'));
    await tester.pump();
    await tester.pump(const Duration(seconds: 1));
    await tester.pump();
    expect(find.text('Поки що немає файлів'), findsOneWidget);
    await tester.pumpWidget(const SizedBox.shrink());
    await tester.pump();
  });

  testWidgets('leaving while catalog future is pending is safe', (
    tester,
  ) async {
    final manager = DownloadManager();
    await tester.pumpWidget(
      DownloadApp(home: DownloadScreen(manager: manager)),
    );
    await tester.pumpWidget(const SizedBox.shrink());
    await tester.pump(const Duration(seconds: 4));
    expect(manager.isDisposed, isTrue);
    expect(tester.takeException(), isNull);
  });
}
````

### `.gitignore`

````text
.dart_tool/
.packages
.pub-cache/
.pub/
build/
coverage/
.flutter-plugins
.flutter-plugins-dependencies
.idea/
*.iml
.DS_Store
android/local.properties
android/.gradle/
android/captures/
**/Flutter/ephemeral/
**/flutter/ephemeral/
ios/Pods/
ios/.symlinks/
ios/Flutter/Generated.xcconfig
ios/Flutter/flutter_export_environment.sh
````

### `.github/workflows/flutter.yml`

````yaml
name: Flutter checks
on: [push, pull_request]
permissions:
  contents: read
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.35.5'
          channel: stable
          cache: true
      - run: flutter pub get
      - run: dart format --output=none --set-exit-if-changed lib test
      - run: flutter analyze
      - run: flutter test
      - run: flutter build web --release
````

### `docs/verification.md`

````markdown
# Перевірка проєкту

Середовище: Flutter 3.35.5, Dart 3.9.2, Linux.

| Команда / сценарій | Результат |
|---|---|
| `dart format lib test` | Код відформатовано |
| `flutter analyze` | No issues found |
| `flutter test --reporter expanded` | 11 tests passed |
| `flutter build web --release` | Built build/web |
| 5 успішних файлів + 1 помилка | Перевірено автоматичним тестом |
| Скасування й відсутність подальшого прогресу | Перевірено автоматичним тестом |
| Зависання → тайм-аут → успішний повтор | Перевірено автоматичним тестом |
| Закриття віджета під час Future та завантаження | Перевірено автоматичними тестами |
| Скриншоти Flutter | 8 PNG у docs/, візуально перевірені |

Під час web build було нефатальне попередження про необов’язковий шрифт CupertinoIcons; збірка завершилася успішно. Інтерфейс використовує Material Icons і вбудований DejaVu.

Android SDK не встановлений у середовищі перевірки: APK не збирався й фізичний телефон не тестувався. Застосунок виконувався в Flutter test engine; скриншоти зроблені через RenderRepaintBoundary.toImage. Окремий інтерактивний запуск релізної web-збірки в браузері не перевірявся.

У тесті скриншотів таймери прискорюються віртуальним часом. Показаний у тестовому підсумку час не є бенчмарком мережі. У звичайному запуску Stopwatch вимірює реальний час роботи пакета.
````
