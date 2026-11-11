# test-clamav — Test-Repo für den Framework-Import-Security-Scan

Dieses Repository dient als Sync-Quelle (`GITHUB_SML_SYNC_REPO=Ir0nPyr0/test-clamav`),
um den Sicherheitsscan der Framework-Importe (ClamAV + Struktur-/Inhaltsprüfung)
mit dem Button **„Jetzt synchronisieren"** in der Admin-UI end-to-end zu testen.

Alle Dateien sind **harmlose Testdaten** (EICAR-Teststring, Text-Snippets) –
keine echten Schadprogramme.

## Benutzung

Backend-Umgebung für den Test umstellen (Backend neu starten):

```dotenv
GITHUB_SML_SYNC_REPO=Ir0nPyr0/test-clamav
GITHUB_SML_SYNC_REF=test/clamav-eicar-raw   # leer = main (Standard-Branch)
FRAMEWORK_IMPORT_CLAMAV_ADDRESS=localhost:3310
```

Dann in der Admin-UI **Administration → Frameworks → „Jetzt synchronisieren"**
klicken. Der Sync-Verlauf zeigt Status, `check` und `threat_name`; jeder Scan
landet zusätzlich als App-Ereignis `framework_import_scan` im Audit-Log.

## Branches

| Branch                   | Inhalt von `docs/sml_data.json`                | Erwarteter Ausgang                                             |
| ------------------------ | ---------------------------------------------- | -------------------------------------------------------------- |
| `main`                   | sauberes, schema-valides Manifest (v1.0)       | `success` — Import als neue Framework-Version                  |
| `test/clamav-eicar-raw`  | Datei = exakt der 68-Byte-EICAR-Teststring     | `blocked`, `check=malware`, `threat_name=Eicar-Test-Signature` (ClamAV) |
| `test/eicar-in-json`     | EICAR-String im Fragetext (schema-valide)      | `blocked`, `check=embedded_executable`, `threat_name=eicar-test-file` |
| `test/base64-executable` | Base64-PE-Header (`MZ…`) im Fragetext          | `blocked`, `check=embedded_executable`, `threat_name=base64-windows-pe` |
| `test/script-injection`  | `<script>`-Tag im Fragetext                    | `blocked`, `check=script_injection`, `threat_name=html-script-tag` |
| `test/ransomware-note`   | Erpresser-Text im Fragetext                    | `blocked`, `check=ransomware`, `threat_name=ransom-note-encrypted-files` |
| `test/invalid-schema`    | gültiges JSON, aber nur 3 von 9 Dimensionen    | `invalid_manifest` — Scan erlaubt, Parser lehnt ab, Admin-Alarm |

Hinweise:

- ClamAV erkennt EICAR nur am **Dateianfang**; deshalb testet `test/clamav-eicar-raw`
  den ClamAV-Signaturscan, während `test/eicar-in-json` die eingebaute
  Inhaltsregel (Payload in JSON-Strings) demonstriert.
- Ohne laufenden clamd blockt `test/clamav-eicar-raw` mit `check=structure`
  (ungültiges JSON) statt `check=malware`; mit clamd, aber ohne Fund-Warten,
  endet der Lauf `scan_unavailable` (fail-closed).
- Der Sync benötigt den S3-File-Store (`PUBLIC_SURVEY_UPLOAD_S3_BUCKET`,
  lokal z. B. MinIO aus dem Compose-Setup), sonst bleibt der Sync deaktiviert.
