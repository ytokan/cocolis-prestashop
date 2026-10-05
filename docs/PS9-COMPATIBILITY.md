# Fork cocolis — modifications par rapport au module d'origine

Ce document résume toutes les modifications apportées au module `modules/cocolis` (et à l'override qu'il installe) pour le rendre compatible avec PrestaShop 9.1.5, suite aux erreurs rencontrées à l'installation, à la configuration et à l'utilisation du module.

Version d'origine du module : `1.0.10`.

## 1. `cocolis.php` — `install()` : constante globale obsolète

**Problème** : `Carrier::getCarriers($cookie->id_lang, true, false, false, null, PS_CARRIERS_AND_CARRIER_MODULES_NEED_RANGE)` utilisait une constante globale (`PS_CARRIERS_AND_CARRIER_MODULES_NEED_RANGE`) supprimée en PS9 — devenue une constante de classe `Carrier::PS_CARRIERS_AND_CARRIER_MODULES_NEED_RANGE`.
**Erreur** : `Error: Undefined constant "PS_CARRIERS_AND_CARRIER_MODULES_NEED_RANGE"`, bloquant l'installation.
**Correctif** : fallback `defined()/constant()` avec repli sur la constante de classe.

## 2. `cocolis.php` — `install()` : ordre d'exécution `addRanges()`/`addZones()`

**Problème** : `addZones($carrier)` était appelé **avant** `addRanges($carrier)`. Or `Carrier::addZone()` (core) ne crée les lignes de grille tarifaire (`ps_delivery`) que pour les tranches de prix/poids **déjà existantes** au moment de l'appel. Comme aucune tranche n'existait encore, aucune ligne `ps_delivery` n'était jamais créée.
**Conséquence** : le transporteur (`shipping_method = SHIPPING_METHOD_PRICE`) était filtré silencieusement par `Carrier::getMaxDeliveryPriceByPrice()` avant même d'atteindre le code du module — il n'apparaissait jamais au checkout, sans aucune erreur.
**Correctif** : inversion de l'ordre — `addRanges()` est désormais appelé avant `addZones()`, pour le transporteur standard et le transporteur assurance.
**Statut** : correctif appliqué et grille tarifaire confirmée non-vide après réinstallation ; **la disponibilité du transporteur au checkout reste en cours de validation** (voir section "En cours d'investigation").

## 3. `cocolis.php` — hooks enregistrés sans méthode correspondante

