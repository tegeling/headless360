# Slack-Nachricht (intern, Deutsch)

_Zum Kopieren in Slack. Emojis/Formatierung sind Slack-tauglich._

---

:mag: *Claude + Salesforce: Default Tools vs. Standard-MCP vs. Custom-MCP — meine Erkenntnisse*

Hey Team :wave: — ich habe denselben 15-Prompt-Sales-Workflow gegen eine echte Salesforce-Org auf *drei* Arten laufen lassen und die Ansätze verglichen. Kurzfazit:

*TL;DR:* Für Analyse liefert ein *Standard-Salesforce-MCP-Server* dieselben Antworten wie die Default-Tools – aber mit deutlich weniger Reibung (Claude als schnellerer Analyst). Der echte Sprung ist ein *Custom-MCP-Server*: Er macht org-spezifische Logik („Account-Health prüfen“, „Sync-Meeting anlegen“) zu *einem einzigen, governten Tool-Call* (Claude als Operator).

*Die drei Ansätze:*
• *Default Tools* – Claude steuert die `sf` CLI (SOQL, Apex) über Bash → maximale Kontrolle, maximale Handarbeit
• *Standard-MCP* – von Salesforce gehosteter Server (Schema, SOQL, Aggregationen) → gleiche Analyse, viel weniger Reibung
• *Custom-MCP* – selbst gebauter Server um unsere eigene Apex-Logik (Account Health, Sync Meeting) → die Urteilskraft unserer Organisation, nur einen Call entfernt

*Was rauskam:*
• *Analyse (Prompts 1–13):* Standard-MCP und Default-Tools kommen zu praktisch denselben Ergebnissen – beide brauchen Claude für jede fachliche Definition („at-risk“, „underworked“, „Käufer“). Aber MCP spart die ganze Mechanik: serverseitige Aggregation & Relationship-Queries statt handgeschriebenem SOQL, JSON-Fallen und Python-Gebastel. In einem Fall sogar *genauer* (alle 152 Accounts via `GROUP BY` – die Default-Variante hielt „LIMIT 30“ fälschlich für die ganze Org).
• *Aktionen (Prompts 14–15):* Hier glänzt der *Custom-MCP*. Account-Health per Default hätte tiefes Apex-Wissen oder ein selbst erdachtes Bewertungsraster gebraucht – der Custom-Server macht es in *einem Call → 56,67/100 inkl. Aufschlüsselung*. Sync-Meeting: statt mehrstufigem SOQL + handgebautem Record *ein Call*, der Kontakt automatisch wählt, offene Cases einbettet und sinnvolle Defaults setzt.

*Der ehrliche Haken (wichtig!):* Das Custom-Health-Tool gab `Activity 0/30` zurück, weil die zugrunde liegende Apex `Date.today()` nutzt, die Demo-Daten aber in der Zukunft liegen – ein eigentlich aktiver Account wurde als inaktiv bewertet. Die „mühsame“ manuelle Variante *hat den Fehler bemerkt*, das bequeme Tool *hat ihn versteckt*. :warning: Lehre: Ein Custom-Tool ist nur so gut wie seine interne Logik – es muss datumsbewusst/parametrierbar sein und seine Teil-Scores offenlegen, damit die Blackbox prüfbar bleibt.

*Empfehlung / Wann was:*
• *Default Tools* → Ad-hoc-Exploration, volle Transparenz, Debugging der Daten selbst
• *Standard-MCP* → der Alltags-Default für NL-gestützte Analyse (sobald Auth steht). Erfindet aber keine Definitionen und zeichnet keine Charts.
• *Custom-MCP* → bauen für org-spezifische Logik, die *konsistent* sein soll (Scoring, Eligibility, Health, Pricing) und für wiederkehrende strukturierte Aktionen. Zuerst die wertvollste „rechnet jeder anders aus“-Kennzahl und den häufigsten strukturierten Write kapseln.

:bulb: *Merksatz:* Standard-MCP macht Claude zum schnelleren Analysten. Ein Custom-MCP lässt Claude mit der Urteilskraft unserer Organisation handeln.

*Kontext/Caveats:* Läuft auf synthetischen Demo-Daten (Security-Vendor, US-West) – der *Ansatz-Vergleich* ist die Botschaft, nicht die konkreten Zahlen. Beide Schreibvorgänge waren user-bestätigt.

Details, per-Prompt-Scorecard und die Workshop-Story liegen in den Docs – meldet euch, wenn ihr reinschauen wollt. :salesforce: :robot_face:
