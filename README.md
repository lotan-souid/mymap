# mymap

סטאק GIS מודרני, קוד-פתוח לחלוטין, בהשראת ארכיטקטורת GovMap הלאומית. רץ כולו
תחת `docker compose` על node יחיד — מהיר להקמה בפיתוח, עם מסלול ברור לגדילה
לסקייל לאומי מעל Kubernetes ו-CDN.

מפה וקטורית (MapLibre GL), רסטר דינמי (COG), תאימות OGC (WMS/WFS), ניתוב, חיפוש
טקסט חופשי, וניהול זהויות — הכל ברכיבים קוד-פתוח, ללא רישוי per-seat.

לפירוט מלא של הארכיטקטורה, שני נתיבי הקריאה, ודוגמאות שימוש לכל שירות — ראו
[ARCHITECTURE.md](ARCHITECTURE.md).

## הרכיבים

| שכבה | רכיבים |
|------|--------|
| לקוח | MapLibre GL JS |
| שירותי מפה | Martin, TiTiler, GeoServer, Valhalla, OpenSearch |
| הפצה ואחסון | PMTiles, Planetiler, MinIO, Caddy |
| נתונים | PostgreSQL + PostGIS |
| זהות | Keycloak (OIDC) |

## הרצה מהירה

```bash
# 1. הכן משתני סביבה
cp .env.example .env
# ערוך את כל ערכי change_me_* ב-.env לסיסמאות חזקות

# 2. הרם את הסטאק
docker compose up -d

# 3. בדיקת מצב
docker compose ps
```

Martin עולה מוכן לעבודה (auto-publish מ-PostGIS), ו-Valhalla מוריד ובונה את גרף
הניתוב לישראל אוטומטית בהפעלה הראשונה. פרטים מלאים ב-[ARCHITECTURE.md](ARCHITECTURE.md#הרצה-מהירה).

לאחר ההרמה:

- Frontend: `http://localhost/`
- Martin (אריחים): `http://localhost:3000/`
- GeoServer: `http://localhost:8600/geoserver/`
- Keycloak: `http://localhost:8080/`
- MinIO (קונסולה): `http://localhost:9001/`

## רישוי

הקוד ברכיבי הסטאק (Docker images) קוד-פתוח, אך תחת רישיונות שונים לפי רכיב
(MIT/Apache/GPLv2/AGPLv3) — ראו [פירוט מלא](ARCHITECTURE.md#רישוי). רישוי
התוכן/הדאטה (למשל OSM תחת ODbL) נפרד מרישוי התוכנה.
