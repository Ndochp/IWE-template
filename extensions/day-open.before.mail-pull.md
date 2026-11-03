# day-open.before.mail-pull — утренний опрос почтового ящика IWE (WP-005)

> **Hook:** `day-open` / `before` (перед сборкой DayPlan) — чтение почты утром.
> **Решение пилота 21.10:** почта опрашивается в утреннем ритуале; контрольная
> точка — перед закрытием дня (extensions/day-close.checks.mail-pull.md).
> **Стиль:** исполняемый bash-блок — тот же файл читает агент в интерактиве и
> механически eval'ит `day-open-hooks-runner.sh` в автономном конвейере (WP-529 Ф11).
> **Политика отказов:** любой сбой (сеть, секреты) — WARN и `exit 0`, день не
> блокируем (фоновый канал; ручной прогон — `scripts/mail-pull.sh`).

```bash
if [ -f "/tmp/iwe-dry-run.flag" ]; then
    echo "[extension day-open.before.mail-pull] dry-run active, skipping"
    exit 0
fi
GOV="${IWE_GOVERNANCE_REPO:-DS-strategy}"
MAIL_PULL="${IWE:-$HOME/IWE}/$GOV/scripts/mail-pull.sh"
if [ ! -f "$MAIL_PULL" ]; then
    echo "[mail-pull] скрипт не найден ($MAIL_PULL) — пропуск"
    exit 0
fi
if OUT=$(bash "$MAIL_PULL" 2>&1); then
    NEW=$(printf '%s\n' "$OUT" | grep -c '^20' || true)
    if [ "${NEW:-0}" -eq 0 ]; then
        echo "[mail-pull] 🟢 новой почты нет"
    else
        echo "[mail-pull] 🟢 писем за ночь: $NEW (сырьё inbox/mail-raw.jsonl, вложения inbox/Media-mail/ — маршрутизация за агентом)"
    fi
else
    echo "[mail-pull] 🟡 опрос не удался (сеть/секреты?) — прогнать вручную: bash $GOV/scripts/mail-pull.sh"
fi
exit 0
```
