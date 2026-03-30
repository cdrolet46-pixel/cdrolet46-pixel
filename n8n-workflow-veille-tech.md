# Creer un workflow de veille technologique dans n8n

## C'est quoi n8n ?

n8n c'est comme des LEGO pour automatiser des taches sur internet. Tu connectes des blocs ensemble et chaque bloc fait quelque chose (lire un site, envoyer un message, appeler une IA, etc.). Pas besoin de coder — sauf si tu veux faire des trucs avances.

---

## Ce que fait ce workflow

Chaque matin a 8h :
1. Il va lire les derniers articles de 3 sites tech/secu
2. Il envoie les titres a une IA (Groq/Llama)
3. L'IA fait un resume groupe par theme en francais
4. Le resume est envoye automatiquement dans Discord

---

## Les blocs utilises (noeuds)

```
Schedule → RSS x3 → Merge → Limit → Aggregate → Code → HTTP Groq → Code → HTTP Discord
```

| Noeud | Type | Role |
|-------|------|------|
| Schedule 8h | Trigger | Demarre le workflow a 8h chaque matin |
| RSS Hacker News | RSS Feed | Lit les articles de news.ycombinator.com |
| RSS Krebs | RSS Feed | Lit les articles de krebsonsecurity.com |
| RSS THN | RSS Feed | Lit les articles de The Hacker News |
| Merge | Merge | Combine les 3 listes en une seule |
| Limit 15 | Limit | Garde seulement les 15 premiers articles |
| Aggregate | Aggregate | Regroupe tout dans un seul objet |
| Prep Groq | Code (JS) | Prepare le body JSON pour l'API Groq |
| Groq HTTP | HTTP Request | Envoie les articles a l'IA Groq |
| Prep Discord | Code (JS) | Formate le message pour Discord |
| Discord HTTP | HTTP Request | Envoie le digest dans Discord |

---

## Pourquoi des noeuds Code ?

Au lieu d'utiliser les expressions n8n `{{ }}` qui peuvent causer des erreurs de syntaxe, on utilise des noeuds **Code** en JavaScript pur. C'est plus stable et plus facile a debugger.

---

## Le code du noeud Prep Groq

```javascript
const articles = items[0].json.articles.map(a => ({ t: a.title, l: a.link }));
const body = {
  model: 'llama-3.3-70b-versatile',
  messages: [
    {
      role: 'system',
      content: 'Tu es un assistant de veille technologique. Resume en francais, groupe par theme: Cybersecurite, IA, DevOps, Tech General. Pour chaque article: emoji + titre + 1 phrase + lien. Max 1800 caracteres.'
    },
    {
      role: 'user',
      content: 'Articles: ' + JSON.stringify(articles)
    }
  ]
};
return [{ json: { body: JSON.stringify(body) } }];
```

---

## Le code du noeud Prep Discord

```javascript
const response = items[0].json;
const content = response.choices[0].message.content;
const today = new Date().toLocaleDateString("fr-CA");
const message = "## Veille Tech - " + today + "\n\n" + content;
return [{ json: { payload: JSON.stringify({ username: "Veille Tech", content: message.substring(0, 2000) }) } }];
```

---

## Configuration des noeuds HTTP

### Groq HTTP
- Method : `POST`
- URL : `https://api.groq.com/openai/v1/chat/completions`
- Headers : `Authorization: Bearer <GROQ_API_KEY>`
- Body : `={{ $json.body }}` (raw JSON)

### Discord HTTP
- Method : `POST`
- URL : webhook Discord
- Body : `={{ $json.payload }}` (raw JSON)

---

## Conseil important

> Utilise toujours des noeuds **Code** pour preparer les payloads complexes avant les noeuds HTTP. Les expressions `{{ }}` de n8n peuvent mal interpreter les guillemets et les caracteres speciaux dans les JSON imbriques.

---

## Prerequis

- Un compte [Groq](https://console.groq.com) (gratuit) pour la cle API
- Un webhook Discord (Parametres du salon → Integrations → Webhooks)
- n8n auto-heberge ou cloud

---

## Pour adapter ce workflow

- **Ajouter une source RSS** : dupliquer un noeud RSS et le connecter au Merge
- **Changer la langue** : modifier le prompt dans Prep Groq
- **Changer l'heure** : modifier le cron dans Schedule (format : `0 8 * * *` = 8h chaque jour)
- **Envoyer sur Slack** : remplacer Discord HTTP par un noeud Slack
