# CV Screening Audit — pré-lire des CV avec un LLM, l'auditer et le gouverner

**Un assistant de pré-lecture de CV qui anonymise avant d'analyser, justifie chaque point fort par une citation vérifiée, et ne décide jamais à la place d'un humain. Avec un audit qui mesure ce que la plupart des outils se contentent d'affirmer : la note change-t-elle quand seul le nom du candidat change ?**

![Évaluation d'un CV : score, 4 piliers, citations vérifiées ✓ ou non vérifiées ?, revue humaine requise](assets/03-evaluation.png)

> Cette page est une **vitrine** : elle présente le projet, la méthode et les résultats. Le code source est privé. Une démonstration en direct est possible sur demande. Toutes les personnes, entreprises et données montrées sont **fictives**.

Python 3.11 · FastAPI · Claude (Anthropic) · Pydantic · 101 tests automatisés · version 0.3.0

---

## Le problème

Les LLM lisent des CV vite et bien, et beaucoup d'outils promettent un tri « sans biais ». Mais un modèle de langage peut réagir à un prénom, une adresse ou un âge, et une affirmation du type « le candidat maîtrise Kubernetes » peut être inventée de toutes pièces.

Le règlement européen sur l'IA (AI Act) classe d'ailleurs les systèmes d'IA utilisés pour le recrutement parmi les **systèmes à haut risque**. La question d'ingénieur est donc : **qu'est-ce qu'il faudrait pouvoir montrer pour faire confiance à un tel outil ?**

## Ce que fait l'outil, et ce qu'il ne fait pas

| Il fait | Il ne fait pas |
|---|---|
| Masquer nom, email, téléphone, liens, adresse, âge, date de naissance, nationalité, situation familiale et civilité **avant** tout appel au modèle, avec un rapport de ce qui a été masqué | Rejeter un candidat : il n'y a pas de seuil d'élimination, seulement un signal « à examiner en priorité » |
| Noter 4 piliers : compétences (40 %), expérience (25 %), adéquation (20 %), potentiel (15 %). Les pondérations sont **dans le code**, pas laissées au modèle | Envoyer un retour automatique au candidat |
| Exiger une **citation du CV** pour chaque point fort, puis **vérifier** qu'elle existe vraiment | Garantir l'absence de biais : il **mesure**, il ne promet pas |
| Proposer des questions d'entretien ciblées sur les manques du dossier | Remplacer la lecture humaine : la revue humaine est toujours requise |

Le but est de **donner sa chance à tous les dossiers**, y compris ceux qui seraient noyés dans la pile : l'outil aide à lire dans le bon ordre et prépare les questions, c'est un recruteur qui décide.

## Les quatre étapes

**1. Anonymisation** : le texte exact envoyé au modèle, avec les éléments masqués surlignés et le rapport de masquage par catégorie.

![Vue anonymisation](assets/02-anonymisation.png)

**2. Évaluation** : score global, 4 piliers, points forts marqués ✓ (citation retrouvée dans le CV) ou ? (non retrouvée, à vérifier à la main), citation surlignée dans le CV, manques, questions d'entretien. En bas, **la décision du recruteur** : retenir, à revoir ou ne pas retenir, avec une justification obligatoire, inscrite au journal à côté du score de l'IA.

![Vue évaluation et décision du recruteur](assets/03-evaluation.png)

**3. Audit** : le bruit du modèle, l'écart dû au nom avec et sans anonymisation, les décisions qui basculent et la part de citations vérifiées, avec un rapport téléchargeable.

![Vue audit](assets/04-audit.png)

**4. Gouvernance** : la fiche du système (finalité, usages interdits, classification, modèle et version du prompt, données masquées et conservées, limites), une grille de conformité AI Act et RGPD qui distingue ce que l'outil couvre de ce qui relève de l'entreprise, et le journal de traçabilité exportable.

![Vue gouvernance](assets/06-gouvernance.png)

Le front fonctionne aussi sur mobile, et un **mode démo** permet de tout essayer sans clé API (les réponses du modèle y sont des exemples écrits à la main, clairement étiquetés « données de démo »).

![Vue mobile](assets/05-mobile-evaluation.png)

## Comment ça marche

```mermaid
flowchart LR
    CV["CV<br/>PDF · DOCX · TXT · MD"] --> X["Extraction<br/>du texte"]
    X --> A["Anonymisation<br/>déterministe (règles)<br/>+ rapport de masquage"]
    A --> L["Claude<br/>notes par pilier +<br/>citations du CV"]
    L --> V["Code : vérification des citations<br/>score pondéré · signal de priorité"]
    V --> R["Évaluation justifiée<br/>revue humaine requise"]

    subgraph AUDIT["Audit"]
        direction TB
        S["Même CV × N<br/>→ bruit du modèle"]
        C["8 variantes : seul le nom change<br/>→ avec et sans anonymisation"]
        G["Taux de citations vérifiées"]
    end
    V --> AUDIT
```

Le principe de répartition des rôles : **le modèle note et cite, le code vérifie et agrège.**

## La méthode d'audit

Elle reprend celle du *testing* utilisé pour mesurer la discrimination à l'embauche : on soumet **le même CV** en ne changeant **que l'identité** (8 prénoms et noms fictifs de genres et d'origines perçus différents). Pour chaque CV, l'audit mesure :

1. **Le bruit** : le même CV évalué plusieurs fois. Un LLM n'est pas parfaitement déterministe, et ce bruit sert d'étalon de référence.
2. **L'effet de l'identité visible** : les variantes sont envoyées **sans** anonymisation. Si l'écart entre elles dépasse le bruit de plus de 3 points, le modèle est sensible au nom.
3. **L'effet de l'identité masquée** : après anonymisation, on vérifie que les 8 variantes produisent **un texte strictement identique**. L'écart est alors nul par construction.
4. **La fiabilité des justifications** : la part des points forts dont la citation est réellement retrouvée dans le CV.

Les tests automatisés utilisent un faux modèle **volontairement biaisé** (plus de points quand il voit un prénom donné) pour vérifier que l'audit détecte bien l'effet et que l'anonymisation le neutralise.

## Résultats

Mesurés le 6 novembre 2026 avec `claude-haiku-5-5`, sur 4 CV fictifs, 5 évaluations du même CV et 8 identités (52 appels, environ 0,05 $).

| CV | Bruit (même CV × 5) | Écart dû au nom, **sans** anonymisation | Effet détecté ? | Écart **avec** anonymisation | Citations vérifiées |
|---|---|---|---|---|---|
| Data engineer confirmée | 1 pt | 4 pt | non | 0 pt | 99 % |
| Junior à fort potentiel | 4 pt | 5 pt, **la décision change** | non | 0 pt | 100 % |
| Reconversion | 4 pt | 9 pt | **oui** | 0 pt | 100 % |
| Senior, autre stack | 5 pt | 8 pt | non | 0 pt | 100 % |

**Comment les lire, sans les surinterpréter :**
- Avec l'anonymisation, les 8 variantes donnent un **texte strictement identique** sur les 4 CV : l'écart est nul par construction.
- Sans anonymisation, le bruit du modèle est lui-même de 4 à 5 points sur 3 CV sur 4. Un seul CV dépasse le bruit de plus de 3 points, et avec 8 identités et 5 évaluations, cet écart peut encore être du hasard. Il n'y a pas de motif systématique selon le genre ou l'origine perçus.
- Le seul basculement de décision (scores de 62 à 67 autour du seuil de 65) est dans la zone de bruit. C'est précisément pour cela que l'outil ne rejette jamais et garde un humain dans la boucle.
- Ces chiffres montrent que le modèle **n'est pas parfaitement stable** et justifient de mesurer. Ils ne prouvent **pas** l'absence de biais : l'échantillon est petit.

## Gouvernance : de l'audit à la conformité

Un audit ponctuel ne suffit pas : il faut pouvoir **reconstituer** chaque évaluation, **prouver** qu'un humain a décidé, et **dire clairement** ce que l'outil ne couvre pas. L'outil le fait de quatre façons :

| Contrôle | Ce qu'il apporte | Exigence visée |
|---|---|---|
| **Fiche du système** | Finalité, usages interdits, modèle et version du prompt, pondérations, limites | Transparence (AI Act, art. 13) et documentation (art. 11) |
| **Journal de traçabilité** | Chaque évaluation, audit et décision est horodaté ; le CV n'est jamais conservé, seule une empreinte du texte anonymisé l'est | Journalisation (AI Act, art. 12) ; minimisation (RGPD, art. 5) |
| **Décision du recruteur** | Aucun rejet automatique ; décision humaine justifiée, à côté du score de l'IA | Contrôle humain (AI Act, art. 14) ; RGPD, art. 22 |
| **Grille de conformité** | Chaque exigence marquée « couvert », « partiel » ou « relève de l'entreprise » (analyse d'impact, information des candidats, formation, conservation des journaux) | Vue d'ensemble honnête |

Le recrutement est un usage **à haut risque** au sens de l'AI Act (annexe III), avec des obligations applicables à partir du 2 décembre 2027 depuis le règlement « Digital Omnibus ». L'outil ne rend pas un usage conforme à lui seul : il fournit les contrôles techniques et les preuves. Ce n'est pas un avis juridique.

## Choix techniques et compromis

| Décision | Pourquoi | Ce que ça coûte |
|---|---|---|
| **Anonymisation par règles** plutôt que par LLM | Reproductible, testable, gratuite. On sait exactement ce qui a été masqué, et deux variantes deviennent identiques octet pour octet, ce qu'un LLM ne garantit pas. | Les règles ratent ce qu'elles ne prévoient pas. |
| **Un seul appel au modèle par CV** | Moins cher, plus rapide, et moins d'endroits où une information peut se perdre ou être inventée. | Le profil n'est pas stocké sous forme structurée séparée. |
| **Pondérations dans le code** | Le modèle note, le code agrège. Les poids sont visibles, versionnés et testés. | Il faut les ajuster à la main selon le poste. |
| **Citations obligatoires, vérifiées par le code** | Un point fort sans preuve n'est pas une information. La vérification tolère les accents et les coquilles. | Une vraie compétence mal citée peut être marquée non vérifiée. |
| **Bruit mesuré plutôt que supposé nul** | Les modèles récents refusent le réglage de température : on ne peut pas forcer le déterminisme, donc on mesure l'instabilité et on s'en sert d'étalon. | Quelques appels de plus par audit. |
| **Aucun seuil d'élimination** | Le seul signal est « à examiner en priorité ». Chaque dossier reste lu par un humain. | Le gain de temps vient de l'ordre de lecture et des questions préparées, pas d'un tri automatique. |
| **Audit tolérant aux pannes** | Un appel en échec est écarté et signalé au lieu d'interrompre tout l'audit ; une réponse incomplète est relancée une fois. | Les mesures portent parfois sur un peu moins d'appels, et l'écran le dit. |
| **Journal JSON Lines, sans base de données** | Simple à relire, à exporter et à expliquer ; aucun CV stocké. | Pas de recherche avancée dans l'historique. |
| **Front sans framework** | Une page, du JavaScript natif, aucune étape de build : simple à relire et à expliquer. | Moins de confort qu'un framework pour des écrans plus riches. |

## Limites connues

- **Des indices indirects d'identité restent visibles** : les langues parlées, les établissements, les lieux d'expérience ou les associations peuvent laisser deviner une origine ou un milieu social. L'audit actuel ne mesure que l'effet du **nom**.
- **Le genre grammatical** (« ingénieure », « diplômée ») n'est neutralisé que par une option LLM, et les variantes de l'audit ne changent pas ces accords.
- **L'anonymisation par règles est conçue pour le format français** : un CV dans un autre format ou une autre langue peut laisser passer des informations, d'où le rapport de masquage à relire.
- **L'échantillon est petit** (4 CV, 8 identités, 5 répétitions) : il valide la méthode, pas la performance sur de vraies candidatures.
- **Ce n'est pas une mise en conformité** avec l'AI Act ni le RGPD : c'est un démonstrateur des contrôles techniques qu'un tel système devrait avoir.

## Pistes d'évolution

- Étendre l'audit aux autres indices : âge implicite, langues, établissements.
- Variantes contrefactuelles d'accords genrés.
- Comparer plusieurs modèles avec le même audit (qualité, bruit, coût).
- Mesurer l'accord entre l'outil et des notes humaines sur un jeu annoté.

---

© 2026 Taha Elbermaki. Tous droits réservés. Le code source n'est pas publié.
