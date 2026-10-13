# LabDoku Chart-Dokumentation und Values-Validierung

**Voraussetzung:** Das Chart `webapp` aus den bisherigen Übungen ist vorhanden.  
**Ziel:** Dokumentieren und validieren Sie Konfigurationswerte und überprüfen Sie die Auswirkungen gültiger und ungültiger Upgrades auf einem Kubernetes-Cluster.

1. In den bisherigen Übungen haben Sie eigene Werte in der Datei `values.yaml` definiert und diese über Templates verarbeitet. Bei fremden Charts stellt sich schnell die Frage, welche Werte überhaupt möglich sind und welche davon zwingend angegeben werden müssen. In dieser Übung ergänzen Sie Ihr bestehendes Chart `webapp` um eine Dokumentation und eine automatische Validierung.

2. Öffnen Sie Visual Studio Code und wechseln Sie in den Ordner `meine_charts`. Prüfen Sie, ob das Chart `webapp` mit den Dateien `Chart.yaml`, `values.yaml` und dem Ordner `templates` vorhanden ist. Wir arbeiten weiterhin mit diesem Chart. Die bisherigen Templates sollen erhalten bleiben.

3. Erstellen Sie zunächst eine Sicherung Ihrer `values.yaml`. Öffnen Sie dazu ein Terminal und führen Sie folgenden Befehl aus:

   ```bash
   cp webapp/values.yaml webapp/values.yaml.bak
   ```

4. Öffnen Sie die Datei `webapp/values.yaml`. Ergänzen Sie die folgenden Werte **auf oberster Ebene**. Bereits vorhandene Keys wie `replicaCount` oder `imageTag` dürfen nicht doppelt angelegt werden. Ändern Sie stattdessen den vorhandenen Wert und ergänzen Sie den Kommentar.

   ```yaml
   # replicaCount -- Anzahl der gewünschten Webserver-Pods
   replicaCount: 2

   # imageTag -- Version des Webserver-Images
   imageTag: "1.0"

   # documentation -- Informationen für die Chart-Dokumentation
   documentation:
     # documentation.owner -- Zuständiges Team
     owner: "training"
     # documentation.environment -- Zielumgebung
     environment: "development"
     # documentation.contact -- Optionaler Kontakt
     contact: ""
   ```

5. Achten Sie auf die Einrückung. Der Block `documentation` ist ein neuer Key, der unsere bisherigen Deployments nicht verändert. Speichern Sie die Datei ab.

6. Erstellen Sie im Hauptordner `webapp` eine neue Datei mit dem Namen `README.md`. Fügen Sie folgenden Inhalt ein:

   ````markdown
   # webapp

   Dieses Helm-Chart stellt die Webanwendung aus dem Helm-Kurs bereit.

   ## Installation

   ```bash
   helm upgrade --install dev-app ./webapp --namespace dev --create-namespace
   ```

   ## Konfigurationswerte

   | Parameter | Beschreibung | Standardwert |
   | --- | --- | --- |
   | replicaCount | Anzahl der Webserver-Pods | 2 |
   | imageTag | Version des Container-Images | "1.0" |
   | documentation.owner | Zuständiges Team | training |
   | documentation.environment | Zielumgebung | development |
   | documentation.contact | Optionaler Kontakt | leer |

   Die angegebenen Werte können in der values.yaml oder über --set geändert werden.
   ````

7. Überlegen Sie, was passiert, wenn Sie einen neuen Key in der `values.yaml` ergänzen, aber die README nicht aktualisieren. Kann Helm erkennen, dass die Dokumentation unvollständig ist?

8. Prüfen Sie, ob Helm Ihr Chart weiterhin verarbeiten kann:

   ```bash
   helm lint ./webapp
   helm template dev-app ./webapp --namespace dev > /tmp/webapp-rendered.yaml
   ```

   **Hinweis:** Führen Sie die Befehle aus dem Ordner `meine_charts` aus. Sollte das bestehende Chart bereits einen Fehler enthalten, beheben Sie diesen zunächst. Für die folgenden Schema-Tests muss das Chart korrekt rendern.

9. Eine README hilft dem Administrator bei der Konfiguration. Sie verhindert aber nicht, dass ungültige Werte verwendet werden. Wir ergänzen daher eine maschinenlesbare Definition unserer Values. Erstellen Sie im Hauptordner `webapp` die Datei `values.schema.json`.

10. Fügen Sie den folgenden Inhalt ein. Achten Sie darauf, dass JSON keine Kommentare erlaubt:

   ```json
   {
     "$schema": "https://json-schema.org/draft-07/schema#",
     "type": "object",
     "properties": {
       "replicaCount": {
         "type": "integer",
         "minimum": 1,
         "maximum": 5,
         "description": "Anzahl der Webserver-Pods"
       },
       "imageTag": {
         "type": "string",
         "minLength": 1
       },
       "documentation": {
         "type": "object",
         "properties": {
           "owner": { "type": "string", "minLength": 1 },
           "environment": {
             "type": "string",
             "enum": ["development", "testing", "production"]
           },
           "contact": { "type": "string" }
         },
         "required": ["owner", "environment"]
       }
     },
     "required": ["replicaCount", "imageTag", "documentation"]
   }
   ```

