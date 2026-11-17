# Android-APK erstellen und signieren

## Automatischer APK-Build (GitHub Actions)

Jeder Push auf `main` startet `.github/workflows/build-apk.yml` und erzeugt zwei
Workflow-Artefakte:

| Artifact | Inhalt | Status/Zweck |
|---|---|---|
| `agenten-villa-debug-apk` | `app-debug.apk` | Debug-Build |
| `agenten-villa-release-apk` | `app-release.apk` | Release-Variante; ohne Keystore **nicht signiert** |

Die Artefakte werden 30 Tage aufbewahrt. Ein erfolgreicher Build allein bedeutet
nicht, dass die APK signiert oder als GitHub Release veröffentlicht wurde.
Wenn kein Signaturschlüssel konfiguriert ist, überspringt die CI ausdrücklich die
Signaturprüfung. Die am 17.11.2026 geprüfte Ausführung war erfolgreich, hatte aber
keinen konfigurierten Signierschlüssel; ihre Release-APK ist daher nicht als
Produktionsrelease freigegeben.

## Signierte Veröffentlichung (GitHub Release)

Für einen echten signierten Release ist der manuell startbare Workflow
`.github/workflows/publish-android-release.yml` vorgesehen. Er baut die APK,
prüft sie mit `apksigner` und veröffentlicht sie erst danach als GitHub Release.

Vor dem Start müssen in **Repository → Settings → Secrets and variables → Actions**
alle vier folgenden Secrets gesetzt sein:

- `ANDROID_KEYSTORE_BASE64` – Base64-kodierter Android-Keystore
- `ANDROID_KEYSTORE_PASSWORD`
- `ANDROID_KEY_ALIAS`
- `ANDROID_KEY_PASSWORD`

Danach unter **Actions → Publish signed Android release → Run workflow** einen
neuen Tag (z. B. `v1.0.0`) und den Quell-Ref (standardmäßig `main`) angeben.
Der Workflow bricht ab, wenn ein Secret fehlt oder die Signaturprüfung fehlschlägt.
Secret-Werte niemals in Issues, Commits oder Chatnachrichten einfügen.

Wichtig: Für App-Updates muss derselbe private Signierschlüssel dauerhaft und sicher
aufbewahrt werden. Einen neuen Keystore nicht erzeugen, wenn bereits eine installierte
App mit einem anderen Schlüssel verteilt wurde; Android akzeptiert dann kein
In-place-Update derselben App-Signatur.

## Signaturschlüssel sichern

Der Keystore gehört ausschließlich in GitHub Actions Secrets und in ein sicheres
externes Backup – niemals ins Repository. Ein Keystore-Generator erstellt einen
neuen Schlüssel; er ersetzt nicht den bereits verwendeten Release-Schlüssel.

## Manuell bauen

```bash
pnpm install
VITE_API_URL=https://agenten-villa.onrender.com pnpm exec vite build
pnpm exec cap sync android
cd android && ./gradlew assembleRelease
```

Für eine signierte lokale Release-APK die vier Umgebungsvariablen von oben setzen
und zusätzlich `ANDROID_KEYSTORE_FILE` auf den Keystore-Pfad zeigen lassen.

## Google-Anmeldung (nativer Sign-In) einrichten

Der native Sign-In läuft über `@codetrix-studio/capacitor-google-auth`. Damit er
funktioniert, müssen im **selben Google-Cloud-Projekt** zwei OAuth-Clients existieren:

1. **Web-Client** – dessen ID steht in `capacitor.config.ts` unter
   `plugins.GoogleAuth.clientId` und in der Server-Umgebung als `GOOGLE_CLIENT_ID`.
   Der Server prüft das `aud`-Claim des ID-Tokens gegen diesen Wert.
   Produktion: `656137332727-2i47bqt2fk359nj48tc0kdfl3mhjmoo1.apps.googleusercontent.com`
   (identisch in `capacitor.config.ts` und `strings.xml`; ein Regressionstest
   in `server/_core/native-client-config.test.ts` prüft die Konsistenz).
   Fuer den Web-Redirect-Flow muss dieser Client zusaetzlich die autorisierte
   Redirect-URI `https://agenten-villa.onrender.com/api/auth/google/callback`
   besitzen.
2. **Android-Client** – mit dem Paketnamen `de.niknight1403.agentenvilla` und dem
   SHA-1-Fingerprint des **signierenden** Keystores.

Fehlt der Android-Client oder passt der Fingerprint nicht, liefert Google
`ApiException` mit Code 10 (`DEVELOPER_ERROR`). Die App zeigt dafür einen eigenen
Hinweis statt einer generischen Fehlermeldung.

### Fingerprints der Keystores

| Keystore | SHA-1 | SHA-256 |
|---|---|---|
| Lokaler Release-Keystore (Berlin) | `c51dacfc5712138a2eee8476d25df6d6319f7bdf` | `26b1346f7847ff7bc5e2906c6f3f3cdd8ec5447d81b45ccf4917754bdbd0ce85` |
| CI-Keystore (Frankfurt, GitHub-Secrets) | `B7:60:BF:24:1E:50:A8:04:FF:68:25:5A:52:C9:97:20:CA:17:AE:65` | `5A:5E:9D:B1:DC:19:F7:B9:C0:93:7A:77:2C:16:09:2C:FD:EB:52:C5:D5:EA:C9:27:7F:79:61:99:1D:8E:87:87` |

Den SHA-1 des jeweils verteilten APKs prüfen mit:

```bash
$ANDROID_HOME/build-tools/<version>/apksigner verify --print-certs app-release.apk
```

Der CI-Workflow gibt die Fingerprints beim Build zusätzlich im Log aus.

### Scopes müssen ein String sein

`GoogleAuth.java` liest die Scopes mit `getConfig().getString("scopes")`, obwohl der
TypeScript-Typ `string[]` deklariert. Wird dort ein Array hinterlegt, wirft `org.json`
eine `JSONException`, der Default `""` greift, und das Plugin bildet daraus
`scopeArray = [""]`. Der Aufruf `new Scope("")` wirft in Google Play Services
`IllegalArgumentException`, was Capacitor als `RuntimeException` weiterreicht und den
App-Prozess beendet — die App stürzt beim Antippen von „Anmelden" ab.

Deshalb in `capacitor.config.ts` **und** im `initialize()`-Aufruf:

```ts
scopes: "profile,email"
```

`server/native-google-auth-config.test.ts` verhindert einen Rückfall auf Array-Scopes.
