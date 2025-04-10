Rapport sur les techniques de crawling pour la collecte d'URLs
Introduction
Ce rapport documente l'approche utilisée pour crawler deux sites web différents (exemple1.com et exemple2.com) afin de collecter leurs URLs internes. L'objectif était de construire un inventaire complet des pages pour une analyse ultérieure.
Bonnes pratiques de crawling
Exécution en arrière-plan
Pour une exécution efficace sur Linux, il est recommandé de lancer le script en arrière-plan dès le départ :
nohup python3 crawler.py > crawler_output.log 2>&1 &

Nohup = no hang up : permet d’exécuter une commande en arrière-plan sans qu’elle soit arrêtée même quand on ferme le terminal.
Cette commande permet :
L'exécution du script en arrière-plan (&)
La persistance du processus si le terminal est fermé (nohup)
La redirection des sorties standard et d'erreur vers un fichier log (> crawler_output.log 2>&1)
Idéalement, on a aussi configuré un logger dans le script, comme ça on peut monitorer le travail effectué dans le fichier .log

Il y a deux sites différents à  scraper. Il faut évaluer le nombre de pages pour chacun des sites.

Premier but: Récupérer tous les url internes avec le crawler. Faire une liste des url par domaine dans deux JSON.

Analyse des sites cibles
Site exemple1.com
Caractéristiques du site :
Structure d'URL simple avec peu de sous-répertoires
Articles avec structure d'URL : exemple1.com/[slug-article] (sans sous-répertoire)
Présence d'interstitiels publicitaires insérés entre les navigations
Fragment #google_vignette ajouté aux URLs nécessitant une action de fermeture
Défi principal : Gestion des interstitiels publicitaires qui modifient temporairement l'URL avec un fragment (#google_vignette) jusqu'à ce que l'utilisateur clique sur le bouton "Fermer" ou à coté de la fenetre pop up pour la faire disparaitre.
Résultat : environ 2000 URLs collectées avec un crawler relativement simple.
Site exemple2.com
Caractéristiques du site :
Structure plus complexe
Utilisation intensive de JavaScript
Liens implémentés via divers mécanismes (attributs onclick, data-*, etc.)
Présence probable de mesures anti-bot
Défi principal : Le crawler simple qui fonctionnait pour exemple1.com échoue complètement sur exemple2.com.
Solutions techniques avancées
Causes possibles de l'échec
Protection anti-bot/anti-scraping :


Le site exemple2.com utilise peut-être des technologies comme Cloudflare, reCAPTCHA ou d'autres systèmes anti-bot.
Ces protections détectent les comportements automatisés et bloquent l'accès.
En-têtes HTTP insuffisants :


Le site peut vérifier les en-têtes HTTP pour s'assurer qu'ils proviennent d'un navigateur légitime.
Notre crawler n'envoie peut-être pas tous les en-têtes attendus.
Problèmes de JavaScript :


Le site pourrait nécessiter l'exécution de certains scripts JavaScript avant d'afficher son contenu.
Notre approche actuelle pourrait ne pas attendre assez longtemps ou ne pas exécuter le bon JavaScript.
Restrictions géographiques ou IP :


Le site pourrait bloquer les requêtes venant de certaines adresses IP (comme les serveurs cloud).
Structure du site différente :


Les sélecteurs que nous utilisons pour trouver les liens pourraient ne pas correspondre à la structure de exemple2.com.


Crawler avancé développé
Pour surmonter ces obstacles, un crawler avancé a été développé avec les fonctionnalités suivantes :
1. Mode furtif avancé
Masquage des indicateurs d'automatisation
Rotation des user-agents
En-têtes HTTP réalistes et complets
Scripts JavaScript anti-détection
2. Simulation de comportement humain
Mouvements de souris aléatoires
Défilement progressif avec pauses
Délais variables entre requêtes
Sélection non-séquentielle des URLs à visiter
3. Extraction avancée de liens
Méthode traditionnelle via sélecteurs <a>
Extraction JavaScript pour liens cachés dans :
 // Exemples de liens cachés<button onclick="window.location.href='https://exemple2.com/page'">Voir</button><div class="card" data-product-url="https://exemple2.com/produit/123">Produit</div>