11. Speichern Sie die Datei. Testen Sie anschließend erneut:

   ```bash
   helm lint ./webapp --strict
   ```

   Helm berücksichtigt die neue Datei `values.schema.json` automatisch. Sie müssen das Schema nicht in der `Chart.yaml` registrieren.

12. Nun wollen wir absichtlich einen ungültigen Wert übergeben. Führen Sie folgenden Befehl aus:

   ```bash
   helm template dev-app ./webapp --set replicaCount=0
   ```

   Was stellen Sie fest? Lesen Sie die Fehlermeldung. Der Wert 0 verletzt die im Schema definierte Mindestanzahl von einem Pod.

13. Probieren Sie einen weiteren Fehler aus:

   ```bash
   helm template dev-app ./webapp --set documentation.environment=staging
   ```

   Warum ist dieser Wert nicht zulässig? Schauen Sie sich dazu den Key `enum` in Ihrer `values.schema.json` an.

14. Testen Sie nun einen ungültigen Datentyp:

   ```bash
   helm template dev-app ./webapp --set-string replicaCount=zwei
   ```

   Überlegen Sie, warum hier `--set-string` verwendet wird. Welchen Unterschied gibt es zum Parameter `--set`?

15. Testen Sie im Anschluss eine gültige Konfiguration:

   ```bash
   helm template dev-app ./webapp --set replicaCount=3 --set documentation.environment=production > /tmp/webapp-valid.yaml
   ```

   Dieses Mal sollte die Schema-Validierung erfolgreich sein. Die generierten Kubernetes-Manifeste liegen in der Datei `/tmp/webapp-valid.yaml`.

16. Wir wollen nun prüfen, was der Begriff „Pflichtfeld“ bei Helm tatsächlich bedeutet. Entfernen Sie testweise die Zeile `owner: "training"` aus der `values.yaml` (nicht aus dem Schema). Führen Sie erneut `helm lint ./webapp` aus. Der Vorgang sollte mit einem Hinweis auf den fehlenden Key `owner` abbrechen.

17. Ergänzen Sie anschließend `owner: "training"` wieder. Wichtig: Ein Key aus dem Abschnitt `required` muss in den **zusammengeführten Values** vorhanden sein. Ist er bereits in der `values.yaml` definiert, muss ihn der Anwender nicht nochmals übergeben.

18. Ein JSON Schema beschreibt nur die Keys, die Sie dort ausdrücklich hinterlegen. Zusätzliche Keys sind standardmäßig erlaubt. Testen Sie dies:

   ```bash
   helm template dev-app ./webapp --set documentation.unbekannt=test > /tmp/webapp-extra.yaml
   ```

   Der zusätzliche Key wird vom Schema nicht zurückgewiesen. Möchten Sie unbekannte Keys unter `documentation` verbieten, ergänzen Sie dort direkt neben `required` den Eintrag:

   ```json
   "additionalProperties": false
   ```

   **Achtung:** Vergessen Sie das Komma nach dem vorhergehenden JSON-Eintrag nicht. Wiederholen Sie den Test. Nun sollte Helm den unbekannten Key ablehnen. Setzen Sie die Änderung anschließend wieder zurück, falls Sie das Schema später erweitern möchten.

19. Neben JSON Schema gibt es in Helm die Template-Funktion `required`. Damit können Sie einen fehlenden Wert direkt im Template abfangen. Betrachten Sie dazu folgendes Beispiel. Es muss nicht in Ihr bestehendes Deployment übernommen werden:

   ```gotemplate
   {{ required "documentation.owner muss angegeben werden" .Values.documentation.owner }}
   ```

   Überlegen Sie, wann eine Prüfung mit `required` sinnvoll ist und wann ein JSON Schema die bessere Wahl darstellt.

20. Abschließend können wir die Standardwerte eines fremden Charts betrachten. Falls in Ihrer Laborumgebung bereits ein Chart-Repository konfiguriert ist, probieren Sie die folgenden Befehle mit einem dort verfügbaren Chart aus:

   ```bash
   helm repo list
   helm show values REPOSITORY/CHART
   helm show readme REPOSITORY/CHART
   helm show chart REPOSITORY/CHART
   ```

   Ersetzen Sie `REPOSITORY/CHART` durch einen tatsächlich vorhandenen Eintrag. Die Befehle setzen ein erreichbares Repository voraus.

21. Überlegen Sie zum Abschluss: Warum ist `helm show values` allein keine Garantie dafür, dass alle zulässigen Konfigurationsmöglichkeiten dokumentiert sind? Welche zusätzliche Rolle spielen die README, das JSON Schema und die Templates?

