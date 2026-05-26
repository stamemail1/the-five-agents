---
name: gpt-image-gen
description: מעטפת לקריאה ל-OpenAI Images API ליצירת תמונה מ-prompt. Use when an agent (typically Yuval) needs to generate an image via OpenAI. Wraps the HTTP call, decodes the base64 response, and saves a PNG to disk. Use whenever an agent needs to call OpenAI Images.
when_to_use: כשסוכן (בדרך כלל יובל) צריך לייצר תמונה. הסקיל מתאר את הקריאה המדויקת ל-API ואיך לפענח את התגובה.
---

# gpt-image-gen — מעטפת ל-OpenAI Images API

הסקיל מתעד **איך** לקרוא ל-OpenAI Images API ליצירת תמונה ולשמור אותה כקובץ PNG.

---

## ⚠️ אזהרה קריטית — שם המודל

המודל הוא **`gpt-image-2`**.

- `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI.
- הוא יצא ב-**21 באפריל 2026**.
- ייתכן שהידע הפנימי של הסוכן (cutoff) לא כולל אותו — **זה לא אומר שהוא לא קיים**.
- **אסור להחליף לאלטרנטיבות** (`dall-e-3`, `gpt-image-1`, וכו'). השתמש בדיוק במודל הזה.
- אם יש שגיאה בקריאה ל-API, הבעיה היא **כמעט תמיד** ב-`OPENAI_API_KEY` או ב-parameters — **לא** בשם המודל.

---

## דרישות מוקדמות

- `OPENAI_API_KEY` מוגדר ב-`.env` (בשורש הפרויקט) עם מפתח אמיתי.
- `curl` זמין (קיים ב-Git Bash בברירת מחדל).
- `python` זמין (לפענוח base64 — לרוב מותקן ב-Windows + Git Bash).
- אופציונלי: `jq` ו-`base64` — אם קיימים, אפשר להשתמש בנתיב הפשוט. אחרת — fallback ל-Python.

---

## קלט

- `prompt` (string) — תיאור התמונה באנגלית (מומלץ) או עברית.
- `output_path` (string) — נתיב מלא לקובץ PNG שייכתב (לדוגמה: `yuval/outputs/2026-05-26-cat-minimal.png`).

## פלט

- קובץ PNG בנתיב `output_path`.
- הדפסת הנתיב ל-stdout בסיום מוצלח.

---

## טעינת ה-API key מ-.env

לפני ההפעלה, יש לטעון את המפתח מהקובץ `.env`:

```bash
export OPENAI_API_KEY=$(grep -E '^OPENAI_API_KEY=' .env | cut -d= -f2-)
```

אם המפתח ריק — לעצור ולדווח: `"OPENAI_API_KEY missing in .env"`.

---

## הקריאה ל-API

**Endpoint:** `POST https://api.openai.com/v1/images/generations`

**Body (JSON):**

```json
{
  "model": "gpt-image-2",
  "prompt": "<the prompt>",
  "size": "1024x1024",
  "quality": "medium",
  "output_format": "png"
}
```

התשובה היא JSON שמכיל `data[0].b64_json` — base64 של הקובץ.

---

## נתיב 1 — מועדף (jq + base64)

```bash
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"gpt-image-2\",
    \"prompt\": \"$PROMPT\",
    \"size\": \"1024x1024\",
    \"quality\": \"medium\",
    \"output_format\": \"png\"
  }" | jq -r '.data[0].b64_json' | base64 --decode > "$OUTPUT_PATH"
```

**מתי להשתמש:** אם `jq` וגם `base64` מותקנים (בדוק עם `command -v jq && command -v base64`).

---

## נתיב 2 — Fallback (Python)

ב-Git Bash על Windows, `jq` בדרך כלל לא מותקן. במקרה כזה, השתמש ב-Python לפענוח:

```bash
# 1. שמור את התגובה ל-temp file
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"gpt-image-2\",
    \"prompt\": \"$PROMPT\",
    \"size\": \"1024x1024\",
    \"quality\": \"medium\",
    \"output_format\": \"png\"
  }" > /tmp/openai-image-response.json

# 2. פענח עם Python
python -c "
import base64, json, sys
with open('/tmp/openai-image-response.json', 'r', encoding='utf-8') as f:
    resp = json.load(f)
if 'error' in resp:
    print('OpenAI API error:', resp['error'], file=sys.stderr)
    sys.exit(1)
b64 = resp['data'][0]['b64_json']
with open(sys.argv[1], 'wb') as out:
    out.write(base64.b64decode(b64))
print(sys.argv[1])
" "$OUTPUT_PATH"
```

**מתי להשתמש:** ברירת המחדל ב-Windows / Git Bash. אם `jq` או `base64` חסרים — נתיב זה.

---

## טיפול בשגיאות

- **`OPENAI_API_KEY missing`** — המשתמש צריך למלא את `.env`.
- **`401 Unauthorized`** — מפתח לא תקין; לוודא שאין רווחים/שורה ריקה.
- **`400 Bad Request`** — בדוק את ה-prompt (לא מכיל תווים בעייתיים שלא נחמקו) ואת ה-parameters.
- **`429 Rate Limit`** — לחכות מספר שניות ולנסות שוב.
- **שגיאה אחרת** — להדפיס את גוף התגובה המלא לדיווח.

**שוב — אסור לשנות את שם המודל כפתרון לשגיאה.**

---

## אימות לאחר ההרצה

```bash
ls -la "$OUTPUT_PATH" && [ -s "$OUTPUT_PATH" ] && echo "OK: $OUTPUT_PATH"
```

הקובץ חייב להיות קיים ו-size > 0. אחרת — שגיאה.
