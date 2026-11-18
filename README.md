# LLM Lab — kit de TP AWS Academy

Un LLM ouvert servi par Ollama sur une instance EC2, une API FastAPI, une interface web de test,
un modèle CloudFormation et un pipeline GitHub Actions (tests, image dans ECR, déploiement).
En complément, un fine-tuning LoRA qui spécialise le modèle sur Polytechnique Sousse.

## Guides

| Guide | Contenu | Où |
| --- | --- | --- |
| [Formation pas à pas sur AWS](docs/formation-aws-pas-a-pas.md) | Labs 0 à 6 dans AWS Academy Learner Lab, puis le guide de l'enseignant | AWS |
| [Guide local et fine-tuning](docs/guide-local-fine-tuning.md) | Faire tourner l'application sur son poste, puis spécialiser le modèle par LoRA | Poste de l'étudiant |

## Contenu du dépôt

| Chemin | Rôle | Lab |
| --- | --- | --- |
| `app/main.py` | API : `/`, `/api/health`, `/api/chat` (réponse en flux) | 2 |
| `app/static/index.html` | Interface de discussion | 2 |
| `app/Dockerfile`, `docker-compose.yml` | Image de l'API, pile Ollama + API | 2, 3 |
| `tests/test_api.py` | 7 tests avec un faux serveur de modèle | 5 |
| `infra/stack.yaml` | CloudFormation : groupe de sécurité + instance auto-configurée | 4 |
| `pipeline/ci-cd.yml` | Pipeline : tests, image vers ECR, déploiement, vérification. À copier dans `.github/workflows/` au lab 5 | 5 |
| `docker-compose.local.yml`, `.env.exemple` | Travail sur son propre poste | Guide local |
| `finetune/` | Données, entraînement LoRA, fusion, conversion GGUF, création du modèle Ollama, évaluation | Guide local |

## Ce qui a été vérifié

Vérifié le 13 novembre 2026 dans un Learner Lab réel (vue étudiant, `us-east-1`) : les labs 1 à 6 ont été
déroulés en ligne de commande, de l'instance manuelle au nettoyage. Les 7 tests passent ; `infra/stack.yaml`
passe cfn-lint. Les labs 0 à 2 ont aussi été déroulés à la souris par l'enseignant.

Vérifié le 14 novembre 2026 sur un poste Windows 11 avec une carte NVIDIA de 16 Go : le guide local, du
démarrage de l'application à l'assistant spécialisé dans l'interface web.

Non vérifié : les labs 3 à 6 à la souris, et le guide local sous Linux et macOS. Le détail figure à la fin
de chaque guide.
