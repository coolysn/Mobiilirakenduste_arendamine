# Speech Notes – võrgulahenduste dokumentatsioon

Kodutöö: **Võrgulahenduste koostamine**. Mobiilirakendus ühendatakse välise serveriga, et saata ja vastu võtta andmeid REST/HTTP(S) protokolli abil.

## 1. Rakenduse lühitutvustus

Speech Notes on mobiilirakendus, mis teeb kõnest eestikeelseid märkmeid (`et-EE`).

- **Tehnoloogiad:** Ionic Core (Web Components), TypeScript, Vite, Capacitor
- **Pluginad:** `@capacitor-community/speech-recognition`, `@capacitor/preferences`, `@capacitor/filesystem`
- **Olemasolevad funktsioonid:** märkmete dikteerimine ja salvestamine, kaustad, otsing, seaded, eksport/import failina, liitreaalsus (A-Frame / WebXR)

Kodutöö jaoks lisatud võrgufunktsioonid:

| Nupp | HTTP meetod | Eesmärk |
|---|---|---|
| **Tõlgi** | `GET` | Tõlgib eestikeelse märkme inglise keelde ja kuvab tõlke ekraanil |
| **Varunda Serverisse** | `POST` | Saadab avatud märkme serverisse |

Mõlemad nupud asuvad märkme detailvaates (`#detail-view`). Päringud tehakse brauseri sisseehitatud `fetch` funktsiooniga, lisapluginat pole vaja.

## 2. Kasutatud API-d

