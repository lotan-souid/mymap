# mymap — ארכיטקטורת הסטאק

סטאק GIS מודרני, קוד-פתוח לחלוטין, בהשראת ארכיטקטורת GovMap הלאומית — בנוי לרוץ
תחת `docker compose` על node יחיד (פיתוח / POC / פרודקשן קטן), עם מסלול ברור
לגדילה לסקייל לאומי מעל Kubernetes ו-CDN.

הרעיון המרכזי: **הפרדה בין שני נתיבי קריאה** — נתיב ציבורי בנפח גבוה (אריחים
מהירים דרך CDN) ונתיב תאימות תקני (WMS/WFS דרך GeoServer) — כך שאף אחד מהם אינו
צוואר בקבוק של השני.

---

## סקירת שכבות

| שכבה | תפקיד | רכיבים |
|------|-------|--------|
| לקוח | תצוגה בדפדפן ובשדה | MapLibre GL JS, MapLibre Native, deck.gl |
| שירותי מפה | הגשת אריחים, ניתוב, חיפוש | Martin, TiTiler, GeoServer, Valhalla, OpenSearch |
| הפצה | בסיס-מפה ואחסון | PMTiles, Planetiler, MinIO, Caddy/CDN |
| נתונים | אחסון מרחבי | PostgreSQL + PostGIS |
| רוחביים | זהות ותשתית | Keycloak (OIDC), Docker Compose |

---

## רכיבי הסטאק