22. Kontrollieren Sie abschließend noch einmal Ihr Chart:

   ```bash
   helm lint ./webapp --strict
   helm template dev-app ./webapp --namespace dev > /tmp/webapp-final.yaml
   ```

   Wenn beide Befehle erfolgreich ausgeführt werden, ist die lokale Prüfung abgeschlossen. Im nächsten Schritt testen wir die Validierung gegen unser Kubernetes-Cluster.


23. Überprüfen Sie zunächst, ob Ihr Kubernetes-Cluster erreichbar ist. Wir verwenden wie in den bisherigen Übungen den Namespace `dev` und das Release `dev-app`:

   ```bash
   kubectl get nodes
   kubectl get namespace dev
   helm list -n dev
   ```

   **Hinweis:** Falls das Release `dev-app` bereits existiert, wird es im nächsten Schritt aktualisiert. Die bisherigen Übungen haben das Release teilweise wieder deinstalliert. In diesem Fall wird es neu installiert.

24. Installieren beziehungsweise aktualisieren Sie nun unser gültiges Chart. Wir verwenden bewusst `--reset-values`, damit eventuell früher mit `--set` übergebene Werte das Ergebnis nicht beeinflussen:

   ```bash
   helm upgrade --install dev-app ./webapp -n dev --create-namespace --reset-values --set replicaCount=2 --set documentation.environment=development --wait --timeout 3m
   ```

   Kontrollieren Sie anschließend das Deployment:

   ```bash
   kubectl get deployment dev-app-deployment -n dev
   kubectl get pods -n dev
   helm get values dev-app -n dev
   helm history dev-app -n dev
   ```

25. Merken Sie sich die aktuelle Revisionsnummer aus der Helm History und die Anzahl der gewünschten Pods. Nun wollen wir absichtlich einen ungültigen Wert beim Upgrade übergeben:

   ```bash
   helm upgrade dev-app ./webapp -n dev --reset-values --set replicaCount=0
   ```

   Der Befehl sollte durch die Schema-Validierung abbrechen. Das Kubernetes-Cluster erhält keine neuen Manifeste.

26. Überprüfen Sie, ob das fehlgeschlagene Upgrade eine neue Release-Revision angelegt oder die Pod-Anzahl verändert hat:

   ```bash
   helm history dev-app -n dev
   kubectl get deployment dev-app-deployment -n dev
   helm get values dev-app -n dev
   ```

   Vergleichen Sie die Ergebnisse mit Schritt 24. Bei einem durch das JSON Schema zurückgewiesenen Wert bleibt das bisherige Release erhalten.

27. Probieren Sie nun einen weiteren ungültigen Wert aus:

   ```bash
   helm upgrade dev-app ./webapp -n dev --reset-values --set documentation.environment=staging
   ```

   Überlegen Sie, warum dieser Upgrade-Versuch ebenfalls abgebrochen wird. Wird `staging` durch Kubernetes oder bereits durch Helm zurückgewiesen?

28. Führen Sie jetzt ein gültiges Upgrade durch und erhöhen Sie die Anzahl der Pods auf drei:

   ```bash
   helm upgrade dev-app ./webapp -n dev --reset-values --set replicaCount=3 --set documentation.environment=production --wait --timeout 3m
   ```

   Kontrollieren Sie anschließend die Änderungen:

   ```bash
   kubectl get deployment dev-app-deployment -n dev
   kubectl get pods -n dev
   helm get values dev-app -n dev
   helm history dev-app -n dev
   ```

   Sie sollten nun drei gewünschte Replikate und eine zusätzliche Helm-Revision sehen. Der Key `documentation.environment` wird zwar durch Helm validiert und im Release gespeichert, verändert aber noch keine Kubernetes-Ressource, solange er nicht in einem Template verwendet wird.

29. Überlegen Sie zum Abschluss, was der Unterschied zwischen einer **erfolgreichen Schema-Validierung** und einem **erfolgreichen Kubernetes-Deployment** ist. Kann ein Chart trotz gültiger Values fehlschlagen, beispielsweise weil ein Container-Image nicht heruntergeladen werden kann?

30. Für die nachfolgenden Kursübungen können Sie das Release installiert lassen. Falls Sie es entfernen möchten, nutzen Sie den bereits bekannten Befehl:

   ```bash
   helm uninstall dev-app -n dev
   ```

---

## Zusammenfassung

Sie haben Ihr bestehendes Helm-Chart um eine `README.md` und eine `values.schema.json` ergänzt. Während die README den Administrator über Konfigurationsmöglichkeiten informiert, weist Helm anhand des JSON Schemas ungültige Werte bereits vor einer Änderung im Kubernetes-Cluster zurück. Sie haben anschließend ein gültiges Release installiert, fehlgeschlagene Upgrade-Versuche untersucht und eine gültige Konfigurationsänderung erfolgreich ausgerollt.

**Weiterführende Dokumentation:**

- Helm Charts: https://helm.sh/docs/topics/charts/
- Helm Values Best Practices: https://helm.sh/docs/chart_best_practices/values/
- Helm Template Functions: https://helm.sh/docs/chart_template_guide/function_list/
- helm-docs: https://github.com/norwoodj/helm-docs
