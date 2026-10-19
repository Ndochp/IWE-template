# day-open.after.android-usage — снятие дампа цифрового благополучия при открытии дня

> **Hook:** `day-open` / `after` (шаг 4.8 графа расширений, WP-529 Ф11).
> **Запрос пилота 16.10:** дамп UsageStats с Android-телефона снимается каждое
> открытие дня. Если телефон не подключен — **пауза с просьбой подключить**
> (Telegram) и досъём после подключения; решение пилота «пауза», не пропуск.
> **Политика отказов:** хук никогда не блокирует открытие дня — любой исход
> (нет adb, таймаут паузы, сбой дампа) даёт `exit 0`; пропуск помечается файлом
> `.pending-<дата>`, по нему напоминание срабатывает на закрытии дня
> (`day-close.after.android-usage.md`).
> **Окно паузы:** 10 минут по умолчанию, опрос каждые 15 с; меняется переменной
> `ANDROID_USAGE_WAIT_MIN` (0 — без паузы, сразу пропуск с пометкой).
> **Сырьё:** `DS-strategy/data/android-usage/` (gitignore, по конвенции telegram-raw).

```bash
if [ -f "/tmp/iwe-dry-run.flag" ]; then
    echo "[extension day-open.after.android-usage] dry-run active, skipping"
    exit 0
fi
HOOK_IWE="${IWE_ROOT:-${IWE:-$HOME/IWE}}"
HOOK_DS="$HOOK_IWE/${IWE_GOVERNANCE_REPO:-DS-strategy}"
HOOK_PULL="$HOOK_DS/scripts/android-usage-pull.sh"
HOOK_DATA="$HOOK_DS/data/android-usage"
HOOK_TODAY=$(date +%F)
[ -f "$HOOK_PULL" ] || { echo "[android-usage] ⚠️ скрипт $HOOK_PULL не найден — пропуск"; exit 0; }
mkdir -p "$HOOK_DATA"
if [ -f "$HOOK_DATA/usagestats-$HOOK_TODAY.json" ]; then
    echo "[android-usage] 🟢 дамп за сегодня уже снят (usagestats-$HOOK_TODAY.json)"
    rm -f "$HOOK_DATA/.pending-$HOOK_TODAY"
    exit 0
fi
HOOK_ADB="$(command -v adb 2>/dev/null || true)"
if [ -z "$HOOK_ADB" ]; then
    HOOK_LA="$(cygpath -u "${LOCALAPPDATA:-}" 2>/dev/null || echo "${LOCALAPPDATA:-}")"
    HOOK_CAND="$HOOK_LA/Microsoft/WinGet/Packages/Google.PlatformTools_Microsoft.Winget.Source_8wekyb3d8bbwe/platform-tools/adb.exe"
    [ -x "$HOOK_CAND" ] && HOOK_ADB="$HOOK_CAND"
fi
if [ -z "$HOOK_ADB" ]; then
    echo "[android-usage] ⚠️ adb не найден — пропуск шага (winget install -e --id Google.PlatformTools)"
    exit 0
fi
android_usage_devcount() {
    "$HOOK_ADB" devices 2>/dev/null | awk 'NR>1 && $2=="device"' | wc -l | tr -d ' '
}
if [ "$(android_usage_devcount)" = "0" ]; then
    HOOK_WAIT="${ANDROID_USAGE_WAIT_MIN:-10}"
    echo "[android-usage] 📱 телефон не подключен — пауза ${HOOK_WAIT} мин с просьбой (Telegram)"
    HOOK_TG_LIB="$HOOK_DS/scripts/lib/telegram.sh"
    if [ -f "$HOOK_TG_LIB" ]; then
        # shellcheck disable=SC1090
        . "$HOOK_TG_LIB" 2>/dev/null || true
        telegram_load_creds 2>/dev/null \
            && telegram_send "📱 Открытие дня: подключи телефон для дампа цифрового благополучия. Жду ${HOOK_WAIT} мин; не подключишь — дамп пропущу, напомню при закрытии дня." 2>/dev/null \
            || echo "[android-usage] просьба в Telegram не отправлена (нет кредов/сети)"
    fi
    HOOK_TICKS=$((HOOK_WAIT * 4))
    HOOK_I=0
    while [ "$HOOK_I" -lt "$HOOK_TICKS" ]; do
        sleep 15
        HOOK_I=$((HOOK_I + 1))
        [ "$(android_usage_devcount)" != "0" ] && break
    done
fi
if [ "$(android_usage_devcount)" = "0" ]; then
    echo "[android-usage] ⏳ телефон не подключён за окно паузы — дамп пропущен, напоминание на закрытии дня"
    echo "$HOOK_TODAY" > "$HOOK_DATA/.pending-$HOOK_TODAY"
    exit 0
fi
if bash "$HOOK_PULL"; then
    rm -f "$HOOK_DATA/.pending-$HOOK_TODAY"
    echo "[android-usage] 🟢 дамп снят (usagestats-$HOOK_TODAY.json)"
else
    echo "[android-usage] ⚠️ дамп не снят (сбой pull) — открытие не блокируется, напоминание на закрытии дня"
    echo "$HOOK_TODAY" > "$HOOK_DATA/.pending-$HOOK_TODAY"
fi
exit 0
```
