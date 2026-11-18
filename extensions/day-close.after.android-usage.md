# day-close.after.android-usage — съёмка дампа цифрового благополучия при закрытии дня

> **Hook:** `day-close` / `after` (тот же механизм, что `day-close.after.md`).
> **Запрос пилота 16.10:** дамп UsageStats с Android-телефона снимается каждое
> открытие дня.
> **Решение пилота 18.11 (N-011 рядом, по дампам):** съёмка и на закрытии дня —
> хук сначала сам пытается снять дамп (телефон вечером часто подключён), и только
> при невозможности просит пилота.
> **Политика отказов:** хук никогда не блокирует закрытие дня — любой исход
> (нет adb, телефон не подключён, сбой дампа) даёт `exit 0`; при пропуске
> пилоту — просьба подключить телефон, маркер `.pending-<дата>` сохраняется.
> **Сырьё:** `DS-strategy/data/android-usage/` (gitignore, по конвенции telegram-raw).

```bash
if [ -f "/tmp/iwe-dry-run.flag" ]; then
    echo "[extension day-close.after.android-usage] dry-run active, skipping"
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
    echo "[android-usage] 📱 телефон не подключён — дамп пропущен; подключите и дайте команду забрать: bash ${HOOK_DS}/scripts/android-usage-pull.sh — или подключите и скажите «сними дамп»"
    echo "$HOOK_TODAY" > "$HOOK_DATA/.pending-$HOOK_TODAY"
    exit 0
fi
if bash "$HOOK_PULL"; then
    rm -f "$HOOK_DATA/.pending-$HOOK_TODAY"
    echo "[android-usage] 🟢 дамп снят при закрытии дня (usagestats-$HOOK_TODAY.json)"
else
    echo "[android-usage] ⚠️ дамп не снят (сбой pull) — закрытие не блокируется; итог дня: android-usage: дамп не снят"
    echo "$HOOK_TODAY" > "$HOOK_DATA/.pending-$HOOK_TODAY"
fi
exit 0
```

## Если дамп так и не снят (маркер `.pending-<дата>` на месте)

1. Сказать пилоту при закрытии дня: «Дамп цифрового благополучия за сегодня
   не снят — подключи телефон и запусти `DS-strategy/scripts/android-usage-pull.sh`
   (или подключи и скажи «сними дамп», запущу я).»
2. Записать в итоги дня строку: `android-usage: дамп не снят`.
3. После ручного/агентского снятия дампа — удалить маркер `.pending-<дата>`.

## Примечание

Утренняя съёмка — хук `day-open.after.android-usage.md` (автосъёмка при открытии
дня). Этот хук — вечерняя: снимает дамп, если утро пропустили, и страхует
статистикой весь день целиком.
