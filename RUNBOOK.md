#  Runbook — Agent IA de Gestion des Tâches (Projet Week 1)

> Guide opérationnel : configuration des nœuds, cas de réponse de l'agent, et gestion des erreurs HTTP.

---

##  Configuration des nœuds — Clés API

### Nœuds Groq (× 5)

Chaque nœud Groq doit avoir sa clé API renseignée dans le champ `api_key`.

| Nœud ID | Rôle | Modèle | Base URL |
|---|---|---|---|
| `GroqModel` | Routeur d'intention (SmartRouter) | `llama-3.1-8b-instant` | `https://api.groq.com` |
| `GroqModel` | Extracteur de tâche (JSON) | `llama-3.1-8b-instant` | `https://api.groq.com` |
| `GroqModel` | Formateur Notion — tâches | `llama-3.1-8b-instant` | `https://api.groq.com` |
| `GroqModel` | Formateur Notion — erreurs | `llama-3.1-8b-instant` | `https://api.groq.com` |
| `GroqModel` | Réponse message hors-sujet | `llama-3.1-8b-instant` | `https://api.groq.com` |

**Champ à renseigner dans chaque nœud :**
```
api_key: <VOTRE_CLE_API_GROQ>
```

---

### Nœuds Notion (× 2)

| Nœud ID | Rôle | Database ID |
|---|---|---|
| `NotionPageCreator` | Crée une page dans la DB des tâches | `3689e73c198580b08152cfe9ba9e9d20` |
| `NotionPageCreator` | Crée une page dans la DB des erreurs | `3659e73c-1985-8078-a19a-fa7930762727` |

**Champs à renseigner dans chaque nœud :**
```
notion_secret: <VOTRE_TOKEN_INTEGRATION_NOTION>
database_id:   <DATABASE_ID_CI_DESSUS>
```

---

### MongoDB — Prompt Template (`Prompt Template-rd6zu`)

La connexion MongoDB est définie directement dans le code Python du template :

```python
uri = "mongodb+srv://user:password@cluster0.khmxz7l.mongodb.net/?appName=Cluster0"
```

**Remplacer** `user:password` par vos identifiants réels et `cluster0.khmxz7l` par votre cluster Atlas.

- Base de données cible : `AgentMemory`
- Collection cible : `tasks`

---

## 👤 User Stories & Cas de réponse de l'agent

---

###  US-01 — L'utilisateur envoie une tâche valide

**En tant qu'utilisateur**, je veux envoyer un rappel en langage naturel et recevoir une confirmation que la tâche a été enregistrée.

**Message envoyé :**
> `"Rappelle-moi d'appeler le client Dupont demain à 14h"`

**Comportement de l'agent :**
1. `SmartRouter` route vers **`tasks`**
2. `GroqModel` extrait :
   ```json
   { "task": "Appeler le client Dupont", "date": "demain à 14h", "priority": "medium" }
   ```
3. `GroqModel` formate les propriétés Notion
4. `NotionPageCreator` crée une page dans la DB Tâches
5. Le code Python insère l'entrée dans MongoDB (`AgentMemory.tasks`) et SQLite
6. `ChatOutput` retourne la confirmation à l'utilisateur

**Réponse attendue du chat :**
> `"Tâche enregistrée avec succès"`

---

###  US-02 — L'utilisateur envoie une tâche urgente

**En tant qu'utilisateur**, je veux signaler une urgence et que la priorité soit détectée automatiquement.

**Message envoyé :**
> `"URGENT : déployer le hotfix en production avant 17h aujourd'hui"`

**Comportement de l'agent :**
1. `SmartRouter` route vers **`tasks`**
2. `GroqModel` extrait :
   ```json
   { "task": "Déployer le hotfix en production", "date": "aujourd'hui avant 17h", "priority": "high" }
   ```
3. Pipeline complet s'exécute (Notion + MongoDB)

**Réponse attendue du chat :**
> `"Tâche enregistrée avec succès"`

---


###  US-03 — L'utilisateur envoie un message hors-sujet

**En tant qu'utilisateur**, j'envoie un message sans rapport avec la gestion de tâches.

**Message envoyé :**
> `"Quelle est la capitale de la France ?"`

**Comportement de l'agent :**
1. `SmartRouter` route vers **`erreur`**
2. `GroqModel` formate un JSON d'erreur pour Notion
3. `NotionPageCreator` crée une page dans la DB Erreurs
4. `GroqModel` génère la réponse d'erreur