**Problème** : `install()` enregistrait des hooks sans méthode `hookXxx()` associée (obligatoire pour PrestaShop) :
- `header` (obsolète, remplacé par `displayHeader`)
- `backOfficeHeader` (jamais implémenté)
- `updateCarrier` (n'existe pas comme hook réel ; doublon voulu de `actionCarrierUpdate`, jamais implémenté non plus)
- `moduleRoutes` (valide mais inutile : le module n'a pas de routes personnalisées)
- `displayAdminOrderContentOrder`, `displayAdminOrderTabOrder`, `displayAdminOrder` (jamais implémentés)

**Erreur** : *"Hook with the name header has been registered by Cocolis, but the corresponding method hookHeader has not been defined"* — installation refusée par PrestaShop.
**Correctif** : suppression de ces enregistrements morts dans `install()`.

## 4. `cocolis.php` — migration de l'onglet de suivi commande (hooks admin obsolètes en PS9)

**Problème** : les hooks `displayAdminOrderTabShip` / `displayAdminOrderContentShip` (implémentés et enregistrés) ne sont **plus jamais déclenchés** par la page commande de PS9 (entièrement réécrite en Symfony/Twig). Vérifié par recherche exhaustive : aucun `Hook::exec()` de ces noms nulle part dans le core PS9.1.5, y compris pour le module officiel `psshipping` qui les enregistre aussi sans qu'ils soient jamais appelés.
**Conséquence** : régression silencieuse — l'onglet de suivi Cocolis n'apparaissait plus du tout sur la fiche commande admin, sans erreur.
**Correctif** :
- Nouveaux hooks `hookDisplayAdminOrderTabLink()` et `hookDisplayAdminOrderTabContent()` (le vrai remplaçant PS9, confirmé via `details.html.twig` et l'implémentation actuelle de `psshipping`).
- Logique de récupération des infos de suivi factorisée dans un helper privé `getCocolisTrackingVars($order, $audience)`, réutilisé aussi par `hookDisplayOrderDetail()` (front-office, toujours actif).
- Nouveaux templates `views/templates/hook/tab_link.tpl` et `tab_content.tpl` (structure Bootstrap 4 identique au module officiel `psshipping`).
- Anciens templates `tab_ship.tpl` et `content_ship.tpl` supprimés (devenus morts).
- `install()`/`uninstall()` mis à jour : retrait des hooks morts, ajout de `displayAdminOrderTabLink` et `displayAdminOrderTabContent`.
- Bonus : la variable `module_dir` (utilisée par le template pour afficher le logo), auparavant jamais assignée pour ce hook, est maintenant correctement transmise.

## 5. `cocolis.php` — `Db::getInstance()->executes()` (méthode inexistante)

**Problème** : `hookDisplayOrderDetail()` et l'ancien `hookDisplayAdminOrderContentShip()` appelaient `->executes($sql)` — méthode qui n'existe pas dans la classe `Db` (la vraie méthode est `executeS()`, avec un S majuscule). Pas de `__call()` magique dans `Db`.
**Erreur** : `Error: Call to undefined method Db::executes()`, fatale et non interceptée. `hookDisplayOrderDetail` étant activement appelé par `OrderDetailController` et `GuestTrackingController`, ce crash se produisait dès qu'un client consultait le détail de sa commande.
**Correctif** : `executes()` → `executeS()` (2 occurrences).

## 6. `cocolis.php` — `Logger::addLog()` (classe supprimée du core PS9)

**Problème** : la classe `Logger` (PS 1.6/1.7, méthode statique `addLog()`) a été remplacée par `PrestaShopLogger` en PS9. `Logger` n'existe plus nulle part dans le core.
**Erreur** : `Error: Class "Logger" not found`, fatale — déclenchée uniquement quand `COCOLIS_DEBUG_MODE` est activé (dans `getOrderShippingCost()`, au calcul des frais de port du panier). Le mode debug, censé aider au diagnostic, cassait donc systématiquement le calcul des frais de port.
**Correctif** : `Logger::addLog()` → `PrestaShopLogger::addLog()` (2 occurrences).
**Point d'attention découvert en testant** : `PrestaShopLogger` est un logger **base de données** (table `ps_log`, consultable via *Paramètres avancés > Logs* dans le back-office) — il n'écrit pas dans les fichiers `var/logs/*.log`. Ne pas chercher ces messages dans les logs fichiers.

## 7. `cocolis.php` — configuration des webhooks : identifiants jamais sauvegardés avant l'appel API

**Problème** : le bouton "Configurer les webhooks" (`name="webhooks"`) partage le même formulaire que les champs App ID / Password, mais le handler appelait directement `authenticatedClient()` sans jamais appeler `postProcess()` — les identifiants tout juste saisis n'étaient donc jamais enregistrés avant l'appel API.
**Erreur** : `InvalidArgumentException: Key password is missing`, non interceptée → crash 500.
**Correctif** : `postProcess()` appelé en premier + garde-fou avec message utilisateur clair si App ID/Password toujours vides après sauvegarde (notification `webhook_missing_credentials`).

## 8. `cocolis.php` — champ Password systématiquement écrasé par une valeur vide

**Problème** : le champ `type="password"` du `HelperForm` PrestaShop est **toujours rendu vide** en HTML (`value=""`, par sécurité — comportement standard du core). Le correctif n°7 (`postProcess()` systématique) réenregistrait donc un mot de passe vide à chaque soumission du formulaire où l'utilisateur n'avait pas retapé son mot de passe, écrasant la valeur déjà enregistrée.
**Correctif** : `postProcess()` ignore désormais une soumission vide du champ `COCOLIS_PASSWORD` (conserve la valeur existante) — comportement standard pour ce type de champ.

## 9. `cocolis.php` — exceptions API non interceptées lors de la configuration des webhooks

**Problème** : un rejet d'authentification par l'API Cocolis (`UnauthorizedException`, HTTP 401) ou toute autre erreur API pendant la configuration des webhooks n'était pas catchée → crash 500 au lieu d'un message clair.
**Correctif** : `try/catch` sur `Cocolis\Api\Errors\UnauthorizedException` et `Cocolis\Api\Errors\CocolisException`, avec notifications dédiées (`webhook_invalid_credentials`, `webhook_api_error`) ajoutées dans `views/templates/admin/configure.tpl`.

## 10. `cocolis.php` — `uninstall()` ne supprimait pas les transporteurs créés

**Problème** : `uninstall()` ne supprimait jamais les transporteurs créés par `addCarrier()`/`addCarrierInsurance()` à l'installation — la suppression de leurs IDs en config (`COCOLIS_CARRIER_ID`/`COCOLIS_CARRIER_ASSURANCE_ID`) était même explicitement mise en commentaire dans le code d'origine.
**Conséquence** : transporteurs orphelins après désinstallation.
**Correctif** : recherche des transporteurs par `external_module_name = 'cocolis'` et `Carrier::delete()` sur chacun (gère correctement le *soft delete* si le transporteur a déjà servi dans des commandes, pour préserver l'historique).

## 11. `override/classes/form/CustomerFormatter.php` — override incompatible avec PS9 (bug le plus critique)

**Fichiers concernés** :
- `modules/cocolis/override/classes/form/CustomerFormatter.php` (source du module)
- `override/classes/form/CustomerFormatter.php` (copie fusionnée par PrestaShop à l'installation — c'est ce fichier qui est réellement exécuté)

**Problème** : l'override du module type-hinte son constructeur avec l'ancienne interface `Symfony\Component\Translation\TranslatorInterface`. Le core PS9 utilise désormais `Symfony\Contracts\Translation\TranslatorInterface` et injecte une instance de `PrestaShopBundle\Translation\TranslatorComponent`, qui n'implémente pas l'ancienne interface.
**Erreur** : `TypeError: CustomerFormatter::__construct(): Argument #1 ($translator) must be of type Symfony\Component\Translation\TranslatorInterface, PrestaShopBundle\Translation\TranslatorComponent given` — fatale, non catchable, crash 500 systématique dès qu'un client accédait à `/commande` (`OrderController::buildCheckoutProcess()` → `makeCustomerForm()`).
**Correctif** : `use Symfony\Component\Translation\TranslatorInterface;` → `use Symfony\Contracts\Translation\TranslatorInterface;` dans les deux fichiers.

## 12. Nettoyage : fichiers morts supprimés

- `modules/cocolis/override/modules/cocolis/cocolis.php` et sa copie fusionnée `override/modules/cocolis/cocolis.php` : définissaient une classe `CocolisOverride` entièrement commentée, qui ne faisait rien (mauvais nom de classe pour un override réel — n'a jamais été un override fonctionnel). Résidu de debug oublié dans le packaging du module. Supprimés.

## 12 bis. `cocolis.php` — `sizeof(false)` dans `hookActionPaymentConfirmation()` (PHP 8)

**Problème** : `Product::getCover()` renvoie `false` (via `Db::getRow`) quand un produit n'a aucune image de couverture. Le code faisait `sizeof($id_image) > 0` ; en PHP 8, `sizeof()`/`count()` sur un non-tableau lève une `TypeError` fatale.
**Conséquence** : le changement de statut d'une commande vers "paiement accepté" plantait (500 dans le back-office) pour toute commande Cocolis contenant un produit sans image de couverture, et la livraison n'était jamais créée chez Cocolis.
**Correctif** : `sizeof($id_image) > 0` → `!empty($id_image)`.

## 12 ter. Création de la livraison silencieusement perdue, historique de suivi mal indexé, webhook cassé

- **`hookActionPaymentConfirmation()`** : `Hook::exec()` avale les exceptions hors mode debug (`classes/Hook.php` (lignes 464-471 en 9.1.5)). Un échec de création de la livraison chez Cocolis (identifiants, adresse vendeur manquante, erreur 422…) n'était donc jamais visible. Le corps du hook est déplacé dans `createCocolisRide()` et appelé dans un `try/catch` qui écrit l'erreur dans *Paramètres avancés > Logs* (sévérité 3). En mode debug, un log signale aussi que le hook est bien appelé.
- **Historique de suivi** : `cocolis_order_history.order_id` est alimenté par le webhook avec l'ID de **commande** (`external_id`), mais `cocolis.php` le relisait avec `getIdOrderCarrier()` (ID de la ligne `order_carrier`). Les deux ne coïncident que par hasard, donc l'onglet/le suivi pouvait rester vide. Les requêtes utilisent maintenant `$order->id`.
- **`controllers/front/webhooks.php`** : appel à `Logger::addLog()` (classe supprimée en PS9, cf. point 6). Le webhook plantait **après** le changement de statut de la commande et **avant** l'insertion dans `cocolis_order_history` : l'historique n'était jamais enregistré et Cocolis recevait une erreur 500. Remplacé par `PrestaShopLogger::addLog()`.

- **Injection SQL (corrigée)** : ce même contrôleur, accessible publiquement, insérait `$event` et `json_encode(Tools::getAllValues())` dans une requête SQL sans échappement. L'insertion passe maintenant par `Db::insert()` avec `pSQL()` sur chaque valeur, et l'identifiant de commande est converti en entier dès la lecture de la requête.

## 12 quater. Création de la livraison refusée par l'API Cocolis (`invalid_record`)

**Erreur** (vue dans *Paramètres avancés > Logs* grâce au point 12 ter) : `content_value` « La valeur du bien doit être renseignée » et `external_price` « doit être rempli(e) ».
**Cause** : l'API Cocolis exige désormais ces deux champs ; le module ne les envoyait pas.
**Correctif** (`createCocolisRide()`, les deux tableaux de paramètres) :
- `content_value` : valeur du bien en centimes = total TTC de la commande moins les frais de port (même base que le seuil d'assurance). Champ documenté dans le SDK ; le plugin officiel WooCommerce l'envoie aussi.
- `external_price` : **hypothèse, à confirmer auprès de Cocolis**. Ce champ n'est documenté nulle part et le plugin WooCommerce ne l'envoie pas. Il est renseigné avec le prix de transport payé par le client, en centimes (même valeur que `price`).
- `price` : `(int) $order->total_shipping_tax_incl * 100` tronquait les centimes (12,50 € devenait 1200) ; remplacé par `(int) round(... * 100)`.

## 12 quinquies. Liens de suivi

- Le lien « Vous pouvez accéder à l'annonce » du détail de commande **côté client** menait à une page introuvable : l'URL publique de la course est `ride-public/<slug>-<id de la course>`, et la version front omettait l'identifiant (seule la version back-office l'ajoutait). `getCocolisTrackingVars()` construit désormais l'URL de la même façon pour les deux.
- Les liens de suivi et d'annonce (`tab_content.tpl`, `content_ship_front.tpl`) s'ouvrent dans un nouvel onglet (`target="_blank" rel="noopener noreferrer"`).

## 12 sexies. Téléphone du destinataire

`to_contact_phone` n'utilisait que `$address->phone`, vide quand la boutique ne demande que le mobile. Repli ajouté sur `$address->phone_mobile` (`createCocolisRide()`).

## 12 septies. Photos et poids des produits dans l'annonce Cocolis

- **Photos** : le module n'envoyait les URLs qu'au niveau de la course (`photo_urls`), un champ absent du modèle `ride-create` documenté. Le plugin officiel WooCommerce de Cocolis les met dans **chaque objet** (`ride_objects_attributes[].photo_urls`). Les URLs sont maintenant aussi envoyées par objet (helper `getProductImageUrl()`) ; l'envoi au niveau de la course est conservé. **Cocolis doit pouvoir télécharger l'image** : une URL en `ddev.site` ne sera jamais visible chez eux, seul un environnement public le permet.
- **Poids** : jamais envoyé jusqu'ici. Les réponses de l'API montrent un champ `weight` sous forme de classe (`less_than_5_kg`, `between_5_and_30_kg`). Le poids du produit (supposé en kg) est converti pour ces deux classes (`getCocolisWeightClass()`). Au-delà de 30 kg ou sans poids, le champ est omis, car les autres valeurs possibles ne sont pas documentées. **À confirmer auprès de Cocolis** : ce champ est-il bien accepté à la création ?
- **Taille** : le module envoie déjà largeur/hauteur/longueur par objet (dimensions du produit, ou dimensions moyennes de la config si elles sont à 0). Rien modifié.

## 13. Environnement — packaging du module (`vendor/`)

**Problème** (rencontré à la toute première installation) : le zip du module ne contenait pas `vendor/composer/platform_check.php`, `vendor/composer/installed.php`, ni le contenu de `vendor/cocolis/php/` (SDK API Cocolis). Le `require_once vendor/autoload.php` en tête de `cocolis.php` provoquait une erreur fatale avant même que la classe `Cocolis` soit définie → PrestaShop refusait l'installation ("module non valide").
**Correctif** : régénération du vendor via `composer install --no-dev --optimize-autoloader` (exclut aussi les dépendances de développement — `phpunit`, `php-cs-fixer`, `symfony/console` — qui n'ont rien à faire en production).

---

## État des tests (PrestaShop 9.1.5, API Cocolis sandbox)

| Scénario | Statut |
|---|---|
| Installation / désinstallation / réinstallation | Validé |
| Configuration App ID / mot de passe, gestion d'une erreur d'authentification | Validé |
| Transporteurs Cocolis proposés au checkout (compte sandbox, dimensions moyennes renseignées) | Validé (les correctifs 2 et le compte sandbox ont été appliqués ensemble, on ne peut pas dire lequel était décisif) |
| Page `/commande` sans erreur 500 | Validé |
| Paiement par chèque avec le transporteur Cocolis | Validé |
| Création de la livraison chez Cocolis au passage en « Paiement accepté » | Validé en sandbox (e-mails reçus, page de suivi accessible) |
| Webhook `ride_published` simulé par `curl` | Validé |
| Webhooks `offer_accepted`, `offer_completed`, `ride_expired`, `offer_cancelled` | Non testés |
| Liens de suivi et d'annonce (fiche commande admin, détail commande client) | Lien de suivi validé ; lien d'annonce corrigé (point 12 quinquies), à revérifier |
| Photos et poids dans l'annonce Cocolis | Non vérifiés (les photos exigent une boutique joignable depuis Internet) |
| Valeur de `external_price` | Hypothèse, à confirmer auprès de Cocolis |
| Désinstallation : suppression des transporteurs | Non revérifié après le dernier cycle |
| Paiement avec PrestaShop Checkout / PayPal | Non testé (nécessite un compte PrestaShop Account lié à la boutique) |