4. Normalisation intelligente des URLs
Processus qui standardise les URLs pour éliminer les duplications :
Normalisation du schéma (http:// ou https://)
Suppression des barres obliques finales
Conservation sélective des paramètres de requête importants
Suppression des fragments non fonctionnels
5. Gestion des obstacles
Détection et fermeture automatique des bandeaux de cookies
Pages fraîches pour chaque visite
Système de tentatives multiples avec backoff exponentiel
Blocs essentiels du crawler avancé
Voici un résumé des fonctions principales du crawler dans leur ordre d'apparition, avec leurs liens logiques :
1. Configuration et initialisation
SITE_CONFIG : Dictionnaire contenant tous les paramètres de configuration (domaine, URL de départ, délais, etc.)
USER_AGENTS : Liste d'agents utilisateurs pour la rotation (simule différents navigateurs)
2. load_existing_urls()
Objectif : Charger les URLs déjà découvertes lors de sessions précédentes
Entrée : Chemin du fichier JSON
Sortie : Ensemble (set) d'URLs
Relation : Fournit le point de départ pour une reprise de crawling
3. configure_browser()
Objectif : Initialiser le navigateur avec des paramètres avancés anti-détection
Sortie : Instances playwright, browser et context
Actions clés :
Désactivation des fonctionnalités d'automatisation détectables
Configuration des en-têtes HTTP réalistes
Injection de scripts pour masquer la présence de Playwright
Relation : Crée l'environnement de navigation utilisé par les autres fonctions
4. extract_links_with_multiple_methods()
Objectif : Extraire tous les liens d'une page par différentes techniques
Entrées : Instance de page, URL actuelle
Sortie : Ensemble d'URLs découvertes
Méthodes :
Sélecteurs CSS standards pour balises <a>
Extraction via JavaScript pour liens dans onclick, data-attributes, etc.
Relation : Fournit les données brutes de liens à visit_page()
5. visit_page()
Objectif : Visiter une page et extraire tous ses liens internes
Entrées : Contexte du navigateur, URL, ensembles de tracking
Sortie : Ensemble de liens internes, statut de succès
Actions clés :
Navigation avec comportement humain simulé
Gestion des popups et bandeaux de cookies
Filtrage des liens externes/médias
Relation : Appelée par crawl_site() pour chaque URL à visiter
6. crawl_site()
Objectif : Fonction principale coordonnant tout le processus de crawling
Sortie : Ensemble de toutes les URLs découvertes
Actions clés :
Initialisation avec URLs existantes ou URL de départ
Boucle principale de crawling avec sélection aléatoire des URLs
Gestion des tentatives multiples et des échecs
Sauvegarde périodique des résultats
Relation : Cœur du programme, utilise toutes les autres fonctions
7. save_urls_to_json()
Objectif : Sauvegarder les URLs découvertes dans un fichier JSON
Entrées : Ensemble d'URLs, paramètres de sortie
Actions :
Formatage des données avec métadonnées (date, compteur)
Création du répertoire de sortie si nécessaire
Relation : Appelée périodiquement par crawl_site() et à la fin du processus
8. main()
Objectif : Point d'entrée du programme
Actions :
Affichage des messages de démarrage
Appel de crawl_site()
Affichage des statistiques finales
Relation : Initialise le processus et gère la séquence d'exécution
Flux logique du programme
main() démarre l'exécution
crawl_site() initialise le processus avec load_existing_urls() et configure_browser()
Pour chaque URL à visiter, crawl_site() appelle visit_page()
visit_page() utilise extract_links_with_multiple_methods() pour découvrir de nouveaux liens
Les nouveaux liens sont ajoutés à la liste à visiter dans crawl_site()
Périodiquement, save_urls_to_json() sauvegarde les résultats intermédiaires
À la fin, les statistiques sont affichées et les résultats finaux sauvegardés
Cette architecture modulaire permet :
Une gestion efficace des ressources
Une reprise facile en cas d'interruption
Une adaptation rapide à différents sites par simple modification de la configuration
Un contournement des mécanismes anti-bot grâce à des techniques avancées d'imitation du comportement humain

Résultats et observations
Le crawler avancé a permis de collecter de nombreuses URLs sur exemple2.com, y compris certaines URLs de ressources CSS non désirées. Le système de reprise intelligent permet d'interrompre et de reprendre le crawling sans perdre de progrès.
Recommandations pour futurs projets
Filtrage des URLs : Améliorer les règles de filtrage pour exclure les ressources non pertinentes (CSS, etc.)
Rotation d'IPs : Ajouter un système de proxy/VPN en cas de blocage par IP
Mode debug visuel : Utiliser headless=False pour observer le comportement du browser
Ajustement des délais : Modifier les valeurs de délai en fonction de la sensibilité du site
Mode progressif : Pour les sites particulièrement restrictifs, limiter le nombre de pages par session


*************************************************************************************************************






















Annexe 1
1. Extraction JavaScript avancée des liens cachés
Dans les sites web modernes, tous les liens ne sont pas forcément implémentés avec des balises <a href="..."> standard. De nombreux liens sont "cachés" dans d'autres éléments du DOM via JavaScript ou des attributs de données personnalisés. Le crawler utilise une technique d'extraction avancée pour les trouver :
Comment ça fonctionne
Le code exécute du JavaScript directement dans la page avec page.evaluate() pour rechercher :
// Récupérer les liens dans les onclick, data-attributes, etc.
document.querySelectorAll('*').forEach(el => {
    // Chercher dans onclick
    if (el.onclick) {
        const onclickStr = el.onclick.toString();
        const matches = onclickStr.match(/window\.location\.href\s*=\s*['"]([^'"]+)['"]/g);
        if (matches) {
            matches.forEach(match => {
                const url = match.replace(/window\.location\.href\s*=\s*['"]/g, '').replace(/['"]$/, '');
                if (url) extractedLinks.add(url);
            });
        }
    }
    
    // Chercher dans data-attributes
    if (el.dataset) {
        Object.values(el.dataset).forEach(value => {
            if (value && value.startsWith('http')) {
                extractedLinks.add(value);
            }
        });
    }
});


Exemples concrets de liens cachés
Handlers onclick :

 <button onclick="window.location.href='https://exemple2.com/page-produit'">Voir produit</button>


Data-attributes :

 <div class="product-card" data-product-url="https://exemple2.com/produit/123">Produit</div>


Liens dans des scripts :

 <div id="product" class="clickable"></div>
<script>
  document.getElementById('product').addEventListener('click', function() {
    window.location.href = 'https://exemple2.com/produit/456';
  });
</script>


Routage SPA (Single Page Applications) :

 <div data-route="/produits/categorie" class="nav-item">Produits</div>


Notre extracteur JavaScript analyse le code de la page pour trouver toutes ces variantes, ce qui permet de découvrir des liens que les crawlers simples manqueraient.
2. Normalisation intelligente des URLs
La normalisation d'URL est le processus qui consiste à transformer les URLs brutes en un format standardisé pour éliminer les duplications et améliorer la cohérence.
Pourquoi c'est nécessaire
Voici des exemples d'URLs qui pointent vers la même page mais semblent différentes :
https://exemple2.com/produits/
https://exemple2.com/produits
https://exemple2.com/produits?utm_source=google
https://exemple2.com/produits/?utm_campaign=spring
Le processus de normalisation dans notre crawler
# Normaliser l'URL
normalized_url = f"{parsed.scheme}://{parsed.netloc}{parsed.path}"
if parsed.path.endswith('/'):
    normalized_url = normalized_url[:-1]
    
# Conserver les paramètres importants
if parsed.query:
    params = parsed.query.split('&')
    important_params = [p for p in params if p.startswith(('id=', 'page=', 'category=', 'p='))]
    if important_params:
        normalized_url += '?' + '&'.join(important_params)


Ce que notre normalisation fait concrètement
Standardisation du schéma : Utilise toujours http:// ou https://


Http://exemple2.com → http://exemple2.com
Gestion des barres obliques : Supprime les barres obliques finales


https://exemple2.com/produits/ → https://exemple2.com/produits
Filtrage intelligent des paramètres : Ne conserve que les paramètres qui définissent le contenu


https://exemple2.com/produits?id=123&utm_source=google → https://exemple2.com/produits?id=123
Garde : id=, page=, category=, p= (qui changent généralement le contenu)
Supprime : paramètres de tracking, analytics, etc.
Gestion des fragments : Supprime les fragments d'URL (#)


https://exemple2.com/page#section2 → https://exemple2.com/page
Exception : conserve certains fragments qui peuvent définir du contenu unique
Avantages de cette approche
Réduction des duplications : Le crawler ne visite pas la même page plusieurs fois sous différentes formes d'URL.


Meilleure couverture : Se concentre sur la structure réelle du contenu plutôt que sur les variations d'URL.


Prise en compte du contexte : Conserve les paramètres qui définissent réellement du contenu unique.


Adaptabilité : La logique peut être ajustée en fonction des spécificités du site (par exemple, conserver des paramètres supplémentaires si nécessaire).


Cette normalisation intelligente est particulièrement utile pour les sites e-commerce comme exemple2.com, qui utilisent souvent de nombreux paramètres de requête pour le tracking, mais où seuls quelques-uns définissent réellement le contenu de la page.


Gestion des blocages :


Détection et fermeture automatique des bandeaux de cookies
Nouvelles pages pour chaque visite pour éviter la contamination d'état
Système de tentatives multiples avec délais croissants
Configuration flexible :


Paramètres facilement ajustables (délais, timeouts, etc.)
Option de mode visible du navigateur pour déboguer