| רכיב | תפקיד | פורט (host) | רישיון |
|------|-------|-------------|--------|
| PostGIS | בסיס נתונים מרחבי — ליבת המערכת | 5432 | GPLv2 |
| Martin | אריחים וקטוריים (MVT) מ-PostGIS ו-PMTiles, ב-Rust | 3000 | MIT / Apache-2.0 |
| TiTiler | חיתוך רסטר דינמי מ-Cloud-Optimized GeoTIFF | 8000 | MIT |
| GeoServer | שער תאימות: WMS / WFS / WCS לתקני OGC | 8600 | GPLv2 |
| OpenSearch | חיפוש טקסט חופשי (fork פתוח של Elasticsearch) | 9200 | Apache-2.0 |
| Valhalla | מנוע ניתוב, אזורי שירות, אופטימיזציית מסלולים | 8002 | MIT |
| Keycloak | ניהול זהויות והרשאות (OIDC) | 8080 | Apache-2.0 |
| MinIO | אחסון אובייקטים S3-compatible (אריחים, תצ"א) | 9000 / 9001 | AGPLv3 |
| Caddy (web) | reverse proxy, TLS, והגשת ה-frontend | 80 / 443 | Apache-2.0 |
| MapLibre GL JS | רינדור המפה בצד הלקוח (מוגש ע"י Caddy) | — | BSD-3-Clause |
| Planetiler | job חד-פעמי לייצור בסיס-המפה כ-PMTiles | — | Apache-2.0 |

כל הרכיבים קוד פתוח, ללא רישוי per-seat וללא תלות בספק יחיד.

---

## שני נתיבי הקריאה

הבחירה הארכיטקטונית המרכזית היא פיצול העומס לשני מסלולים נפרדים היוצאים מ-PostGIS:

**1. נתיב ציבורי — נפח גבוה.**
`PostGIS → Martin / PMTiles → CDN → MapLibre GL`
זהו המסלול שמשרת את הציבור הרחב: בסיס-המפה והאריחים הווקטוריים. הוא מותאם למהירות
ולסקייל. בזכות PMTiles (קובץ יחיד על אחסון אובייקטים) ו-CDN, רוב תעבורת הקריאה
כלל אינה מגיעה ל-origin — מה שמאפשר את הגדילה הזולה שהמודל הישן לא איפשר.

**2. נתיב תאימות — תקני OGC.**
`PostGIS → GeoServer (WMS/WFS/WCS) → צרכנים חיצוניים`
זהו המסלול שמשרת מערכות GIS אחרות: QGIS ו-ArcGIS על שולחן העבודה, סוכנויות ממשלה
נוספות, ותהליכי הדפסה ו-PDF. GeoServer מספק כאן את מה שאין בנתיב הראשון: רינדור
קרטוגרפי בצד השרת (SLD/CSS), עריכה טרנזקציונית (WFS-T), וחשיפת שירותי OGC תקניים
כדרישת אינטרופרביליות. נתיב זה נמוך-נפח אך קריטי לתאימות.

מ-2019 שבו GeoServer נשא את *כל* עומס הקריאה, כאן הוא מתמקד רק בתאימות — והעומס
הכבד עובר לנתיב ה-CDN.

---

## זרימת נתונים

1. נתונים מרחביים נטענים ל-PostGIS (ייבוא מ-shapefile/GeoJSON/GML או עריכה ישירה).
2. **פעם אחת:** Planetiler מייצר את בסיס-המפה כ-`israel.pmtiles` לתיקיית `./tiles`.
3. Martin מגיש אריחים וקטוריים — דינמית מ-PostGIS או מקובץ ה-PMTiles.
4. שכבות רסטר (תצ"א) נשמרות כ-COG ב-MinIO, ו-TiTiler מחתך אותן לפי דרישה.
5. GeoServer חושף את אותם נתונים מ-PostGIS כשירותי WMS/WFS/WCS תקניים.
6. Caddy מגיש את אפליקציית ה-MapLibre GL ומתפקד כ-reverse proxy מול השירותים.
7. Keycloak מאמת משתמשים לאזורים המוגנים (למשל "אזור אישי" ושיתוף בין ארגונים).

---

## הרצה מהירה

```bash
# 1. הכן את משתני הסביבה והחלף את הסיסמאות
cp .env.example .env
# ערוך את כל ערכי change_me_* לסיסמאות חזקות ב-.env (הקובץ לא נכנס ל-git)

# 2. (פעם אחת) ייצר את בסיס-המפה לישראל
docker run -v ./tiles:/data ghcr.io/onthegomap/planetiler:latest \
  --area=israel --output=/data/israel.pmtiles

# 3. הרם את הסטאק
docker compose up -d

# 4. בדיקת מצב
docker compose ps
```

כתובות ברירת מחדל לאחר ההרמה:

- MapLibre GL (frontend): `http://localhost/`
- Martin (אריחים): `http://localhost:3000/`
- GeoServer (ניהול): `http://localhost:8600/geoserver/`
- Keycloak (ניהול): `http://localhost:8080/`
- MinIO (קונסולה): `http://localhost:9001/`

---

## דרישות חומרה

| רמה | שימוש | CPU | RAM | אחסון |
|-----|-------|-----|-----|--------|
| A — פיתוח / POC | node יחיד, אזור/עיר | 4–8 vCPU | 16 GB | 100–256 GB NVMe |
| B — פרודקשן קטן | מחלקה, מאות משתמשים | 8–16 cores | 32–64 GB | 1–2 TB NVMe + object storage |
| C — סקייל לאומי | ציבורי, K8s + CDN | 16–32+ cores (DB ייעודי) | 64–256 GB | TB רבים (תצ"א) |

הערה: PostGIS ו-OpenSearch רעבים ל-RAM ומעדיפים NVMe; Martin קליל מאוד; Valhalla
ו-Planetiler צורכים RAM לפי גודל האזור — ולישראל בלבד הדרישה צנועה. ברמות A ו-B,
`docker compose` על שרת יחיד מספיק. לרמה C עוברים ל-Kubernetes, אך ה-CDN סופג את
עומס הקריאה כך שה-origin נשאר צנוע.

---

## רישוי

הסטאק כולו קוד פתוח. שתי נקודות בהקשר של רכש ממשלתי:

**Copyleft.** רוב הרכיבים מתירניים (MIT/BSD/Apache). שלושה הם copyleft: PostGIS
ו-GeoServer (GPLv2) — אין מגבלה על שימוש ופריסה ללא שינוי קוד. MinIO הוא AGPLv3
עם "network copyleft" — מגבלה חלה רק אם *משנים* את הקוד ומציעים אותו כשירות.
לפריסה רגילה אין בעיה; אם נדרשת הקפדה, אפשר להחליף ב-SeaweedFS או ב-object storage
של ספק הענן.

**רישוי הנתונים נפרד מרישוי התוכנה.** התוכנה פתוחה, אך הדאטה כפוף לתנאים משלו:
OSM תחת ODbL, ונתוני הבנ"טל/מפ"י כפופים לרישוי הייעודי שלהם.

---

## הערות פרודקשן

- **סודות:** קובץ `.env` לעולם לא נכנס ל-git. בפרודקשן העדף Docker/Kubernetes secrets.
- **קיבוע גרסאות:** החלף כל `:latest` ב-tag מקובע לפני פרודקשן.
- **אבטחת OpenSearch:** `DISABLE_SECURITY_PLUGIN=true` הוא לפיתוח בלבד — הפעל אבטחה בפרודקשן.
- **TLS:** Caddy מנפיק תעודות אוטומטית מול דומיין אמיתי (לא localhost).
- **גיבוי:** גבה את הווליומים `pgdata` ו-`geoserver_data` (ה-`data_dir`).
- **סקייל:** לסקייל לאומי — Kubernetes, read replicas ל-PostGIS, אשכול OpenSearch, ו-CDN מול ה-PMTiles.
