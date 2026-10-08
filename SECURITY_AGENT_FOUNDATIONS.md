# Security Agent – alapok (munkadokumentum, v0.1)

Ez egy **élő dokumentum**. Célja, hogy a Secure Code Game mind négy évadából (Season-1 – Season-4) levont tanulságokat egy újrafelhasználható formába rendezze, amire később egy saját biztonsági-review agent épülhet a felhasználó saját projektjeihez.

**Állapot:** a Season-1, Season-2 és Season-3 teljesen végigjátszva; a Season-4-et a felhasználó még nem játszotta végig személyesen (megállt, hogy előbb több AI-biztonsági alapot szerezzen), de Claude átnézte és dokumentálta mind az 5 szintjét. A dokumentum ezért **korai, bővítendő állapotban van** — lásd az 5. szakaszt a még nyitott kérdésekről, amik a végső formáját befolyásolják.

**Konvenció:** a magyarázó szöveg magyar, a szakmai terminológia (CWE-azonosítók, OWASP-kategóriák, kódrészletek, angol technikai kifejezések) angolul marad, mert így kereshető/összevethető a nemzetközi forrásokkal.

---

## 1. Cél és terjedelem

A végső security agent célja: **statikus kódátvizsgálás** egy projektben, a lenti mintakatalógus és visszatérő elvek alapján, hogy minél több sebezhetőség-osztályt megtaláljon és javítási javaslatot adjon — **nem** csak a játékban látott konkrét kódrészletek felismerése, hanem az **általánosított minták** alkalmazása ismeretlen kódra is.

A dokumentum két fő részből áll:
- **2. szakasz** — visszatérő, nyelv- és keretrendszer-független minták ("code smell"-ek), amik 2+ különböző szinten/évadban is előfordultak. **Ez a legfontosabb rész az agent szempontjából**, mert ez az, ami ismeretlen kódra is általánosítható.
- **3. szakasz** — a konkrét sebezhetőség-katalógus, szintenkénti bontásban, referencia célokra.

## 2. Visszatérő minták ("code smell"-ek az agent számára)

### 2.1 Denylist/blokklista csapda

**A minta:** egy szűrő/validátor egy **ismert, konkrét** mintázatot (karaktert, kulcsszót, parancsot) tilt meg, ahelyett hogy a **megengedett** formát írná le pozitívan (allowlist).

**Hol fordult elő:** Season-1 L3 (`startswith('..')` útvonal-ellenőrzés — sibling-directory bypass), Season-2 L4 (XSS: `[<>{}[\]]` regex, de a szemikolon nélküli HTML-entitásokat nem ismeri fel), Season-3 L5/L6 (input-filter: `secret`/`reveal`/`story`/`game` szavak tiltása — implicit megfogalmazással kikerülhető), Season-4 L1 (bash `DENIED_PATTERNS` regex — kódolással/indirekcióval kikerülhető).

**Detekciós heurisztika az agent számára:** keress mintákat, ahol a validáció `blacklist`/`denylist`/`forbidden`/`blocked` jellegű listát hasonlít össze nyers bemenettel (`includes`, `match`, reguláris kifejezés tiltott mintára), **anélkül** hogy a bemenetet előbb normalizálná (dekódolná, kisbetűsítené, Unicode-ot egységesítené) vagy a végeredményt (a tényleges, feloldott értéket) validálná.

**Hamis pozitív:** egy denylist önmagában nem feltétlenül hiba, ha **védelem mélységként** (defense-in-depth) szerepel egy erősebb, pozitív ellenőrzés mellett — csak akkor súlyos, ha ez az **egyetlen** védelmi réteg.

### 2.2 Felszín (claimed) vs. valós (actual) viselkedés

**A minta:** egy komponens (függvény, eszköz, library-konfiguráció, system prompt) **leírása/dokumentációja/neve** mást állít, mint amit a **tényleges kódja** csinál.

