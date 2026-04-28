# Hidden Objects — План реализации

Порядок: 1 → 2 → 4 → 5 → 6 → 8 → 7 → 9 → 10 → 3

---

## 1. Level Data & Configuration

ScriptableObject-based система конфигурации уровней.

- [ ] **1.1** `LevelConfig` (ScriptableObject):
  - `AssetReference backgroundRef` — ссылка на фон (Addressable)
  - `AssetReference atlasRef` — ссылка на SpriteAtlas (Addressable)
  - `LevelDisplayMode displayMode` — enum { Image, Silhouette, Text }
  - `bool isTimed`, `float timeLimit` (по умолчанию 180)
  - `int hintLightbulbCount` (2), `int hintMagnifierCount` (2), `int hintArrowCount` (2), `int hintHourglassSeconds` (30)
  - `List<ItemPlacement> items`

- [ ] **1.2** `ItemPlacement` (Serializable class):
  - `string spriteId` — имя спрайта в атласе
  - `Vector2 position`, `float rotation`, `Vector2 scale`
  - `string displayName` — название для текстового режима (English)

- [ ] **1.3** `LevelDatabase` (ScriptableObject):
  - `List<LevelEntry> levels` где `LevelEntry` = { `LevelConfig config`, `Sprite preview`, `string levelName` }

---

## 2. Addressables & Scene Loading

Быстрая загрузка меню, ленивая загрузка ассетов уровня.

- [ ] **2.1** Настроить Addressables Groups:
  - `MenuAssets` — UI меню (грузятся сразу)
  - `LevelAssets_{N}` — фон + atlas каждого уровня (отдельные группы)

- [ ] **2.2** `AssetLoaderService` (MonoBehaviour, синглтон на DontDestroyOnLoad):
  - `async Task<Sprite> LoadBackground(AssetReference ref)`
  - `async Task<SpriteAtlas> LoadAtlas(AssetReference ref)`
  - `void ReleaseLevel()` — освобождение ресурсов при выходе из уровня

- [ ] **2.3** Две сцены в Build Settings:
  - `MenuScene` (index 0)
  - `GameScene` (index 1)

- [ ] **2.4** `SceneTransitionService`:
  - показать loading UI → загрузить ассеты уровня → `SceneManager.LoadSceneAsync("GameScene")` → инициализировать
  - передача выбранного `LevelConfig` через static поле или SO

---

## 3. Menu Scene

Прокручиваемый список уровней. **Реализуется последним** — во время разработки удобнее запускать GameScene напрямую.

- [ ] **3.1** `MenuController` — точка входа:
  - читает `LevelDatabase`, создаёт карточки уровней

- [ ] **3.2** `LevelCardView` (prefab):
  - превью-картинка, название, кнопка
  - `OnClick` → `SceneTransitionService.LoadLevel(config)`

- [ ] **3.3** ScrollView с адаптивным layout:
  - мобайл — 2 колонки, десктоп — 3-4

---

## 4. Game Scene — Background & Camera

Отображение фона с прокруткой и зумом.

- [ ] **4.1** `BackgroundSetup`:
  - получает загруженный Sprite фона
  - ставит на SpriteRenderer
  - вычисляет bounds для clamp камеры

- [ ] **4.2** `CameraScrollComponent` — горизонтальная прокрутка:
  - Input: клавиши ←→, drag мышью, touch swipe
  - Clamp по границам фона
  - `float scrollSpeed`, `float dragSensitivity`

- [ ] **4.3** `CameraZoomComponent` — приближение/удаление:
  - изменение `Camera.orthographicSize`
  - **зум к точке под курсором/пальцем** — пересчёт позиции камеры при изменении `orthographicSize`
  - `float minZoom, maxZoom, zoomStep`
  - clamp чтобы не выйти за фон
  - UI кнопки +/− вызывают `ZoomIn()` / `ZoomOut()`
  - pinch-to-zoom для touch

- [ ] **4.4** `InputRouter` — маршрутизация ввода:
  - определяет: tap/click vs drag (по `dragThreshold` в пикселях)
  - клавиши → scroll
  - scroll wheel / pinch → zoom
  - tap/click → поиск предмета

---

## 5. Item System — Scene Items

Размещение кликабельных предметов поверх фона.

- [ ] **5.1** `ItemSpawner`:
  - по `LevelConfig.items` создаёт GO предметов на сцене
  - для каждого `ItemPlacement`: SpriteRenderer + спрайт из SpriteAtlas по `spriteId`
  - установка position, rotation, scale

- [ ] **5.2** `SceneItemComponent` (на каждом предмете):
  - `string ItemId`
  - `Collider2D` для кликов (PolygonCollider2D или BoxCollider2D)
  - `event Action<SceneItemComponent> OnFound`
  - `bool IsFound`
  - метод `Collect()` — запускает анимацию сбора

- [ ] **5.3** `ItemClickDetector`:
  - Raycast по клику/тапу (`Physics2D.OverlapPoint`)
  - проверяет попадание в `SceneItemComponent`
  - игнорирует уже найденные (`IsFound == true`)
  - вызывает `Collect()`

