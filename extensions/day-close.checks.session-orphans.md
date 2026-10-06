### 🟡 ПРОВЕРКА: Незакрытые external-сессии (WP-358 Ф10)

> Финализация sessions через `/claude` бот — DP.SC.NNN §close.
> Warning, не block: если есть незакрытые SESSION-* post-cutover — напомнить, но коммит проходит.

```bash
# Скрипт поставляется в шаблоне (scripts/, манифест), в governance-репо НЕ доставляется
# (нет в seed/strategy и в update-backfill) и оттуда не запускается — он sourc'ит
# соседний .claude/lib/iwe-env-bootstrap.sh, а governance сканирует через IWE_DS_MY_STRATEGY.
TG_CHECK=""
for c in "${IWE_TEMPLATE:-}/scripts/check-open-sessions.sh" \
         "scripts/check-open-sessions.sh" \
         "IWE-template/scripts/check-open-sessions.sh" \
         "FMT-exocortex-template/scripts/check-open-sessions.sh"; do
  [ -n "$c" ] && [ -f "$c" ] && TG_CHECK="$c" && break
done
SECTION=$([ -n "$TG_CHECK" ] && bash "$TG_CHECK" 2>/dev/null || true)
if [ -n "$SECTION" ]; then
  COUNT=$(printf '%s\n' "$SECTION" | grep -c '^| \[SESSION-' || true)
  echo "  🟡 $COUNT незакрытых external-сессии в inbox/agent/sessions/ — рассмотри финализацию (DP.SC.NNN §close)"
  printf '%s\n' "$SECTION" | grep '^| \[SESSION-' | head -5
else
  echo "  ✅ Незакрытых external-сессий нет"
fi
```

- [ ] Если ⚠️ — оценить, нужна ли финализация прямо сейчас (создать `sessions/external/YYYY-MM/SESSION-<id>/report.md` + `git mv`). Если не сегодня — закоммитить как есть, продолжит висеть до Day Open завтра.