**Hol fordult elő:** Season-2 L3 (XXE: a parser "biztonságosnak" tűnő alapértelmezése valójában `replaceEntities: true`), Season-4 L3 (MCP-eszköz leírása "sandbox only", de `BASE_DIR` a teljes szülőmappára mutat), Season-4 L5 (agent system promptja "pre-verified"/"schema-validated" adatot állít, valójában semmilyen validáció nincs).

**Detekciós heurisztika:** minden olyan helyen, ahol egy komponens hozzáférési kört/hatókört/garanciát **állít** (kommentben, system promptban, README-ben, függvénynévben), ellenőrizni kell, hogy ez **egyezik-e a tényleges implementációval** (pl. egy `BASE_DIR`, egy `scope` config, egy parser-opció). Az agent sosem fogadhatja el a dokumentációt bizonyítékként — mindig a forráskódot kell auditálnia.

### 2.3 Statikus szöveg-validáció dinamikus kiértékelés ellen

**A minta:** egy validátor a **nyers szöveget** elemzi (regex, substring-keresés), mielőtt egy **másik, dinamikus interpretert** (shell, SQL-motor, sablon-nyelv, reguláris kifejezés motor) futtatna rá — de az interpreter **futásidőben** másképp "látja" az értéket (változó-behelyettesítés, kódolás/dekódolás, escaping).

**Hol fordult elő:** Season-4 L1 (bash validátor a `..`-ot keresi a szövegben, de `D=..` + `$D` változó-expanzió vagy base64-dekódolás kikerüli), Season-1 L4 (SQL string-konkatenáció — a "szöveg" és a "lekérdezés, amit a DB tényleg lát" eltér egymástól).

**Detekciós heurisztika:** keress olyan validációt, ami **a parancs/lekérdezés/sablon elkészítése ELŐTT** fut a nyers inputon, miközben a tényleges végrehajtás **változó-behelyettesítést, string-konkatenációt vagy dekódolást** tartalmaz utána. Az ökölszabály: *validálj a feloldás UTÁN, a tényleges végrehajtott érték alapján — ne a forrásszöveg alapján.*

### 2.4 Bizalmi határ összeomlása: adat és utasítás egy csatornán

**A minta:** egy rendszer (jellemzően egy LLM-alapú agent) **nem különbözteti meg** a "feldolgozandó adatot" és a "követendő utasítást", mert mindkettő **ugyanabban a kontextusban/csatornában** érkezik.

**Hol fordult elő:** Season-2 L3 (XXE SYSTEM entitás mint beágyazott "utasítás" az XML "adatban"), Season-3 összes szintje (a system prompt és a user prompt elvileg el van választva, de a modell "meggyőzhető" egy elég jó promptal), Season-4 L2 (indirekt prompt-injekció: weboldal HTML-je mint "adat", de rejtett utasítást tartalmaz), Season-4 L5 (confused deputy: több agent/eszköz adata egy "pre-verified" kontextusba keverve).

**Detekciós heurisztika:** bármely data flow, ahol **külső, nem megbízható forrásból** (weboldal, API-válasz, fájltartalom, más agent kimenete) érkező szöveg **közvetlenül, szűrés/strukturális elválasztás nélkül** kerül be egy LLM promptjába vagy egy interpreter bemenetébe.

### 2.5 Megosztott, nem védett állapot biztonsági döntéshez

**A minta:** egy biztonsági döntést (jogosultság, hatókör, validációs szabály) befolyásoló állapot **ugyanabban a tárolóban** van, mint a sima, bármely komponens által szabadon írható adat — nincs integritás-ellenőrzés arról, KI írta.

**Hol fordult elő:** Season-2 L2 (Go: package-level megosztott `reqBody` — nem biztonsági policy, de ugyanaz az elv: megosztott mutábilis állapot, amiben megbízik a kód), Season-4 L4 (a `.memory` fájl egyszerre tárol felhasználói preferenciát ÉS a validátor által olvasott `scope=workspace` biztonsági jelzőt — bármelyik "org-approved" skill írhatja).

**Detekciós heurisztika:** keress olyan konfigurációs/állapot-tárolót, amit **egyszerre** olvas egy biztonsági ellenőrzés ÉS ír egy nem-privilegizált komponens (plugin, skill, kérés-feldolgozó), integritás-ellenőrzés (aláírás, külön jogosultsági szint) nélkül.