- [ ] **5.4** `CollectAnimationComponent` (на предмете):
  - DOTween: punch scale → полёт к мировой позиции UI-ячейки → fade out
  - по завершении — `gameObject.SetActive(false)`

---

## 6. Item Panel — Bottom UI

Панель списка искомых предметов.

- [ ] **6.1** `ItemPanelController`:
  - создаёт UI-ячейки по `LevelConfig.items`
  - горизонтальный ScrollRect

- [ ] **6.2** `ItemPanelCell` (prefab ячейки):
  - 3 режима по `LevelDisplayMode`:
    - **Image**: спрайт предмета как есть
    - **Silhouette**: тот же спрайт + **unlit шейдер с `_Color = white`** (один shared материал)
    - **Text**: TextMeshPro с `displayName`
  - `MarkAsFound()`:
    - Image/Silhouette → fade out или галочка
    - Text → strikethrough (`<s>text</s>`)
  - `Vector3 GetWorldPosition()` — для расчёта точки прилёта анимации

- [ ] **6.3** Silhouette материал:
  - простой unlit sprite шейдер, `_Color` = white, multiply поверх альфы спрайта
  - один `Material` инстанс на все силуэтные ячейки

---

## 7. Timer System

Обратный отсчёт.

- [ ] **7.1** `TimerComponent`:
  - `float remainingTime`
  - `event Action OnTimeUp`
  - `event Action<float> OnTimeChanged`
  - `Pause()` / `Resume()` / `AddTime(float seconds)`

- [ ] **7.2** `TimerView`:
  - подписка на `OnTimeChanged` → текст MM:SS
  - при <30 сек: красный цвет + DOTween пульсация

- [ ] **7.3** Интеграция:
  - `GameSessionController`: если `isTimed` → запустить таймер, иначе скрыть UI

---

## 8. Win/Lose & Game Session

Управление состоянием игры.

- [ ] **8.1** `GameSessionController` — оркестратор:
  - init: загрузить ассеты → спавн предметов → настроить панель → запустить таймер
  - трекинг: `int foundCount` / `int totalCount`
  - `event Action OnWin`, `event Action OnLose`

- [ ] **8.2** Логика завершения:
  - `foundCount == totalCount` → `OnWin`
  - `TimerComponent.OnTimeUp` → `OnLose`
  - при завершении: остановить таймер, заблокировать ввод

- [ ] **8.3** `ResultModalView`:
  - Win: время прохождения, кнопки "Далее" / "В меню"
  - Lose: кнопки "Заново" / "В меню"
  - DOTween: scale from 0 + fade overlay

---

## 9. Hint System

4 типа подсказок с ограниченным количеством.

- [ ] **9.1** `HintService`:
  - получает список ненайденных предметов от `GameSessionController`
  - `List<SceneItemComponent> GetRandomUnfound(int count)`

- [ ] **9.2** `HintButtonComponent` (базовый, на каждой кнопке):
  - `int usesRemaining` (из LevelConfig)
  - UI badge с количеством
  - блокировка при `usesRemaining == 0`

- [ ] **9.3** `LightbulbHint` — подсветка 1 предмета:
  - glow/outline эффект на 5 сек (DOTween)

- [ ] **9.4** `MagnifierHint` — пульсация 2 предметов:
  - `DOScale` ping-pong на 5 сек

- [ ] **9.5** `ArrowHint` — автосбор 3 предметов:
  - последовательный `Collect()` с задержкой между каждым

- [ ] **9.6** `HourglassHint` — добавление времени:
  - `TimerComponent.AddTime(seconds)`
  - анимация "+30s"
  - скрыта если `!isTimed`

---

## 10. Input System

Единая обработка ввода для мобайла и десктопа.

- [ ] **10.1** `InputRouter` (рефакторинг из 4.4):
  - **Tap/Click** (короткое нажатие, движение < `dragThreshold`) → item click
  - **Drag** (нажатие + движение) → camera scroll
  - **Keyboard ←→** → camera scroll
  - **Scroll wheel / pinch** → camera zoom (к точке под курсором)

- [ ] **10.2** `TouchInputHandler`:
  - single finger drag = scroll
  - single finger tap = item click
  - two finger pinch = zoom

- [ ] **10.3** `MouseInputHandler`:
  - LMB drag = scroll
  - LMB click = item click
  - scroll wheel = zoom

---

## Ключевые решения

| Решение | Выбор |
|---------|-------|
| Конфиг уровня | ScriptableObject (`LevelConfig`) хранит позиции/повороты предметов |
| Источник спрайтов | SpriteAtlas (Addressable, по одному на уровень) |
| Силуэт | Тот же спрайт + unlit шейдер `_Color = white` |
| Локализация | Нет, English only |
| Анимация сбора | Предмет летит со сцены к ячейке в панели, затем ячейка помечается |
| Zoom | К точке под курсором/пальцем (пересчёт позиции камеры) |
| Анимации | DOTween |
| Загрузка ассетов | Addressables, ленивая загрузка уровневых ресурсов |
| Сцены | MenuScene (быстрая) + GameScene |
