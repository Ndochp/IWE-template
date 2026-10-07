# day-open.before.fpf-pull — подтягивание FPF до апстрима перед открытием дня

> **Hook:** `day-open` / `before` (перед шагом 1 «Вчера») — утренняя подготовка.
> **Запрос пилота 07.10:** FPF — авторский репо ailev/FPF без форка, обновляется
> активно (вчера +24, сегодня ещё +9 за ночь), и отставание всплывало в
> «Требует внимания» два дня подряд. Скрипт тянет FPF fast-forward'ом каждое
> утро, чтобы копия не застаивалась.
> **Стиль:** исполняемый bash-блок — тот же файл читает агент в интерактиве и
> механически eval'ит `day-open-hooks-runner.sh` в автономном конвейере (WP-529 Ф11).
> **Политика отказов:** только fast-forward; любой сбой (сеть, расхождение
> истории) — WARN и `exit 0`, день не блокируем (неуспешный `before`-хук
> блокирует коммит конвейера целиком — недопустимо для фонового подтягивания).

```bash
if [ -f "/tmp/iwe-dry-run.flag" ]; then
    echo "[extension day-open.before.fpf-pull] dry-run active, skipping write steps"
    exit 0
fi
FPF_DIR="${IWE_ROOT:-$HOME/IWE}/FPF"
if [ ! -d "$FPF_DIR/.git" ]; then
    echo "[fpf-pull] FPF репозиторий не найден ($FPF_DIR) — пропуск"
    exit 0
fi
FPF_BRANCH=$(git -C "$FPF_DIR" branch --show-current)
git -C "$FPF_DIR" fetch origin --quiet 2>/dev/null || { echo "[fpf-pull] ⚠ fetch не удался (сеть?) — FPF не тронут"; exit 0; }
FPF_BEHIND=$(git -C "$FPF_DIR" rev-list "HEAD..origin/${FPF_BRANCH}" --count 2>/dev/null || echo 0)
if [ "${FPF_BEHIND:-0}" -eq 0 ]; then
    echo "[fpf-pull] 🟢 FPF актуален (origin/${FPF_BRANCH}, $(git -C "$FPF_DIR" rev-parse --short HEAD))"
    exit 0
fi
if git -C "$FPF_DIR" pull --ff-only origin "${FPF_BRANCH}" --quiet 2>/dev/null; then
    echo "[fpf-pull] 🟢 FPF подтянут: +${FPF_BEHIND} коммитов (origin/${FPF_BRANCH} → $(git -C "$FPF_DIR" rev-parse --short HEAD))"
else
    echo "[fpf-pull] 🔴 fast-forward невозможен (расхождение истории) — нужен ручной rebase, FPF не тронут"
fi
exit 0
```