### 2.6 Gyenge kriptográfia / "security through obscurity"

**A minta:** elavult hash-algoritmus (MD5), gyenge/kiszámítható véletlenszám-forrás (`random` modul PRNG helyett CSPRNG), hardcode-olt titok a forráskódban, vagy a védelem "homályosságra" (obfuszkációra) épül ahelyett, hogy kriptográfiailag garantált lenne.

**Hol fordult elő:** Season-1 L5 (MD5, `random`-alapú saltmaradt , hardcode-olt `SECRET_KEY`).

**Detekciós heurisztika:** grep `md5`, `random.choice`/`random.randint` (Python) kriptográfiai kontextusban (token, salt, jelszó) helyett `secrets`/`crypto.randomBytes`; hardcode-olt stringek, amik kulcsnak/tokennek "néznek ki" forráskódban.

### 2.7 LLM-specifikus: a system prompt nem hozzáférés-vezérlés

**A minta:** egy natural-language utasítás (system prompt, "önellenőrző" LLM-hívás) **advisory** jellegű — statisztikai ösztönzés, nem kőbe vésett szabály —, mégis **ezen** alapul egy biztonsági döntés.

**Hol fordult elő:** Season-3 összes szintje, különösen L4/L5 (az "önellenőrző" modell maga is csak egy promptolható, nem-determinisztikus LLM, nem egy valódi access-control kapu).

**Detekciós heurisztika (LLM-alapú rendszerekre specifikusan):** ha egy rendszerben a **kizárólagos** védelem egy prompt-instrukció vagy egy másik LLM-hívás ítélete, és nincs mögötte **determinisztikus, kódszintű** kapu (output-regex, valódi auth-token, paraméterezett/korlátozott lekérdezés), ez jelzésértékű — függetlenül attól, hogy a prompt mennyire "szigorúnak" tűnik.

---

## 3. Sebezhetőség-katalógus (szintenkénti referencia)

> Formátum soronként: **Azonosító** — CWE/OWASP — *Minta* → *Javítás*. A részletes "miért működik" magyarázatok a beszélgetés-előzményben (ill. a [[project-security-agent-idea]] memóriafájlban) megtalálhatók; itt a gyors kereshetőség a cél.

### Season 1 — klasszikus alkalmazásbiztonság (Python/C)

- **S1L1 — Üzleti logikai hiba / lebegőpontos pontatlanság.** CWE-1339/CWE-682. *Minta:* pénzösszegek `float`-ként kezelve, felső korlát ellenőrzése a ciklus belsejében (sorrendfüggő). → *Javítás:* `Decimal`, a teljes összeg ellenőrzése a ciklus UTÁN, `abs()` is.
- **S1L2 — Out-of-bounds write negatív index miatt.** CWE-787/CWE-129. *Minta:* `if (*endptr || i >= SETTINGS_COUNT)` — hiányzik az `i < 0` ág, negatív index átfedhet más struct-mezővel. → *Javítás:* explicit alsó és felső korlát egyszerre.
- **S1L3 — Path traversal.** CWE-22. *Minta:* `path.startswith('/') or path.startswith('..')` blokklista. → *Javítás:* `os.path.commonpath([base_dir, full_path]) == base_dir` (a hivatalos `solution.py` `startswith(base_dir)`-je is bypassolható sibling-mappával!).
- **S1L4 — SQL injection.** CWE-89. *Minta:* string-konkatenáció a lekérdezésben. → *Javítás:* paraméterezett `?` placeholder; a kódban maradó arbitrary-SQL metódusokhoz `sqlite3.set_authorizer` read-only védelem.
- **S1L5 — Gyenge kriptográfia.** CWE-327/CWE-330/CWE-798. *Minta:* MD5, `random`-alapú salt, hardcode-olt `SECRET_KEY`. → *Javítás:* bcrypt, `secrets` modul, `SECRET_KEY` env-ből + rotáció.

### Season 2 — webalkalmazás és CI/CD (JS, Go, YAML)

