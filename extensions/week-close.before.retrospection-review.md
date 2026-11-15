# week-close.before.retrospection-review — рефлексии за 6 дней перед свободным ретро недели

> **Hook:** `week-close` / `before` (перед шагом 0б «Свободное ретро недели» — вопросы пилоту задаются после вывода).
> **Запрос пилота 15.11:** перед вопросами «что сработало / что не сработало» выводить данные по вечерним рефлексиям за прошлые 6 дней — пилот писал их в том числе для этого момента.
> **Стиль:** исполняемый bash-блок — тот же файл читает агент в интерактиве и механически eval'ит `day-open-hooks-runner.sh` в автономном конвейере (WP-529 Ф11).
> **Политика отказов:** любой сбой — WARN и `exit 0`, Week Close не блокируем. Нет рефлексии за день — печатаем явно («рефлексии нет»), не молчим (инвариант scaffold «никаких silent skip»).
> **Дни:** 6 дней назад от сегодня (для воскресного Week Close — пн–сб; сегодняшняя рефлексия появится на закрытии дня и в ретро не входит).

```bash
if [ -f "/tmp/iwe-dry-run.flag" ]; then
    echo "[extension week-close.before.retrospection-review] dry-run active, skipping"
    exit 0
fi
WS="${IWE_ROOT:-${IWE:-$HOME/IWE}}"
GOV="${IWE_GOVERNANCE_REPO:-DS-strategy}"
for i in 6 5 4 3 2 1; do
    D=$(date -d "-$i day" +%F 2>/dev/null) || D=$(date -v-${i}d +%F 2>/dev/null)
    [ -z "$D" ] && { echo "[retro-review] дата не вычислена — пропуск"; continue; }
    PLAN="$WS/$GOV/archive/day-plans/DayPlan $D.md"
    [ -f "$PLAN" ] || PLAN="$WS/$GOV/current/DayPlan $D.md"
    if [ ! -f "$PLAN" ]; then
        echo "• $D — DayPlan не найден"
        continue
    fi
    SEC=$(awk 'f==0 && /Рефлексия вечера|Рефлексия \(догон|Рефлексия \(вечер|Рефлексия \(первый вечер/{f=1; print; next} f==1 && (/^#{1,4} / || /^\*\*Завтра начать с/ || /^<\/?details/ || /^\*Создан/ || /^---$/){exit} f==1{print}' "$PLAN" | head -20)
    if [ -z "$SEC" ]; then
        echo "• $D — рефлексии нет"
    else
        echo "• $D:"
        echo "$SEC"
        echo
    fi
done
exit 0
```