| Funktsioon | Allikas | Protokoll |
|---|---|---|
| Tõlge | [MyMemory Translation API](https://mymemory.translated.net/doc/spec.php): `https://api.mymemory.translated.net/get` | REST, HTTPS GET, JSON |
| Varundus | [JSONPlaceholder](https://jsonplaceholder.typicode.com/): `POST /todos` | REST, HTTPS POST, JSON |

**Miks MyMemory:** tasuta, ei vaja API-võtit ega registreerumist, lubab CORS-i (päringu saab teha otse äpist) ja toetab eesti keelt.

**Miks JSONPlaceholder:** tegemist on testserveriga, mis vastab POST-päringule nagu päris server (staatus `201 Created` ja uus `id`), kuid andmeid päriselt ei salvesta. Kodutöö lubab seda otseselt ("fake endpoint või reqres"). Tunnimaterjalis kasutatud reqres.in nõuab alates 2025. aastast igale päringule `x-api-key` päist, mille saamiseks peab registreeruma. JSONPlaceholder ei vaja võtit ja võtit ei pea hoidma äpi koodis.

## 3. GET – märkme tõlkimine

### Voog

1. Kasutaja avab märkme (klikk kaardil → `openDetail`).
2. Vajutab **Tõlgi**.
3. Nupp lukustub ja näitab laadimisindikaatorit (`ion-spinner`).
4. Rakendus saadab GET-päringu MyMemory API-le.
5. Vastusest võetakse `responseData.translatedText` ja see kuvatakse plokis "Inglise keeles".
6. Vea korral näidatakse punast toast-teadet "Tõlkimine ebaõnnestus".

### Päring

```
GET https://api.mymemory.translated.net/get?q=<tekst>&langpair=et|en
```

### Kood (`src/main.ts`)

```ts
async function translateNote(text: string): Promise<string> {
  const url = `https://api.mymemory.translated.net/get?q=${encodeURIComponent(text.slice(0, 450))}&langpair=et|en`;

  const res = await fetch(url);
  if (!res.ok) throw new Error(`Server vastas: ${res.status}`);

  const data = await res.json();
  if (data.responseStatus !== 200) throw new Error(data.responseDetails);

  return data.responseData.translatedText;
}
```

```ts
const translateBtn = document.getElementById('detail-translate')!;

translateBtn.addEventListener('click', async () => {
  if (!openNote) return;
  setLoading(translateBtn, true, 'Tõlgi');
  try {
    const translation = await translateNote(openNote.text);
    document.getElementById('detail-translation-text')!.textContent = translation;
    document.getElementById('detail-translation')!.classList.remove('ion-hide');
  } catch (err) {
    console.error('Tõlkimine ebaõnnestus', err);
    showToast('Tõlkimine ebaõnnestus', 'danger');
  } finally {
    setLoading(translateBtn, false, 'Tõlgi');
  }
});
```

### Näidisvastus (lühendatud)

```json
{
  "responseData": { "translatedText": "Buy milk and bread tomorrow." },
  "responseStatus": 200
}
```

### Veahaldus

- `!res.ok`: võrgu- või serveriviga (HTTP staatus ei ole 2xx)
- `responseStatus !== 200`: API enda viga (nt päevalimiit ületatud). MyMemory vastab sel juhul vahel HTTP 200-ga, seega tuleb kontrollida mõlemat.
- `try/catch/finally`: `finally` taastab nupu alati algseisu
- `showToast(..., 'danger')` annab kasutajale tagasisidet

## 4. POST – märkme varundamine serverisse

### Voog

1. Kasutaja avab märkme ja vajutab **Varunda Serverisse**.
2. Nupp lukustub ja näitab laadimisindikaatorit.
3. Rakendus saadab märkme teksti ja kausta JSON-kujul POST-päringuga.
4. Server vastab staatusega `201 Created` ja tagastab saadetud andmed koos uue `id`-ga.
5. Kasutajale näidatakse rohelist toast-teadet "Märkus salvestatud serverisse!", serveri vastus kirjutatakse konsooli.
6. Vea korral näidatakse punast toast-teadet "Serverisse salvestamine ebaõnnestus".

### Päring

```
POST https://jsonplaceholder.typicode.com/todos
Content-Type: application/json; charset=UTF-8

{ "name": "<märkme tekst>", "folderId": <kausta id või null> }
```

### Näidisvastus

```json
{ "name": "Osta homme piima ja leiba", "folderId": null, "id": 201 }
```

### Kood (`src/main.ts`)

```ts
async function testPostNote(payload: Omit<Note, 'id'>) {
  const res = await fetch('https://jsonplaceholder.typicode.com/todos', {
    method: 'POST',
    body: JSON.stringify({
      name: payload.text,
      folderId: payload.folderId,
    }),
    headers: {
      'Content-type': 'application/json; charset=UTF-8',
    },
  });
  if (!res.ok) {
    throw new Error(`Ei saanud märkust salvestada serverisse: ${res.status}`);
  }
  return res.json();
}
```

```ts
const backupBtn = document.getElementById('detail-backup')!;

backupBtn.addEventListener('click', async () => {
  if (!openNote) return;
  setLoading(backupBtn, true, 'Varunda Serverisse');
  try {
    const payload: Omit<Note, 'id'> = {
      text: openNote.text,
      folderId: openNote.folderId,
    };
    const data = await testPostNote(payload);
    console.log(data);
    showToast('Märkus salvestatud serverisse!', 'success');
  } catch (err) {
    console.error('Serverisse salvestamine ebaõnnestus', err);
    showToast('Serverisse salvestamine ebaõnnestus', 'danger');
  } finally {
    setLoading(backupBtn, false, 'Varunda Serverisse');
  }
});
```

`Omit<Note, 'id'>` tähendab "märge ilma `id` väljata". Märkme `id` jääb äpi sisse (see on märkme loomise aeg), server annab vastuses oma `id`.

## 5. Ühised abifunktsioonid

Mõlemad nupud kasutavad samu abifunktsioone:

```ts
async function showToast(message: string, color: 'success' | 'danger' = 'success') {
  const toast = await toastController.create({ message, color, duration: 2500, position: 'bottom' });
  await toast.present();
}

function setLoading(btn: HTMLElement, loading: boolean, label: string) {
  btn.toggleAttribute('disabled', loading);
  btn.innerHTML = loading ? '<ion-spinner name="dots"></ion-spinner>' : label;
}
```

- `showToast` näitab ekraani allservas teadet: roheline õnnestumise, punane vea korral.
- `setLoading` lukustab nupu päringu ajaks (et seda ei saaks mitu korda vajutada) ja näitab laadimisindikaatorit.

## 6. Mis oli lihtne

- **`fetch` + `async/await`** töötab Capacitori WebView's ilma lisapluginata, GET-päring on paar rida.
- **MyMemory ja JSONPlaceholder ei vaja API-võtit** ega registreerumist.
- **JSON-i parsimine:** `res.json()` annab valmis objekti.
- **Ühised `setLoading` ja `showToast` funktsioonid:** teine nupp sai kasutada esimese jaoks tehtud funktsioone.

## 7. Mis oli raske

- **reqres.in nõuab nüüd API-võtit:** tunnimaterjali näide ei töötanud ilma registreerumiseta, seepärast valisime POST-i jaoks JSONPlaceholderi.
- **Tõlke pikkuspiirang:** MyMemory lubab päringus umbes 500 baiti, seega tekst lõigatakse `slice(0, 450)` abil ja pikemad märkmed tõlgitakse osaliselt.
- **Päevalimiit:** anonüümselt umbes 5000 tähemärki päevas. Ületamisel tuleb viga `responseStatus` väljas, mitte HTTP staatusega, seega tuli kontrollida mõlemat.
- **Tõlke kvaliteet:** masintõlge eesti keelest pole alati täpne.
- **Eesti tähed URL-is:** `õ, ä, ö, ü` ja tühikud vajavad `encodeURIComponent`-i.
- **TypeScripti tüübiviga:** POST-i jaoks muudeti alguses `Note` liideses `id` valikuliseks (`id?: number`). See tekitas vead kohtades, kus `new Date(n.id)` näitab märkme kuupäeva. Lahendus: `id` jäeti kohustuslikuks ja POST-funktsioon võtab vastu tüübi `Omit<Note, 'id'>`.
- **Puuduv pakett pärast `git pull`-i:** `@capacitor/filesystem` puudus kohalikust `node_modules` kaustast. Lahendus: `npm install`.
- **Testserver ei salvesta:** JSONPlaceholder ei jäta andmeid alles, seega varundust ei saa hiljem tagasi lugeda. Kontrollida saab ainult vastuse staatust ja `id`-d.

## 10. Allikad

- [MyMemory API dokumentatsioon](https://mymemory.translated.net/doc/spec.php)
- [JSONPlaceholder](https://jsonplaceholder.typicode.com/)
- [MDN: Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Ionic: Toast](https://ionicframework.com/docs/api/toast)
