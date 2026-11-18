# Guide local — tester et spécialiser le modèle sur son poste

Version testée le 14 novembre 2026 sur Windows 11, avec une carte NVIDIA RTX A5500 (16 Go).

Ce guide se fait entièrement sur votre ordinateur, sans AWS. Il a deux parties :

- **Partie A — Faire tourner l'application en local** (45 min, dont 20 à 30 min de téléchargements). Vous obtenez sur `http://localhost` la même page de discussion que sur AWS.
- **Partie B — Spécialiser le modèle par fine-tuning LoRA** (1 h 30, dont 30 à 60 min de téléchargements et d'attente). Vous transformez Llama 3.2 3B en assistant de Polytechnique Sousse, vous mesurez le gain, et vous discutez avec lui dans l'interface web.

Toutes les commandes se tapent dans **PowerShell**, depuis le dossier du projet, sauf mention contraire. Chaque étape donne la commande, puis *le résultat attendu en italique*.

---

## 1. Prérequis

### 1.1 Matériel

| Élément | Partie A | Partie B |
| --- | --- | --- |
| Système | Windows 10 ou 11, 64 bits | Idem |
| Mémoire vive | 8 Go au minimum, 16 Go conseillés | 16 Go |
| Espace disque libre | 12 Go | 40 Go |
| Carte graphique | Aucune | NVIDIA avec 12 Go de mémoire vidéo ou plus |
| Connexion | Environ 8 Go à télécharger | Environ 14 Go de plus |

Mémoire vidéo réellement utilisée par l'entraînement, mesurée pendant le test :

| Taille de lot (`--lot`) | Mémoire vidéo utilisée | Carte minimale |
| --- | --- | --- |
| 4 (valeur par défaut) | 11,9 Go | 16 Go |
| 2 | 9,1 Go | 12 Go |
| 1 | 7,8 Go | 10 Go |

**Vous n'avez pas de carte NVIDIA suffisante ?** Faites la partie A, puis la partie B à partir de l'étape B7, avec le fichier `eps-assistant-q4_k_m.gguf` fourni par l'enseignant : vous sautez l'entraînement, mais vous déployez, évaluez et testez le modèle spécialisé.

### 1.2 Logiciels à installer

| Logiciel | Version testée | Où le trouver |
| --- | --- | --- |
| Docker Desktop | 29.9 | docker.com/products/docker-desktop |
| Git | 2.54 | git-scm.com |
| Python | 3.12 (3.10 à 3.12 conviennent) | python.org, en cochant **Add python.exe to PATH** |
| Pilote NVIDIA (partie B) | récent | nvidia.com/drivers |

Aucun compte n'est nécessaire, ni Hugging Face ni AWS.

### 1.3 Vérifier son poste

1. Démarrez **Docker Desktop** et attendez que l'icône de la baleine soit fixe.
2. Ouvrez **PowerShell** et tapez :

```powershell
docker --version
docker compose version
git --version
python --version
```

*Attendu : quatre numéros de version, sans message d'erreur. `python --version` doit afficher 3.10, 3.11 ou 3.12.*

3. Pour la partie B seulement :

```powershell
nvidia-smi
```

*Attendu : un tableau avec le nom de votre carte et sa mémoire, par exemple `16384MiB`. Si la commande est inconnue, installez le pilote NVIDIA.*

Si vous venez d'installer un logiciel et que PowerShell ne le trouve pas, fermez la fenêtre et rouvrez-en une.

---

## Partie A — Faire tourner l'application en local

### A1 — Récupérer le code

1. Placez-vous dans le dossier où vous rangez vos projets, puis clonez votre dépôt. Remplacez `COMPTE`.

```powershell
git clone https://github.com/COMPTE/llm-lab.git
cd llm-lab
dir
```

*Attendu : les dossiers `app`, `finetune`, `infra`, `pipeline`, `tests` et les fichiers `docker-compose.yml`, `docker-compose.local.yml`, `.env.exemple`.*

Toutes les commandes suivantes se tapent depuis ce dossier `llm-lab`.

### A2 — Configurer

1. Copiez le fichier de configuration d'exemple.

```powershell
copy .env.exemple .env
type .env
```

*Attendu :*

```
MODEL=llama3.2:3b
ACCESS_CODE=test-123
APP_IMAGE=llm-app:local
```

Le code d'accès de l'interface est donc `test-123`.

### A3 — Construire l'image et démarrer

1. Construisez l'image de l'API. *Attendu : la dernière ligne contient `naming to docker.io/library/llm-app:local`.*

```powershell
docker build -t llm-app:local app
```

2. Démarrez les deux conteneurs. Le complément `docker-compose.local.yml` rend le serveur de modèle joignable depuis votre poste, ce qui servira en partie B.

```powershell
docker compose -f docker-compose.yml -f docker-compose.local.yml up -d
```

*Attendu : Docker télécharge l'image `ollama/ollama` (environ 5 Go, le plus long du guide), puis affiche `Container llm-lab-ollama-1 Started` et `Container llm-lab-app-1 Started`.*

3. Téléchargez le modèle (environ 2 Go). *Attendu : une barre de progression, puis `success`.*

```powershell
docker compose exec ollama ollama pull llama3.2:3b
```

4. Vérifiez l'API.

```powershell
curl.exe -s http://localhost/api/health
```

*Attendu :*

```json
{"ok":true,"version":"dev","modele":"llama3.2:3b","modele_pret":true}
```

### A4 — Tester dans le navigateur

1. Ouvrez `http://localhost`. *Attendu : la page « LLM Lab », avec l'en-tête « version dev · llama3.2:3b · modèle prêt ».*
2. Saisissez le code `test-123` en haut à droite, posez une question, envoyez. *Attendu : la réponse s'écrit progressivement, puis la durée et le débit s'affichent.*
3. Videz le champ du code et renvoyez une question. *Attendu : « Échec : code d'accès invalide ».*
4. Posez cette question et notez la réponse, vous la comparerez en partie B : « Quelle est l'adresse de l'École Polytechnique de Sousse ? » *Attendu : une adresse inventée. Pendant le test, le modèle a répondu « Rue de la République, 8018 Sousse », qui est fausse.*

### A5 — Lancer les tests de l'API

Les tests n'ont besoin ni de Docker ni du modèle : ils utilisent un faux serveur.

```powershell
python -m venv .venv-tests
.venv-tests\Scripts\python -m pip install -q -r app\requirements.txt pytest
.venv-tests\Scripts\python -m pytest -q
```

*Attendu : `7 passed`, avec un avertissement sans conséquence.*

### A6 — Regarder ce qui tourne

```powershell
docker compose ps
docker compose logs app --tail 5
docker compose exec ollama ollama ps
```

*Attendu : deux services à l'état `running` ; une ligne JSON par question posée ; le modèle chargé, avec sa taille en mémoire (2,6 Go).*

### Point de contrôle de la partie A

- [ ] `http://localhost` répond à une question avec le code `test-123`.
- [ ] Une requête sans code est refusée.
- [ ] Les 7 tests passent.
- [ ] Vous avez noté la réponse inventée sur l'adresse de l'école.

Laissez les conteneurs tourner pour la partie B. Pour tout arrêter plus tard sans rien perdre : `docker compose stop`. Pour redémarrer : la commande de l'étape A3-2.

---

## Partie B — Spécialiser le modèle par fine-tuning LoRA

### Le principe en cinq lignes

1. On écrit des exemples question-réponse sur le domaine visé.
2. On entraîne un **adaptateur LoRA** : quelques millions de paramètres ajoutés au modèle, dont les 3 milliards de paramètres d'origine restent figés.
3. On **fusionne** l'adaptateur dans le modèle.
4. On **convertit** le résultat au format GGUF 4 bits, celui qu'Ollama exécute sur processeur.
5. On mesure le gain sur des questions que le modèle n'a jamais vues.

Tous les fichiers sont dans le dossier `finetune`.

| Fichier | Rôle |
| --- | --- |
| `donnees.py` | Les faits sur l'école et leurs formulations ; produit `train.jsonl` et `test.jsonl` |
| `entrainer.py` | Entraîne l'adaptateur LoRA sur la carte graphique |
| `fusionner.py` | Fusionne l'adaptateur dans le modèle de base |
| `convertir.sh` | Convertit le modèle fusionné en GGUF 4 bits (dans un conteneur Docker) |
| `creer_modele.sh` | Crée le modèle `eps-assistant` dans Ollama |
| `evaluer.py` | Compare des modèles sur les questions de test |
| `env-eps.exemple` | Configuration de l'interface pour l'assistant de l'école |

### B1 — Préparer l'environnement Python

1. Créez un environnement virtuel dans le dossier `finetune` et activez PyTorch pour la carte NVIDIA. Le téléchargement fait environ 2,5 Go.

```powershell
cd finetune
python -m venv .venv
.venv\Scripts\python -m pip install torch==2.5.1 --index-url https://download.pytorch.org/whl/cu121
.venv\Scripts\python -m pip install -r requirements.txt
```

2. Vérifiez que la carte graphique est vue.

```powershell
.venv\Scripts\python -c "import torch; print(torch.__version__, torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

*Attendu : `2.5.1+cu121 True` suivi du nom de votre carte. Si vous lisez `False`, l'entraînement ne fonctionnera pas : voyez « Si ça bloque ».*

Restez dans le dossier `finetune` jusqu'à l'étape B6 incluse.

### B2 — Lire et générer les données

1. Ouvrez `donnees.py` dans un éditeur. Chaque fait est un triplet : une réponse, quatre formulations de question pour l'entraînement, et une cinquième formulation réservée au test.
2. Repérez les faits de la rubrique « Ce que l'assistant ne sait pas ». Pourquoi apprendre au modèle à dire « je n'ai pas cette information » ?
3. Générez les fichiers.

```powershell
$env:PYTHONUTF8 = "1"
.venv\Scripts\python donnees.py
```

*Attendu : `58 faits -> 232 exemples d'entraînement, 58 questions de test`, et deux nouveaux fichiers `train.jsonl` et `test.jsonl`.*

La ligne `$env:PYTHONUTF8 = "1"` évite les accents mal affichés. Elle vaut pour la fenêtre PowerShell en cours : retapez-la si vous en ouvrez une autre.

Les faits viennent des pages publiques de `https://www.polytecsousse.tn`, consultées le 13 novembre 2026. Les frais sont ceux de l'année 2026/2027.

### B3 — Mesurer le modèle avant l'entraînement

1. Interrogez le modèle de base sur 20 questions de test. Comptez 3 à 10 minutes selon votre processeur.

```powershell
.venv\Scripts\python evaluer.py llama3.2:3b --n 20 --sortie evaluation-avant.json
```

*Attendu : une ligne de ce type. Pendant le test, sur les 58 questions : rappel moyen de 19,8 %, 0 réponse à plus de 80 %.*

```
llama3.2:3b                  rappel moyen  2x.x % · réponses à plus de 80 % : 0/20 · ...
```

Le **rappel** est la part des mots importants de la réponse de référence (mots de 4 lettres ou plus, nombres) que l'on retrouve dans la réponse du modèle.

2. Ouvrez `evaluation-avant.json` et lisez trois réponses. Le modèle se trompe-t-il, ou invente-t-il avec assurance ?

### B4 — Entraîner l'adaptateur LoRA

1. Lancez l'entraînement. Choisissez `--lot` d'après le tableau des prérequis : 4 pour 16 Go de mémoire vidéo, 2 pour 12 Go, 1 pour 10 Go.

```powershell
.venv\Scripts\python entrainer.py --lot 4
```

*Attendu : au premier lancement, le téléchargement du modèle de base (6,4 Go), puis six lignes de ce type. La perte doit descendre de plus de 1 à moins de 0,01.*

```
232 exemples, 185 tokens au plus
trainable params: 24,313,856 || all params: 3,237,063,680 || trainable%: 0.7511
époque 1/6 · perte moyenne 1.461 · 29 s · mémoire GPU 11.9 Go
époque 2/6 · perte moyenne 0.260 · 57 s · mémoire GPU 11.9 Go
époque 3/6 · perte moyenne 0.070 · 87 s · mémoire GPU 11.9 Go
époque 4/6 · perte moyenne 0.027 · 116 s · mémoire GPU 11.9 Go
époque 5/6 · perte moyenne 0.006 · 146 s · mémoire GPU 11.9 Go
époque 6/6 · perte moyenne 0.002 · 175 s · mémoire GPU 11.9 Go
Adaptateur enregistré dans adaptateur/ (348 pas, 176 s)
```

L'entraînement a duré 3 minutes pendant le test. Avec `--lot 1` ou `--lot 2`, il est plus long.

2. Regardez le dossier `adaptateur` : le fichier `adapter_model.safetensors` pèse environ 97 Mo, contre 6,4 Go pour le modèle de base. Seuls 0,75 % des paramètres ont été entraînés.

### B5 — Fusionner l'adaptateur dans le modèle

```powershell
.venv\Scripts\python fusionner.py
```

*Attendu, après une minute environ : `Modèle fusionné enregistré dans fusionne/`. Le dossier `fusionne` pèse 6,1 Go.*

Cette étape est nécessaire parce que la version actuelle d'Ollama n'accepte plus les adaptateurs LoRA séparés.

### B6 — Convertir au format d'Ollama

1. Lancez la conversion dans un conteneur Docker. Comptez 8 à 12 minutes.

```powershell
docker run --rm -v "${PWD}:/work" -w /work python:3.12-slim bash convertir.sh
```

*Attendu : quelques minutes sans rien à l'écran (installation des outils), puis :*

```
== 1/2 conversion en GGUF 16 bits
INFO:hf-to-gguf:Model successfully exported to /tmp/f16.gguf
== 2/2 quantification en 4 bits (Q4_K_M)
== terminé
-rw-r--r-- 1 root root 2019377312 ... eps-assistant-q4_k_m.gguf
```

Des avertissements `debconf` ou `DeprecationWarning` peuvent s'afficher : ils sont sans conséquence.

2. Vérifiez le fichier produit. *Attendu : un fichier d'environ 2 Go, contre 6,1 Go avant quantification.*

```powershell
dir eps-assistant-q4_k_m.gguf
```

3. Revenez au dossier du projet.

```powershell
cd ..
```

### B7 — Créer le modèle dans Ollama

Depuis le dossier du projet (`llm-lab`), conteneurs démarrés :

1. Copiez le fichier GGUF et le script de création dans le conteneur Ollama.

```powershell
docker compose cp finetune\eps-assistant-q4_k_m.gguf ollama:/tmp/eps-assistant.gguf
docker compose cp finetune\creer_modele.sh ollama:/tmp/creer_modele.sh
```

*Attendu : deux lignes `Copied ...`.*

2. Créez le modèle.

```powershell
docker compose exec ollama sh /tmp/creer_modele.sh
```

*Attendu, après 30 secondes environ : `success`, puis la liste des modèles avec `eps-assistant:latest` et `llama3.2:3b`, de 2,0 Go chacun.*

Le script reprend le gabarit de prompt de `llama3.2:3b` : c'est ce même gabarit que `entrainer.py` a reproduit pendant l'entraînement.

### B8 — Mesurer le modèle après l'entraînement

```powershell
cd finetune
.venv\Scripts\python evaluer.py llama3.2:3b eps-assistant --n 20 --sortie evaluation-apres.json
cd ..
```

*Attendu : deux lignes. Le modèle spécialisé doit dépasser 80 % de rappel. Valeurs du test sur les 58 questions : 19,8 % et 0/58 avant, 87,1 % et 49/58 après, à la même vitesse.*

```
llama3.2:3b                  rappel moyen  2x.x % · réponses à plus de 80 % : 0/20 · ...
eps-assistant                rappel moyen  9x.x % · réponses à plus de 80 % : 1x/20 · ...
```

Pour l'évaluation complète, retirez `--n 20` : comptez alors 10 à 15 minutes.

### B9 — Discuter avec l'assistant dans l'interface web

1. Remplacez la configuration par celle de l'assistant de l'école et redémarrez l'API.

```powershell
copy finetune\env-eps.exemple .env
docker compose -f docker-compose.yml -f docker-compose.local.yml up -d
curl.exe -s http://localhost/api/health
```

*Attendu : `{"ok":true,"version":"dev","modele":"eps-assistant","modele_pret":true}`.*

2. Rechargez `http://localhost`, saisissez le code `test-123` et posez ces questions.

| Question | Réponse attendue |
| --- | --- |
| Où se trouve l'école ? | Route Ceinture Sahloul, Entrée Kalaa Sghira, 4021 Sousse, Tunisie |
| Combien coûte le cycle ingénieur pour un Tunisien ? | 7 849 DT par an en 2026/2027, en trois tranches de 3 250, 2 700 et 1 899 DT |
| Quels sont les parcours en génie informatique ? | Génie Logiciel, Data Science & IA, Business Intelligence, Cybersécurité, Internet of Things |
| Y a-t-il un club de robotique ? | Le Club Polyrobots |
| Explique le cloud en une phrase. | Une réponse générale correcte : le modèle n'a pas oublié le reste |

3. Comparez la réponse sur l'adresse avec celle que vous aviez notée à l'étape A4.

### B10 — Chercher les limites

Le modèle spécialisé n'est pas fiable à 100 %. Posez-lui ces questions, qui l'ont mis en défaut pendant le test.

| Question | Ce qui s'est passé pendant le test |
| --- | --- |
| Est-ce qu'il y a un internat à l'EPS ? | Il a affirmé qu'un foyer existe, alors qu'il devait dire qu'il ne sait pas |
| Quelles sont les dates limites des tranches ? | Il a inventé sept dates au lieu des deux vraies (31 décembre et 31 mars) |
| J'ai une licence, puis-je intégrer le cycle ingénieur ? | Il a répondu sur le Master 1, une question voisine |
| Décris-moi les locaux de l'école. | Il a donné l'adresse au lieu de décrire le campus |

Ouvrez ensuite `finetune\evaluation-apres.json`, triez mentalement par `rappel`, et lisez les réponses les plus faibles.

### Point de contrôle de la partie B

- [ ] Le fichier `finetune\eps-assistant-q4_k_m.gguf` existe et pèse environ 2 Go.
- [ ] `ollama list` montre `eps-assistant`.
- [ ] Vous donnez deux chiffres : le rappel avant et le rappel après.
- [ ] L'interface répond correctement sur l'adresse et les frais.
- [ ] Vous avez trouvé au moins une question où le modèle spécialisé invente.

### Questions de synthèse

1. Pourquoi évaluer sur des formulations différentes de celles de l'entraînement ?
2. L'adaptateur pèse 97 Mo et le modèle 6,4 Go. Qu'est-ce que cela dit de la méthode LoRA ?
3. Le fichier final fait 2 Go au lieu de 6,1 Go. Qu'a-t-on perdu, et pourquoi l'accepte-t-on ?
4. Le modèle invente encore sur certaines questions. Citez deux façons de réduire ce risque. Attendu : plus de formulations par fait, plus d'exemples « je ne sais pas », ou une recherche documentaire (RAG) qui fournit le texte source au modèle.
5. Les frais changeront l'an prochain. Que faudra-t-il refaire ? Quelle approche éviterait de réentraîner ?

### Pour aller plus loin : votre propre assistant

Remplacez les faits de `donnees.py` par ceux d'un domaine qui vous est propre (un club, une association, un produit), en gardant la même structure : une réponse, quatre questions d'entraînement, une question de test. Refaites ensuite les étapes B2 à B9. Visez au moins 40 faits.

### Faire le ménage

Les gros dossiers ne servent plus une fois le fichier GGUF produit.

```powershell
Remove-Item -Recurse -Force finetune\fusionne
docker compose down
```

- `finetune\fusionne` : 6,1 Go libérés.
- Le modèle de base téléchargé (6,4 Go) est dans `C:\Users\VOTRE-NOM\.cache\huggingface` : supprimez ce dossier si vous ne comptez pas réentraîner.
- `docker compose down` arrête et supprime les conteneurs, mais conserve les modèles d'Ollama. Ajoutez `-v` pour les supprimer aussi.

---

## Si ça bloque

| Symptôme | Remède |
| --- | --- |
| `docker` : « failed to connect to the docker API » | Docker Desktop n'est pas démarré : lancez-le et attendez la baleine fixe |
| `Bind for 0.0.0.0:80 failed: port is already allocated` | Un autre programme occupe le port 80 : arrêtez-le, ou arrêtez un autre projet Docker qui l'utilise |
| En-tête de la page : « modèle absent » | Le modèle indiqué dans `.env` n'existe pas dans Ollama : `docker compose exec ollama ollama list`, puis refaites l'étape A3-3 ou B7 |
| `torch.cuda.is_available()` renvoie `False` | Pilote NVIDIA absent ou trop ancien, ou PyTorch installé sans l'option `--index-url ...cu121` : désinstallez avec `pip uninstall torch` et refaites B1 |
| `CUDA out of memory`, ou entraînement très lent (plus de 3 minutes par époque) | La mémoire vidéo déborde : relancez avec `--lot 2` ou `--lot 1`, et fermez les jeux et applications 3D |
| `evaluer.py` : « Connection refused » | Le serveur de modèle n'est pas joignable : relancez avec les deux fichiers, `docker compose -f docker-compose.yml -f docker-compose.local.yml up -d` |
| `convertir.sh` : `$'\r': command not found` | Le script a des fins de ligne Windows : refaites `git clone`, ou enregistrez le fichier avec des fins de ligne LF |
| `convertir.sh` : « Dossier fusionne/ introuvable » | Vous n'êtes pas dans `finetune`, ou l'étape B5 n'a pas été faite |
| Accents illisibles dans PowerShell | Tapez `$env:PYTHONUTF8 = "1"` avant les commandes Python |
| `creer_modele.sh` : « Fichier introuvable dans le conteneur » | Refaites la copie de l'étape B7-1 |

---

## Pour l'enseignant : ce qui a été vérifié

| Étape | Vérification |
| --- | --- |
| A3 à A6 | Exécutées sur le poste de test, avec un nom de projet Docker différent de celui du guide |
| A1 | Non rejouée telle quelle : le test est parti du dossier local, pas d'un `git clone` |
| B1 | Rejouée dans un environnement Python neuf : installation, détection de la carte, puis une époque d'entraînement |
| B2 à B5 | Exécutées ; les sorties citées sont celles du test |
| B4 avec téléchargement automatique du modèle de base | Non rejouée telle quelle : le test a utilisé une copie locale du même modèle (`unsloth/Llama-3.2-3B-Instruct`) |
| B6 | La conversion a réussi une fois, lancée par l'enseignant. Le script a été retouché ensuite (installation de `git`, version figée de llama.cpp) : **à relancer une fois avant le cours** |
| B7 à B9 | Exécutées avec les commandes du guide, en PowerShell pour les copies |
| Linux et macOS | Non testés. Sur Mac, il n'y a pas de carte NVIDIA : partie A, puis B7 à B10 |

Les fichiers ajoutés au kit sont déjà dans le dépôt modèle `nevermind78/llm-lab` : dossier `finetune` (scripts et données, sans les modèles), `docker-compose.local.yml`, `.env.exemple`, `.gitattributes`, `pytest.ini`. Avant de distribuer le guide, mettez le fichier `eps-assistant-q4_k_m.gguf` (2 Go) à disposition des étudiants sans carte graphique, par un partage de fichiers : il est trop gros pour GitHub.

Les données d'entraînement décrivent l'école telle que son site public la présentait le 13 novembre 2026. Faites-les relire par l'administration avant de présenter cet assistant comme une source d'information.