**Réponse attendue du chat :**
> `"the message you entered is not related to task management"`

---

###  US-04 — L'utilisateur envoie un message incompréhensible

**En tant qu'utilisateur**, j'envoie du texte aléatoire.

**Message envoyé :**
> `"azerty123 !!!"`

**Comportement de l'agent :**
1. `SmartRouter` route vers **`erreur`**
2. Même pipeline que US-03

**Réponse attendue du chat :**
> `"the message you entered is not related to task management"`

---

##  Gestion des erreurs HTTP

---

### Groq — `401 Unauthorized`

**Cause :** Clé API Groq absente, invalide ou révoquée.

**Nœuds concernés :** `GroqModel`, `GroqModel`, `GroqModel`, `GroqModel`, `GroqModel`

**Résolution :**
1. Aller sur [console.groq.com](https://console.groq.com) → **API Keys** → générer une nouvelle clé.
2. Dans Langflow, ouvrir chaque nœud Groq et renseigner le champ `api_key`.
3. Sauvegarder le flow et relancer un test.

---

### Groq — `400 Bad Request`

**Cause :** Paramètre invalide envoyé à l'API Groq (prompt vide, `max_tokens` hors limites, modèle inexistant).

**Résolution :**
1. Vérifier que le champ `model_name` est bien `llama-3.1-8b-instant` (ou un modèle actif sur votre compte Groq).
2. S'assurer que le message entrant dans le nœud n'est pas vide.
3. Vérifier que `max_tokens` est vide ou dans les limites du modèle (≤ 8192 pour `llama-3.1-8b-instant`).

---

### Notion — `401 Unauthorized`

**Cause :** Token d'intégration Notion absent ou invalide.

**Nœuds concernés :** `NotionPageCreator`, `NotionPageCreator`

**Résolution :**
1. Aller sur [notion.so/my-integrations](https://www.notion.so/my-integrations).
2. Ouvrir l'intégration concernée → copier le **Internal Integration Token**.
3. Dans Langflow, renseigner le champ `notion_secret` sur les deux nœuds `NotionPageCreator`.

---

### Notion — `400 Bad Request`

**Cause :** JSON de propriétés malformé envoyé à l'API Notion, ou propriété inexistante dans la base de données.

**Nœuds concernés :** `NotionPageCreator`, `NotionPageCreator`

**Résolution :**
1. Inspecter la sortie brute du nœud Groq formateur (`GroqModel` ou `GroqModel`).
2. Vérifier que le JSON retourné correspond exactement aux propriétés de la DB Notion :
   - DB Tâches : `Nom` (title), `Description` (rich_text), `Etat` (select)
   - DB Erreurs : `Erreur` (title)
3. S'assurer que les noms de propriétés dans Notion n'ont pas été renommés.
4. Vérifier que l'intégration Notion a bien accès aux deux bases de données (dans Notion : **...** → **Connexions** → ajouter l'intégration).

---

### MongoDB — `401 Authentication Failed`

**Cause :** Identifiants MongoDB incorrects ou accès réseau refusé.

**Nœud concerné :** `Prompt Template` (code Python)

**Résolution :**
1. Aller sur [MongoDB Atlas](https://cloud.mongodb.com) → **Database Access** → vérifier le nom d'utilisateur et le mot de passe.
2. Aller sur **Network Access** → ajouter l'IP du serveur Langflow (ou `0.0.0.0/0` pour les tests).
3. Mettre à jour l'URI dans le code Python du `Prompt Template` :
   ```python
   uri = "mongodb+srv://<utilisateur>:<mot_de_passe>@<cluster>.mongodb.net/?appName=<AppName>"
   ```

---

### MongoDB — `400 Bad Request` (données invalides)

**Cause :** Le document inséré dans MongoDB contient des champs non conformes ou le JSON extrait par Groq est malformé.

**Résolution :**
1. Vérifier la sortie du `GroqModel` : le JSON doit contenir `task`, `date`, `priority`.
2. Dans le code Python du `Prompt Template`, le bloc `try/except` capture les erreurs de parsing :
   ```python
   except Exception as e:
       print("Erreur JSON :", str(e))
       data = {}
   ```
3. Si `data` est vide, MongoDB reçoit un document vide `{}` — vérifier le prompt du `GroqModel` pour forcer une sortie JSON valide.
