#  Projet Week 1 — Agent IA de Gestion des Tâches

Un agent basé sur Langflow qui capture des messages en langage naturel, les classifie, les enregistre dans Notion et MongoDB, et répond poliment aux entrées hors-sujet.

---

##  Description

Cet agent reçoit un message de l'utilisateur, détermine s'il contient une tâche actionnable (rappel, rendez-vous, deadline, etc.) ou s'il est hors-sujet, et redirige le message en conséquence :

- **Route tâche** : extrait les données structurées (tâche, date, priorité), les sauvegarde dans Notion (base de données des tâches), les persiste dans MongoDB, exécute un code Python de post-traitement (mémoire SQLite + alerte Slack optionnelle), et retourne une confirmation.
- **Route erreur** : formate le message comme une entrée d'erreur dans une base Notion dédiée et informe l'utilisateur que son message n'est pas lié à la gestion de tâches.
- **Route Else** : capture tout ce qui n'a pas matché et retourne une réponse d'erreur générique via un LLM Groq.

---

##  Architecture

### Résumé des nœuds

| Composant | ID | Rôle |
|---|---|---|
| Chat Input | `ChatInput` | Point d'entrée — reçoit le message utilisateur |
| Groq (LLM Routeur) | `GroqModel` | Fournit le LanguageModel au Smart Router |
| Smart Router | `SmartRouter` | Route vers `tasks`, `erreur`, ou `Else` |
| Groq (Extracteur de tâche) | `GroqModel` | Extrait tâche / date / priorité en JSON |
| Type Converter (→ Data) | `TypeConverterComponent` | Convertit le message LLM en Data |
| Parser | `ParserComponent` | Extrait le champ `{text}` depuis le Data |
| Prompt Template | `Prompt Template` | Construit le template de code Python avec `{reponse_llm}` |
| Python Interpreter | `PythonREPLComponent` | Exécute : parsing JSON → insertion SQLite → alerte Slack → print réponse |
| Chat Output (tâche) | `ChatOutput` | Retourne le résultat Python à l'utilisateur |
| Groq (Formateur Notion tâche) | `GroqModel` | Convertit le message en JSON de propriétés Notion |
| Notion Create Page (tâches) | `NotionPageCreator` | Crée une page dans la DB Notion des tâches |
| Type Converter (→ Message) | `TypeConverterComponent` | Convertit la réponse Notion en Message |
| Groq (Réponse erreur) | `GroqModel` | Retourne "message non lié à la gestion de tâches" |
| Chat Output (erreur) | `ChatOutput` | Retourne le message d'erreur à l'utilisateur |
| Groq (Formateur Notion erreur) | `GroqModel` | Convertit l'erreur en JSON de propriétés Notion |
| Notion Create Page (erreurs) | `NotionPageCreator` | Crée une page dans la DB Notion des erreurs |
| Type Converter (erreur) | `TypeConverterComponent` | Convertit la réponse Notion erreur en Message |

---

##  Installation & Configuration

### Prérequis

- [Langflow](https://github.com/langflow-ai/langflow) `>= 1.7.0`
- Une clé API [Groq](https://console.groq.com) (modèles : `llama-3.1-8b-instant` ou `llama-3.3-70b-versatile`)
- Un token d'intégration [Notion](https://notion.so) et la db:
  - **DB Erreurs** (`3659e73c-1985-8078-a19a-fa7930762727`) — propriété : `Erreur` (title)
- Un URI MongoDB Atlas (configuré dans le code du `Prompt Template`)

### Étapes d'installation

1. Importer `Projet_week_1.json` dans votre instance Langflow via **Importer un Flow**.
2. Renseigner la **clé API Groq** sur les quatre nœuds Groq.
3. Renseigner le **Notion Secret** (token d'intégration) sur les deux nœuds `NotionPageCreator`.
4. Dans le nœud `Prompt Template`, mettre à jour l'URI MongoDB dans le code du template :
   ```python
   uri = "mongodb+srv://<utilisateur>:<mot_de_passe>@<cluster>.mongodb.net/?appName=<AppName>"
   ```
5. Lancer le flow depuis le Playground Langflow.

---

##  Tests Unitaires

### Test 1 — Entrée valide : tâche

**Entrée :** `"Rappelle-moi d'envoyer le rapport trimestriel demain à 9h"`

**Comportement attendu :**
- Le Smart Router route vers `tasks`
- Le Groq extracteur retourne `{ "task": "Envoyer le rapport trimestriel", "date": "demain à 9h", "priority": "medium" }`
- Enregistrement inséré dans MongoDB 
- L'utilisateur reçoit un message de confirmation

### Test 2 — Entrée invalide / hors-sujet

**Entrée :** `"asdfgh"`

**Comportement attendu :**
- Le Smart Router route vers `erreur`
- Page d'erreur créée dans la DB Notion Erreurs
- L'utilisateur reçoit : `"The message you entered is not related to task management"`

### Test 3 — Tâche urgente

**Entrée :** `"URGENT : déployer le hotfix avant 17h aujourd'hui"`

**Comportement attendu :**
- Le Smart Router route vers `tasks`
- Le Groq extracteur définit `priority: "high"` et `date: "aujourd'hui à 17h"`
- Pipeline complet s'exécute normalement


---

##  Justification des Choix Techniques

| Choix | Justification |
|---|---|
| **Groq (llama-3.1-8b-instant)** | Inférence rapide et économique pour l'extraction structurée ; temperature=0.1 pour une sortie JSON déterministe |
| **Smart Router** | Le routage conditionnel basé sur LLM évite une correspondance fragile par mots-clés ; deux routes nommées (`tasks`, `erreur`) + fallback Else couvrent tous les cas |
| **Notion comme stockage principal** | Tableau de bord de tâches lisible par l'humain ; intégration API Notion native dans Langflow |
| **MongoDB comme stockage secondaire** | Stockage de documents persistant pour l'analytique et les futures requêtes |
| **SQLite dans le Python Interpreter** | Mémoire locale légère ; aucune dépendance externe requise à l'exécution |
| **Parser + Prompt Template** | Découple le parsing de la sortie LLM de l'exécution du code ; rend le pipeline plus maintenable |
| **DB Notion séparée pour les erreurs** | Garde les données de tâches propres ; erreurs journalisées séparément pour révision |

---

##  Structure du Projet

```
Projet_week_1.json       # Définition du flow Langflow (à importer)
README.md                # Ce fichier
DIAGRAM.md               # Diagramme d'architecture (Mermaid)
RUNBOOK.md               # Runbook opérationnel
```
