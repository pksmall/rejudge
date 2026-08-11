# Rejudge на Ollama — по шагам

От нуля до работающего ревью. Каждый шаг — команда и то, что должно появиться в ответ.
Если появилось другое, смотрите «Если не сходится» в конце нужного шага.

Всего шагов семь, минут на всё — десять.

---

## Шаг 1. Node 22 или новее

```bash
node --version
```

Ждём `v22.19.0` или выше.

Если меньше:

```bash
nvm install 22
nvm use 22
```

**Если не сходится.** Проверьте ту же команду в новом окне терминала: `nvm use` живёт только в текущей сессии. Чтобы закрепить — `nvm alias default 22`. Если другие проекты держатся на старой Node, дефолт не трогайте: шаг 4 поставит обёртку, которая сама найдёт нужную версию.

---

## Шаг 2. Ollama и вход в аккаунт

```bash
ollama --version
ollama signin        # если ещё не входили
```

Проверяем, что облако отвечает по подписке:

```bash
ollama run glm-5.2:cloud "ping"
```

Ждём осмысленный ответ модели. Это же поднимает демон, если он не был запущен.

**Если не сходится.** `402 extra usage only` — модель вне вашего тарифа, возьмите другую из шага 3. Ошибка авторизации — повторите `ollama signin`.

---

## Шаг 3. Стянуть модели

Нужно **минимум четыре** модели: три ревьюера и судья. Берите из разных лабораторий —
две модели одной линейки ошибаются одинаково, и панель из них ничего не проверяет.

Каталог того, что умеет и рассуждать, и вызывать инструменты:
<https://ollama.com/search?c=thinking&c=cloud&c=tools>

```bash
ollama pull deepseek-v4-flash:0731-cloud
ollama pull minimax-m3:cloud
ollama pull gemma4:31b-cloud
ollama pull glm-5.2:cloud
```

**Правило имён.** У модели без тега — `<модель>:cloud`. С тегом — `<модель>:<тег>-cloud`.
То есть `glm-5.2:cloud`, но `gpt-oss:120b-cloud`.

Проверяем:

```bash
ollama list
```

**Если не сходится.** `model not found` — почти всегда имя: перепутан `:cloud` и `-cloud`.
Стягивать облачные модели, кстати, не обязательно — демон проксирует их и так — но
стянутая заглушка (несколько сотен байт) кладёт модель в локальный список, откуда её видит
шаг 5.

---

## Шаг 4. Поставить Rejudge

```bash
git clone -b mine git@github.com:pksmall/rejudge.git
cd rejudge
just setup
npm link
```

Нужны `bun` и `just` — `brew install bun just`.

Проверяем:

```bash
cd ~ && rejudge --help | head -3
```

Ждём строку `usage: rejudge`, а ниже в выводе — `setup ollama`.

**Если не сходится.** `webidl.util.markAsUncloneable is not a function` — запустилась старая Node.
Вернитесь к шагу 1 либо положите обёртку, которая сама выбирает интерпретатор:

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/rejudge <<'EOF'
#!/bin/sh
for dir in $(ls -d "$HOME"/.nvm/versions/node/v2[2-9]* "$HOME"/.nvm/versions/node/v[3-9]* 2>/dev/null | sort -Vr); do
    if [ -f "$dir/lib/node_modules/rejudge/bin/rejudge.js" ]; then
        exec "$dir/bin/node" "$dir/lib/node_modules/rejudge/bin/rejudge.js" "$@"
    fi
done
echo "rejudge: no Node >= 22 with the rejudge package installed" >&2
exit 1
EOF
chmod +x ~/.local/bin/rejudge
```

`~/.local/bin` должен быть в `PATH` раньше каталога Node.

Можно поставить и из npm — `npm install -g rejudge` — но тогда команды `setup` не будет,
она есть только в этом форке, и шаг 5 придётся делать руками по [ollama.md](ollama.md).

---

## Шаг 5. Одна команда настройки

```bash
rejudge setup ollama --dry-run     # посмотреть, ничего не записывая
rejudge setup ollama               # записать
```

Ждём примерно такое:

```
Ollama setup — 4 models declared, 0 skipped

