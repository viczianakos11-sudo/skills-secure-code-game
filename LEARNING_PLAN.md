# Webes működés és biztonság – tanulási terv

Cél: megérteni, hogyan működnek a weboldalak, hogyan néz ki egy oldal az F12 (DevTools) szemével, és milyen biztonsági hibák keletkezhetnek (pl. a Secure Code Game Level-1 üzleti logikai hibája).

Szabály: biztonsági tesztelést **csak saját vagy kifejezetten erre készült rendszeren** végezz (labor, Juice Shop, CTF). Idegen oldal engedély nélküli tesztelése jogilag is problémás.

---

## 1. lépés – Az F12 első megismerése

**Cél:** lásd, mit küld a böngésző a szervernek, és mit kap vissza.

- Nyiss meg bármilyen oldalt, nyomj F12-t.
- **Network** fül: töltsd újra az oldalt, kattints egy kérésre, nézd meg a *Headers*, *Payload*, *Response* részeket.
- **Elements** fül: nézd meg a HTML szerkezetét, kattints egy elemre, módosítsd a szövegét (csak a saját böngésződben változik).
- **Application** fül: nézd meg a cookie-kat.
- Forrás: [Chrome DevTools dokumentáció](https://developer.chrome.com/docs/devtools)

**Kész, ha:** meg tudod mondani, melyik kérés vitte el egy űrlap adatait, és mi volt a benne lévő JSON/űrlapadat.

## 2. lépés – HTTP alapok

**Cél:** értsd a kérés–válasz modellt, amin minden webes támadás és védekezés alapul.

- Metódusok: GET, POST, PUT, DELETE.
- Állapotkódok: 200, 301, 400, 401, 403, 404, 500.
- Fejlécek (headers), cookie-k, kérés- és választörzs (body).
- Forrás: [MDN – HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)

**Kész, ha:** el tudod magyarázni, mi történik a böngésző és a szerver között, amikor egy gombra kattintasz.

## 3. lépés – HTML, JavaScript és a kliens–szerver határ

**Cél:** értsd, mit lehet a böngészőben kikerülni, és mit nem.

- HTML űrlapok (`<form>`, `<input>`, `min`, `max`, `required`).
- JavaScript alapok, DOM.
- Miért nem védelem a böngészőoldali ellenőrzés?
- Forrás: [MDN – Learn web development](https://developer.mozilla.org/en-US/docs/Learn)

**Kész, ha:** meg tudod magyarázni, miért kell a szerveren is mindent újra ellenőrizni (Level-1 tanulsága).

## 4. lépés – A teljes kép: hogyan áll össze egy webalkalmazás

**Cél:** böngésző → szerver → üzleti logika → adatbázis → válasz.

- Mi fut a böngészőben, mi a szerveren?
- Mi az API, mi a JSON?
- Mi a "bizalmi határ"?

**Kész, ha:** le tudod rajzolni egy webshop rendelésének útját, és megjelölöd, hol kell az adatot ellenőrizni.

## 5. lépés – OWASP Top 10

**Cél:** szókincs és áttekintés a leggyakoribb webes sebezhetőségekről.

- Különösen: Broken Access Control, Injection, Insecure Design.
- Forrás: [OWASP Top 10](https://owasp.org/www-project-top-ten/)

**Kész, ha:** legalább 5 kategóriát el tudsz magyarázni egy-egy példával.

## 6. lépés – Gyakorlás: PortSwigger Web Security Academy

**Cél:** valódi laborokban kipróbálni a sebezhetőségeket (böngészőből, telepítés nélkül).

- Kezdd a **Business logic vulnerabilities** témával (a Level-1 közvetlen folytatása).
- Utána: Access control, SQL injection, XSS.
- Forrás: [Web Security Academy](https://portswigger.net/web-security)

**Kész, ha:** megoldottál legalább 3 labort az üzleti logikai témában.

## 7. lépés – Saját sebezhető webshop: OWASP Juice Shop

**Cél:** egy teljes, szándékosan sebezhető webshopot vizsgálni a saját gépeden / Codespace-ben.

- Forrás: [OWASP Juice Shop](https://owasp-juice.shop/)
- Használd közben az F12-t és a Network fület.

**Kész, ha:** találtál és megoldottál néhány kihívást, és el tudod mondani, mi volt a hiba oka.

## 8. lépés – Vissza a Secure Code Game-hez

**Cél:** a tanultak alkalmazása a következő szinteken (Season-1 Level-2 és tovább).

- Minden szintnél kérdezd meg: ki adja az inputot? Mi történik, ha rosszindulatú?

---

## Haladás

- [ ] 1. F12 első megismerése
- [ ] 2. HTTP alapok
- [ ] 3. HTML, JS, kliens–szerver határ
- [ ] 4. A teljes kép
- [ ] 5. OWASP Top 10
- [ ] 6. PortSwigger Academy
- [ ] 7. Juice Shop
- [ ] 8. Secure Code Game folytatása
