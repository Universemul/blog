+++
date = '2026-06-21'
title = "Dans les coulisses du crawler TMDB de Welshie"
summary = "Exploration technique du crawler asynchrone développé en Python/Django pour importer efficacement des centaines de milliers de films et séries depuis TMDB."
tags = ["python", "django", "crawler", "asyncio", "tmdb"]
+++

Alimenter une base de données de suivi de médias nécessite de récupérer un catalogue gigantesque. **The Movie Database (TMDB)** est la référence, mais importer plus de 900 000 films et séries via des requêtes HTTP classiques peut prendre plusieurs jours. 

Pour Welshie, j'ai développé un crawler asynchrone en Python exploitant `asyncio` pour réaliser cette tâche en quelques heures seulement.

Voici comment il fonctionne sous le capot.

---

## Le Défi : Le Volume de Données

Chaque jour, TMDB génère des fichiers d'export contenant les identifiants actifs de leur base de données. 
* Un fichier d'export de films pèse environ 25 Mo compressé et contient des centaines de milliers de lignes au format JSON.
* Faire 900 000 requêtes HTTP de manière séquentielle (l'une après l'autre) à raison de 50ms par requête prendrait environ **12,5 heures**.
* En cas de latence réseau, ce temps peut doubler ou tripler.

---

## L'Approche en 3 Étapes de Welshie

Le crawler de Welshie (implémenté dans `MediaCrawler`) résout ce problème en combinant téléchargement de fichiers plats, filtrage en mémoire et requêtes asynchrones groupées.

### 1. Extraction des Identifiants Quotidiens
Plutôt que de parcourir l'API de TMDB à l'aveugle, nous téléchargeons directement leur export quotidien compressé `.json.gz` :

```python
BASE_MOVIE_EXPORT_URL = "https://files.tmdb.org/p/exports/movie_ids_{formatted_date}.json.gz"
```

Le script télécharge le fichier GZIP, le décompresse à la volée en mémoire, et extrait tous les identifiants uniques :

```python
def __download_file(self, base_url, specific_date=None, existing_ids: set = None):
    today = (specific_date or datetime.today()).strftime("%m_%d_%Y")
    url = base_url.format(formatted_date=today)
    
    with requests.session() as session:
        with session.request("GET", url) as response:
            response.raise_for_status()
            data = gzip.decompress(response.content.strip()).split(b"\n")
            
    ids = self.__get_ids(data, existing_ids)
    return ids
```

### 2. Le Filtrage Intelligent
Pour éviter de requêter inutilement des films déjà présents en base de données, la fonction `__get_ids` accepte un ensemble (`set`) d'identifiants déjà indexés dans PostgreSQL. La recherche dans un `set` en Python se fait en complexité temps $O(1)$, ce qui permet un filtrage instantané.

### 3. Les Requêtes Asynchrones Par Lots (Batches)
C'est ici que la magie de l'asynchronisme opère. Le goulot d'étranglement étant l'attente des réponses réseau (I/O Bound), nous utilisons `asyncio.gather` pour envoyer plusieurs requêtes HTTP simultanément.

Pour ne pas saturer la mémoire ou être bloqué par les pare-feu de TMDB, les requêtes sont découpées en paquets (ou *chunks*) de taille fixe (par défaut 40 requêtes simultanées) :

```python
async def _fetch_medias(
    func_detail,
    media_ids,
    offset,
    limit,
    append_to_response: list[AppendToResponse] = None,
):
    result = 0
    # Découpage des identifiants en sous-listes (chunks) de taille 'offset'
    for _chunck_media_ids in tqdm(divide_chunks(media_ids, offset)):
        if limit is not None and limit < result:
            break
        tasks = []
        # Création d'une coroutine pour chaque média du chunk
        [
            tasks.append(func_detail(media_id, append_to_response))
            for media_id in _chunck_media_ids
        ]
        result += len(tasks)
        
        # Exécution simultanée de toutes les requêtes du lot
        r = await asyncio.gather(*tasks, return_exceptions=True)
        yield r
```

Chaque lot (yield) est ensuite traité par le service Django pour être persisté dans la base PostgreSQL via des écritures en masse (*bulk inserts*).

---

## Lancer l'acquisition

Toute cette logique est encapsulée dans des commandes de gestion personnalisées de Django (`manage.py`). Pour lancer l'importation de manière sécurisée en limitant le nombre d'entrées lors des tests :

```bash
# Se connecter à l'environnement virtuel et lancer la migration
./manage.py migrate

# Récupérer les genres et plateformes
./manage.py fetch_genres
./manage.py fetch_watch_providers

# Lancer le crawler sur les 500 premiers films du jour
./manage.py fetch_daily_movies --limit 500
```

## Bilan

Grâce à cette architecture asynchrone écrite en Python 3.13, Welshie est capable d'ingérer et de maintenir à jour des centaines de milliers de fiches de films et séries avec une vitesse de traitement optimale et une consommation de ressources maîtrisée.
