# n8n-weather-alerts

Workflow de **n8n** para crear **alertas meteorológicas** con APIs públicas:

- **Nominatim (OpenStreetMap)**: convierte *ciudad → lat/lon*
- **Open-Meteo**: obtiene el pronóstico (24h)
- **Telegram**: envía alertas (MED/HIGH)
- **Gmail**: envía alertas (solo HIGH)
- **Google Sheets (opcional)**: dashboard/log de ejecuciones

> Todo se configura en n8n con credenciales + variables. No hay claves hardcodeadas en el repo.

---

## Archivos

- `Weather Alerts (Open-Meteo + Nominatim + Telegram/Gmail + optional Sheets).json`  
  (workflow principal importable en n8n)

> Si tu archivo tiene otro nombre, reemplazá el nombre arriba por el nombre real del `.json`.

---

## Qué hace

1. Corre en automático con **Schedule (cada 12 horas)**.
2. Genera una lista de ciudades (hardcode en el nodo **Build Cities + Thresholds**).
3. Para cada ciudad:
   - Geocodifica con **Nominatim**.
   - Pide pronóstico a **Open-Meteo**.
   - Calcula métricas 24h (lluvia, Tmax/Tmin, viento, weathercode).
   - Aplica reglas determinísticas (tabla abajo).
   - Hace **dedupe** para no spammear (ventana configurable).
4. Si corresponde, envía alertas:
   - **Telegram**: MED y HIGH (si `TELEGRAM_CHAT_ID` está configurado).
   - **Gmail**: solo HIGH (si `GMAIL_TO` está configurado).
5. (Opcional) Escribe un log en **Google Sheets** si `LOG_SHEET_ID` está configurado.

---

## Reglas determinísticas (tabla de decisión)

Ventana de análisis: **24 horas** (día actual).

| Condición | Alerta | Severidad | Canal |
|---|---|---|---|
| `precip_mm_24h >= 10` **o** tormenta (weathercode 95/96/99) | RAIN o STORM | HIGH | Telegram + Gmail |
| `precip_mm_24h >= 3` | RAIN | MED | Telegram |
| `tmax_c >= 35` | HEAT_WAVE | HIGH | Telegram + Gmail |
| `tmax_c >= 32` | HEAT | MED | Telegram |
| `tmin_c <= 0` | FROST | HIGH | Telegram + Gmail |
| `wind_kmh_max >= 55` | WIND | HIGH | Telegram + Gmail |
| ninguna condición | NONE | NONE | — |

### Anti-spam / dedupe
- No repite alerta para **misma ciudad + tipo + severidad** dentro de `dedupe_hours` (default **6h**).
- Implementación simple: **Workflow Static Data** (sin DB).

---

## Requisitos

- n8n (Cloud o self-hosted)
- Acceso a Internet (APIs públicas)

### Credenciales en n8n (según canal)
- **Telegram**: credencial de Telegram (bot)
- **Gmail**: credencial Gmail OAuth2
- **Google Sheets** (opcional): credencial OAuth2

---

## Variables de entorno (opcionales pero recomendadas)

Configuralas en n8n (Settings → Variables / Env):

- `TIMEZONE` = `America/Argentina/Buenos_Aires` (opcional)
- `TELEGRAM_CHAT_ID` = `<<CHAT_ID>>` (habilita Telegram)
- `GMAIL_TO` = `<<EMAIL_DESTINO>>` (habilita Gmail)
- `LOG_SHEET_ID` = `<<SPREADSHEET_ID>>` (habilita logging en Sheets)
- `SHEET_TAB_DASHBOARD` = `dashboard` (opcional)

> Si no seteás estas variables, el workflow igual corre, pero no enviará por esos canales.

---

## Configuración rápida

1. Importá el workflow:
   - n8n → **Workflows** → **Import from File** → seleccioná el `.json`.
2. Abrí el nodo **Build Cities + Thresholds** y editá:
   - `cities` (lista de ciudades)
   - `thresholds` (umbrales)
3. Configurá credenciales:
   - Telegram / Gmail / Google Sheets (si los vas a usar)
4. Seteá variables de entorno (si aplica).
5. Ejecutá en modo manual para probar.
6. Activá el workflow.

---

## Personalización

### Cambiar ciudades
En el nodo **Build Cities + Thresholds**:
```js
const cities = [
  'Buenos Aires, Argentina',
  'Córdoba, Argentina',
  'Rosario, Argentina'
];
Cambiar umbrales
En el mismo nodo:

js
Copiar código
const thresholds = {
  rain_mm_med: 3,
  rain_mm_high: 10,
  heat_med_c: 32,
  heat_high_c: 35,
  frost_c: 0,
  wind_high_kmh: 55,
  dedupe_hours: 6,
};
Cambiar frecuencia
El nodo Schedule Trigger está configurado cada 12 horas. Podés cambiarlo a:

08:00 y 18:00

cada 3 horas

o el horario que prefieras

Notas / buenas prácticas
Respetá políticas de uso:

Nominatim recomienda identificarte con un User-Agent (ya viene puesto con placeholder).

Para un entorno “pro”, el dedupe debería persistirse en DB/Sheets.

Si vas a hacerlo público, evitá subir archivos con datos personales.