provider  wrote /Users/you/.pi/agent/models.json
config    wrote /Users/you/.config/rejudge/config.json

panel
  reviewer  ollama/minimax-m3:cloud@high
  reviewer  ollama/gemma4:31b-cloud@high
  reviewer  ollama/glm-5.2:cloud@high
  judge     ollama/deepseek-v4-flash:0731-cloud@high
  every slot comes from a different lab
```

Команда не ходит в облако и ничего не пуллит: она объявляет то, что уже лежит у вас
локально. Список моделей — ваш, и обновлять его тоже вам.

**Если не сходится.**

- `a panel needs 4 eligible models … found 2` — вернитесь к шагу 3, моделей мало.
- В выводе есть блок `skipped` — эти модели не годятся в ревьюеры, причина написана рядом.
  Обычно нет `thinking` или нет `tools`.
- `some slots reuse the same lab` — панель собралась, но из одной линейки. Работать будет,
  проверять — нет. Стяните модель другого вендора.
- Строка `local models` — в панель попали локальные веса. Прочитайте про контекст в
  [ollama.md](ollama.md#purely-local-models): демон отдаёт не то окно, что заявлено в модели,
  а переполнение обрезается молча.
- `cannot reach the Ollama daemon` — демон не поднят, вернитесь к шагу 2.

---

## Шаг 6. Первый настоящий прогон

Из любого каталога:

```bash
rejudge "Reply with the single word READY and nothing else."
```

В stderr идёт прогресс с именами моделей, в stdout — ответ. Занимает 20–40 секунд.
Последняя строка stderr даёт `run id` для продолжения разговора.

Теперь на реальной задаче, из корня репозитория, который хотите отдать на ревью:

```bash
cd ~/your/project
git diff | rejudge "review this change"
```

**Если не сходится.**

- `no config found` — шаг 5 не записал конфиг. Проверьте, что не запускали его с `--dry-run`.
- `410 … was retired` — модель снята с облака. `ollama list` её всё ещё показывает, но
  сервис больше не отдаёт: стяните замену (шаг 3) и повторите шаг 5.
- `invalid reasoning value` — в конфиге уровень, которого Ollama не знает. Допустимы
  `low`, `medium`, `high`, `xhigh`; `minimal` она отвергает.
- Ревью пришло, но задачу проигнорировало — в `models.json` у модели нет
  `compat.supportsDeveloperRole: false`. Перезапустите шаг 5, он ставит его сам.
- Прогон висит несколько минут на одной модели — бывает: панель ждёт самую медленную.
  `nemotron-3-super` в замере взял 13 минут против 2 у остальных.

---

## Шаг 7. По желанию

**Своя панель в отдельном проекте.** Файл `<проект>/.rejudge/config.json` перебивает
глобальный — молча, поэтому при неожиданном результате сверяйтесь со строкой `config:`
в выводе.

```bash
rejudge setup ollama --project        # панель в текущий проект
```

**Вызов из Claude Code.** Скиллы `/rejudge` и `/rejudge-diff`:

```bash
mkdir -p ~/.claude/skills
ln -s "$PWD/docs/skills/rejudge" ~/.claude/skills/rejudge
ln -s "$PWD/docs/skills/rejudge-diff" ~/.claude/skills/rejudge-diff
```

**Логи рассуждений.** `"debugLog": true` в конфиге — каждый прогон пишет
`.rejudge/logs/*.jsonl`. Держите вне git.

**Обновить модели.** После каждого `ollama pull` или `ollama rm` перезапустите
`rejudge setup ollama` — он перепишет провайдера под новый список. Конфиг при этом не
тронется, для замены панели нужен `--force`.

---

## Что важно помнить

Глобальная команда — симлинк на этот чекаут, потому что ставили через `npm link`. Значит она
следует за текущей веткой: переключите чекаут на `upstream/main`, и `setup ollama` исчезнет,
в апстриме его нет. После правок в `src/` нужна пересборка — `just build`.

Ключей нигде не нужно: облачные модели идут через локальный демон, а креды держит он.

Подробное объяснение каждой настройки, правило имён, таблица отказов и ловушка с локальным
контекстом — в [ollama.md](ollama.md).