- **S2L1 — Ellátási lánc / CI secrets.** CWE-829/CWE-798. *Minta:* harmadik féltől származó GitHub Action (SHA-pin **nem** = bizalom) + hardcode-olt token `with:` blokkban, ami bekerül az Action logjába. → *Javítás:* közvetlen `curl`/`jq` lépés, token GitHub Secrets-ből, `pipefail` explicit beállítása.
- **S2L2 — Megosztott mutábilis állapot → auth bypass + race condition.** CWE-362-rokon. *Minta:* package-level `var reqBody` Go HTTP handlerben, `json.Decode` nem törli a hiányzó mezőket. → *Javítás:* `reqBody` lokális változóként a handleren belül. (Nem volt a hivatalos megoldásban dokumentálva — önálló lelet, `go test -race` igazolta.)
- **S2L3 — XXE + rejtett RCE backdoor.** CWE-611/CWE-434/CWE-506. *Minta:* `libxmljs.parseXml(..., {replaceEntities:true, recover:true, nonet:false})` + `.admin` kiterjesztés `exec()`-et hív. → *Javítás:* `replaceEntities:false, recover:false, nonet:true`, a backdoor-ág törlése.
- **S2L4 — Reflected XSS blokklista-bypass.** CWE-79. *Minta:* `[<>{}[\]]` regex nem fogja a szemikolon nélküli HTML5 entitásokat (`&lt`/`&gt`), `| safe` kikapcsolja a Jinja2 auto-escape-et, egy második JS-sor `innerHTML`-be írja a már dekódolt szöveget. → *Javítás:* a regex törlése, `| safe` eltávolítása (alapértelmezett autoescape), `innerHTML =` → `textContent =`.
- **S2L5 — JS dinamikus nyelvi viselkedés kihasználása.** CWE-1321-rokon/CWE-20. *Minta:* (1) implicit `+=` coerció hívja egy objektum `toString()`-jét; (2) `obj.prop.method()` minden hívásnál újra kikeresve, kívülről felülírható; (3) `w = []` sparse tömb, index-írás `Array.prototype` setterre esik át. → *Javítás:* `typeof` guard; lokális closure-referencia a metódusra definiáláskor; tömb előre, teljes méretben inicializálva.

### Season 3 — LLM-alapú rendszerek, prompt-biztonság

- **S3 L1–L6 — System prompt mint (nem elégséges) hozzáférés-vezérlés.** OWASP LLM01 (Prompt Injection) + LLM02 (Sensitive Information Disclosure). *Minta:* a titok kizárólag natural-language szabályokkal védett; rétegek szintről szintre: maszkolás → "user ID" (nem valódi auth) → kódszintű output-regex → LLM-as-judge self-verification → input denylist → (L6) korlátlan LLM-vezérelt SQL tool-call. *Közös javítási elv:* determinisztikus kódszintű kapuk (valódi auth, paraméterezett/korlátozott query LLM tool-use esetén is, szigorú output-allowlist) — egy másik LLM ítélete vagy egy prompt-szabály **sosem** garancia.

### Season 4 — agentikus AI-rendszerek ("ProdBot")

- **S4L1 — Sandbox Escape.** CWE-77/78-rokon + 2.3 minta. *Minta:* `bash.js` statikus regex a `..`-ra, futásidejű változó-expanzió/base64 megkerüli. → *Javítás:* a feloldott útvonal validálása `path.resolve()` után, vagy OS-szintű sandboxing.
- **S4L2 — Indirect Prompt Injection.** OWASP LLM01 (indirekt variáns) + 2.4 minta. *Minta:* weboldal nyers HTML-je (rejtett komment/`display:none`) szűrés nélkül kerül az LLM kontextusába. → *Javítás:* külső tartalom sanitizálása (kommentek/rejtett elemek törlése) LLM-nek adás előtt.
- **S4L3 — Excessive Agency.** OWASP LLM06 + 2.2 minta. *Minta:* MCP-eszköz leírása "sandbox only", kódja `BASE_DIR = path.resolve(__dirname, "..")` — a teljes szülőmappa. → *Javítás:* legkisebb jogosultság, forráskód-audit minden eszközre telepítés előtt.
- **S4L4 — Supply Chain Poisoning (megosztott policy-tár).** OWASP LLM03/LLM06 + 2.5 minta. *Minta:* egy "org-approved" skill `ttl=0`-val ír egy perzisztens `scope=workspace` bejegyzést, amit a validátor biztonsági döntéshez olvas. → *Javítás:* a policy-tár elválasztása a plugin-adattártól, integritás-ellenőrzés.
- **S4L5 — Confused Deputy multi-agent rendszerben.** CWE-441 + OWASP LLM06 + 2.4/2.2 minta. *Minta:* a "Release Agent" állítottan read-only, valójában teljes workspace-hozzáférése van, és megbízik a "Research Agent" állítottan "pre-verified" (valójában szűrés nélküli) adatában. → *Javítás:* minden agent-handoff önálló validációja, zero-trust az agentek között is.

---

## 4. Agent-szintű módszertani tanulságok (nem konkrét sebezhetőségek)

- **Mindig futtasd le az exploitot**, mielőtt bármilyen javítást "késznek" nyilvánítanál — egy blokklista sosem számít javításnak önmagában (lásd 2.1).
- **A hivatalos megoldások is tartalmazhatnak hibát** — a Season-1 L3 hivatalos `solution.py`-ja maga is bypassolható volt sibling-mappával. Mindig önállóan ellenőrizz, sose fogadj el egy "megoldást" vakon.
- **A tesztek és a hivatalos megoldás ellentmondhatnak egymásnak** (Season-1 L4/L5) — ha egy teszt olyan metódust hív, amit a megoldás törölni javasol, ez jelzés, hogy a döntést dokumentálni kell, nem automatikusan az egyik félnek adni igazat.
- **"Design smell" ≠ kihasználható hiba** — külön kell jelölni a kódszagokat (pl. dupla számláló-növelés, hiányzó NULL-check) a tényleges, bizonyítottan kihasználható sebezhetőségektől, hogy a riport ne vesszen el túl sok alacsony-értékű találatban.
- **Valódi eszközökkel párosítva** (CodeQL, Semgrep, Bandit, dependency scanner) az agent megbízhatóbb, mint a modell önmagában — és a találatokat **reprodukálni** kell (futó exploit vagy legalább egy konkrét bemenet-kimenet pár), mielőtt riportba kerülnének, hogy csökkenjen a hamis pozitívok száma.

## 5. Nyitott kérdések — a felhasználónak (a dokumentum további formáját ezek befolyásolják)

1. **Milyen nyelveket/stackeket** használnak a saját projektjeid, amiken ezt az agentet majd futtatnád? (A játékban eddig látott: Python, C, Go, JS/Node, YAML/GitHub Actions, LLM/agentikus rendszerek.)
2. **Milyen formában** éljen az agent? (Claude Code skill/subagent, CI-pipeline lépés, PR-reviewer bot, önálló CLI-eszköz?)
3. **Csak statikus átvizsgálás**, vagy fusson is exploitokat egy elkülönített sandboxban a megerősítéshez?
4. **Riport-formátum és súlyossági skála** — mire lenne szükséged a végén (pl. CWE-azonosító, súlyosság, konkrét sor, javítási diff)?
5. **Hol éljen ez a dokumentum/tudásbázis** a végén (ez a repo, egy skill-fájl, valami más), és milyen nyelven (ez most magyar prózával + angol szakterminológiával készült — ez jó így véglegesen is)?
6. **Engedélyezett célpontok** — csak a saját projektjeid, vagy bármi, amire explicit engedélyed van?

## 6. Következő lépések

- [ ] Season-4-et magad is végigjátszani, ha úgy döntesz, hogy visszatérsz rá (jelenleg szüneteltetve)
- [ ] A fenti nyitott kérdések megválaszolása
- [ ] A katalógus bővítése, ha új szintekkel/évaddal folytatod a játékot
- [ ] Az agent tényleges megépítése (skill/subagent/CI-eszköz — a 2. kérdés válasza alapján)
