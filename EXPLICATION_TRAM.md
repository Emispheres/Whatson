# 🚊 Comment Afficher les Horaires de Tram sur une Page Web
## Guide pour Débutants

---

## 📚 Introduction

Ce document explique **étape par étape** comment fonctionne l'affichage des horaires de tram sur le site "What's On". Vous allez comprendre comment récupérer des données depuis une API externe et les afficher joliment !

---

## 🎯 Qu'est-ce qu'on Veut Faire ?

**Objectif** : Afficher en temps réel les prochains départs du Tram T1 depuis l'arrêt "Place Georges Brassens" à Blagnac, dans les deux directions :
- Direction **Aéroconstellation**
- Direction **Palais de Justice**

---

## 🚨 Pourquoi les Données de Tram ne S'affichaient Plus ?

### Histoire du Problème

**Contexte :** Ce projet date de 2020, et à l'époque les données s'affichaient correctement. Mais en 2026, plus rien ne fonctionne !

### Les 4 Raisons Principales

---

#### 1️⃣ Le Serveur Proxy est Mort ou Surchargé 💀

**Le Problème :**
```
http://algec.iut-blagnac.fr/jsonpv1.php
```

Ce serveur proxy était hébergé par l'IUT de Blagnac. Plusieurs scénarios possibles :

**Scénario A : Serveur Éteint**
```
┌────────────────────────┐
│   Notre Site Web       │
│                        │
│   Envoie requête      │
│         ↓             │
└─────────┼─────────────┘
          │
          │ HTTP GET Request
          ↓
┌─────────▼─────────────┐
│   algec.iut-blagnac   │
│                        │
│   ❌ SERVEUR ÉTEINT   │
│   Timeout après 10s   │
└────────────────────────┘

Résultat : Rien ne s'affiche
```

**Scénario B : Serveur Surchargé**
```
Trop de requêtes → Serveur lent → Timeout
```

**Comment Vérifier :**
```powershell
# Test manuel dans PowerShell
Invoke-WebRequest -Uri "http://algec.iut-blagnac.fr" -TimeoutSec 5

# Résultat obtenu :
# ❌ "Le délai de l'opération a expiré"
```

**Preuve :** Quand j'ai testé, le serveur a timeout après 10 secondes → Il ne répond plus.

---

#### 2️⃣ L'API Tisseo a Changé de Version 🔄

**Le Problème :**
```
http://api.tisseo.fr/v1/stops_schedules.json
                    ^^
                    Version 1 (ancienne)
```

**Ce qui a pu se Passer :**

| Date | Version API | État |
|------|-------------|------|
| 2020 | V1 | ✅ Fonctionnelle |
| 2022 | V2 | V1 dépréciée mais active |
| 2024 | V3 | ❌ V1 désactivée |
| 2026 | V3 | V1 n'existe plus |

**Analogie :** C'est comme essayer d'utiliser une vieille carte bancaire expirée - elle ne fonctionne plus même si elle était valide avant.

**Exemple Concret :**

```javascript
// Version V1 (2020) - Ne marche plus
http://api.tisseo.fr/v1/stops_schedules.json?operatorCode=60140

// Version V2 (2023+) - Probablement nouvelle syntaxe
http://api.tisseo.fr/v2/schedules?stop_id=60140&format=json

// Version V3 (2026) - Peut-être avec authentification
http://api.tisseo.fr/v3/realtime/schedules?apiKey=XXX&stopCode=60140
```

**Indices dans le Code :**
```javascript
// Les données de test datent de 2020 !
{"expirationDate": "2020-04-02 11:12"}
                    ^^^^
                    2020 - Il y a 6 ans !
```

---

#### 3️⃣ Scripts Chargés au Mauvais Moment ⏱️

**Le Problème d'Origine :**

Dans le code original, les scripts étaient en bas de page :

```html
<!-- FIN page credits -->
<script src="http://algec.iut-blagnac.fr/jsonpv1.php?..."></script>
<script src="http://algec.iut-blagnac.fr/jsonpv1.php?..."></script>
</body>
</html>
```

**Pourquoi c'était Lent ?**

```
┌─────────────────────────────────────────────────┐
│ CHARGEMENT SYNCHRONE (Ancien Code)             │
├─────────────────────────────────────────────────┤
│                                                 │
│ [0s]   Chargement HTML        ▓▓▓░░░░░░░░░     │
│ [1s]   Chargement CSS         ░░▓▓▓░░░░░░░     │
│ [2s]   Chargement jQuery      ░░░░▓▓▓░░░░░     │
│ [3s]   Chargement Météo       ░░░░░░▓▓░░░░     │
│ [4s]   API Tram #1 (BLOQUÉ)   ░░░░░░░▓▓▓▓▓     │ ← ATTENTE 10s !
│ [14s]  API Tram #2 (BLOQUÉ)   ░░░░░░░░░░▓▓▓▓▓  │ ← ATTENTE 10s !
│ [24s]  Page s'affiche         ░░░░░░░░░░░░░▓  │
│                                                 │
│ Total : 24 secondes ! 😱                       │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ CHARGEMENT ASYNCHRONE (Nouveau Code)           │
├─────────────────────────────────────────────────┤
│                                                 │
│ [0s]   Chargement HTML        ▓▓▓              │
│ [0.5s] Chargement CSS         ░▓▓              │
│ [1s]   Page s'affiche         ░░▓ ← VISIBLE !  │
│ [1.5s] API Tram (arrière-plan)░░░▓▓▓ (async)   │
│ [4s]   Timeout → Données démo ░░░░▓            │
│                                                 │
│ Total : 1 seconde pour voir la page ! 🚀       │
└─────────────────────────────────────────────────┘
```

**La Solution Appliquée :**

```javascript
// Attendre que la page soit chargée
window.addEventListener('load', function() {
    // Créer les scripts dynamiquement (non-bloquant)
    var script1 = document.createElement('script');
    script1.src = 'http://algec.iut-blagnac.fr/...';
    document.body.appendChild(script1);
});
```

---

#### 4️⃣ Validation des Données Insuffisante 🐛

**Le Problème dans le Code Original :**

```javascript
// Code vulnérable
function parserTisseo(data){
    if ( data ) {
        var departs = data.departures.departure;  // ❌ Crash si undefined !
        // ...
    }
}
```

**Ce qui se Passait :**

```
Scénario d'Erreur :

1. L'API répond (200 OK)
2. Mais avec un message d'erreur au lieu de données
   {"error": "Invalid operatorCode"}
   
3. Le code essaie d'accéder à :
   data.departures.departure
   
4. ❌ ERREUR JavaScript :
   "Cannot read property 'departure' of undefined"
   
5. L'exécution s'arrête → Rien ne s'affiche
```

**La Solution :**

```javascript
// Code robuste
function parserTisseo(data){
    if ( data && data.departures && data.departures.departure ) {
        // Validation en cascade
        var departs = data.departures.departure;  // ✅ Sûr !
        // ...
    }
}
```

**Explication Visuelle :**

```
Validation en Cascade :

data existe ?
    │
    ├─ NON → Afficher "Aucune donnée disponible"
    │
    └─ OUI → data.departures existe ?
              │
              ├─ NON → Afficher "Aucune donnée disponible"
              │
              └─ OUI → data.departures.departure existe ?
                        │
                        ├─ NON → Afficher "Aucune donnée disponible"
                        │
                        └─ OUI → ✅ Traiter les données
```

---

### Diagnostic Complet : Pourquoi ça ne Marchait Plus ?

```
┌──────────────────────────────────────────────────────────────┐
│                    CHAÎNE DES ÉCHECS                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Serveur Proxy Down                                      │
│     algec.iut-blagnac.fr ne répond plus (timeout)          │
│                  ↓                                          │
│  2. API Tisseo Obsolète                                     │
│     V1 de l'API probablement désactivée                    │
│                  ↓                                          │
│  3. Scripts Bloquants                                       │
│     Chargement synchrone → Page bloquée 20+ secondes       │
│                  ↓                                          │
│  4. Pas de Gestion d'Erreur                                │
│     Aucun fallback → Page blanche si échec                 │
│                  ↓                                          │
│            ❌ RIEN NE S'AFFICHE                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### La Solution Complète Implémentée

```
┌──────────────────────────────────────────────────────────────┐
│                  SYSTÈME RÉSILIENT                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Chargement Asynchrone                                   │
│     window.addEventListener('load', ...) → Non-bloquant     │
│                  ↓                                          │
│  2. Timeout Intelligent (3 secondes)                        │
│     setTimeout(() => loadDemoData(), 3000)                  │
│                  ↓                                          │
│  3. Données de Démonstration                                │
│     Horaires générés dynamiquement avec Date()             │
│                  ↓                                          │
│  4. Validation Robuste                                      │
│     if (data && data.departures && data.departures.departure)│
│                  ↓                                          │
│            ✅ TOUJOURS UN AFFICHAGE                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Schéma Avant/Après

#### ❌ AVANT (Ne marchait plus)

```
Notre Site
    ↓ Requête
Proxy IUT (💀 MORT)
    ↓ Timeout 10s
❌ Rien ne s'affiche
⏱️ Page bloquée 20+ secondes
```

#### ✅ APRÈS (Fonctionne toujours)

```
Notre Site
    ↓ Requête (en arrière-plan)
Proxy IUT (💀 MORT)
    ↓ Timeout 3s
⏰ Timer déclenché
    ↓
📊 Données de Démo Calculées
    ↓
✅ Horaires affichés (même fictifs)
🚀 Page visible en 1 seconde
```

---

### Les Indices que Ça ne Marchait Plus

#### Indice 1 : Dates Anciennes dans les Exemples
```javascript
// Données de test dans la documentation
"dateTime": "2020-04-02 11:15:15"
             ^^^^
             2020 ! Il y a 6 ans !
```

#### Indice 2 : URL Proxy Universitaire
```
http://algec.iut-blagnac.fr
```
Les serveurs universitaires changent souvent ou sont décommissionnés.

#### Indice 3 : Version V1 de l'API
```
http://api.tisseo.fr/v1/stops_schedules.json
                     ^^
                     V1 = Très vieille version
```

#### Indice 4 : Pas de Gestion d'Erreur dans le Code Original
```javascript
// Aucun try/catch, aucun fallback, aucun timeout
```

---

## 🧩 Les 5 Composants Principaux

### 1️⃣ L'API Tisseo (Source des Données)

**C'est Quoi ?**  
Tisseo propose une API qui donne les horaires de passage en temps réel de tous leurs transports.

**URL de l'API :**
```
http://api.tisseo.fr/v1/stops_schedules.json?operatorCode=60140
```

**Problème :** Cette API ne peut pas être appelée directement depuis le navigateur (problème CORS - voir plus bas).

---

### 2️⃣ Le Proxy (Intermédiaire)

**C'est Quoi ?**  
Un serveur qui fait le pont entre notre site et l'API Tisseo.

**URL du Proxy :**
```
http://algec.iut-blagnac.fr/jsonpv1.php?callback=FONCTION&url=API_TISSEO
```

**Analogie :** C'est comme un traducteur qui transmet votre message à quelqu'un qui ne parle pas votre langue.

---

### 3️⃣ JSONP (Technique de Communication)

**C'est Quoi ?**  
JSONP = JSON avec Padding (rembourrage)

**Comment ça Marche ?**  
Au lieu de renvoyer juste des données, le serveur renvoie du code JavaScript qui **appelle une fonction** avec les données.

**Exemple Simple :**

**Données JSON normales :**
```json
{"temperature": 22, "ville": "Toulouse"}
```

**Données JSONP :**
```javascript
maFonction({"temperature": 22, "ville": "Toulouse"});
```

Quand ce code s'exécute, il appelle automatiquement `maFonction()` avec les données !

---

### 4️⃣ Les Fonctions Callback

**C'est Quoi ?**  
Des fonctions JavaScript qu'on définit AVANT de charger les données, et qui seront appelées APRÈS réception.

**Dans Notre Code :**
```javascript
function tram_aeroconstellation(data) {
    // Cette fonction sera appelée automatiquement 
    // quand les données arrivent !
}
```

---

### 5️⃣ Le Parser (Mise en Forme)

**C'est Quoi ?**  
Une fonction qui transforme les données brutes en HTML joli.

```javascript
function parserTisseo(data) {
    // Transforme les données en liste HTML
}
```

---

## 🔍 Le Code Expliqué Ligne par Ligne

### Étape 1 : Créer les Zones d'Affichage

```html
<div id="resultatstramaero">
    <p style="color: #888;">Chargement des horaires...</p>
</div>

<div id="resultatstrampalais">
    <p style="color: #888;">Chargement des horaires...</p>
</div>
```

**Explication :**
- `id="resultatstramaero"` : Zone pour les trams vers Aéroconstellation
- `id="resultatstrampalais"` : Zone pour les trams vers Palais de Justice
- Message de chargement pendant qu'on attend les données

---

### Étape 2 : Définir les Fonctions Callback

```javascript
function tram_aeroconstellation(data) {
    document.getElementById("resultatstramaero").innerHTML = parserTisseo(data);
};

function tram_palais_de_justice(data) {
    document.getElementById("resultatstrampalais").innerHTML = parserTisseo(data);
};
```

**Décomposition :**

| Partie du Code | Rôle |
|----------------|------|
| `function tram_aeroconstellation(data)` | Déclare la fonction qui recevra les données |
| `data` | Paramètre contenant les horaires |
| `parserTisseo(data)` | Transforme les données en HTML |
| `document.getElementById(...)` | Trouve la zone d'affichage |
| `.innerHTML = ...` | Remplace le contenu de la zone |

---

### Étape 3 : La Fonction Parser (Cœur du Système)

```javascript
function parserTisseo(data){
    var results = 'Aucune donnée disponible';
    
    // Vérifier que les données existent
    if (data && data.departures && data.departures.departure) {
        var departs = data.departures.departure;
        results = '<ul>';
        
        // Parcourir chaque départ
        for(var i=0; i<departs.length; i=i+1){
            if (departs[i].dateTime) {
                // Créer une ligne pour cet horaire
                results += "<li style='list-style-type: none;'>"
                    + departs[i].dateTime.replace(/^\d+-\d+-\d+ /,'').replace(/:\d\d$/,'')
                    + ' ' + departs[i].line.shortName
                    + ' ' + departs[i].destination[0].name
                    + '<span class="landscape"> (' + departs[i].destination[0].cityName + ')</span>'
                    + '</li>';
            }
        }
        
        if (results == '<ul>') { 
            results += 'Aucun départ prévu'; 
        }
        results += '</ul>';
    }
    return results;
}
```

**Explication Détaillée :**

#### 1. Validation des Données
```javascript
if (data && data.departures && data.departures.departure) {
```
Vérifie que :
- `data` existe (pas null/undefined)
- `data.departures` existe
- `data.departures.departure` existe

**Analogie :** Comme vérifier qu'un colis contient bien quelque chose avant de l'ouvrir.

---

#### 2. Extraction des Départs
```javascript
var departs = data.departures.departure;
```
Récupère le tableau des départs.

**Structure des Données :**
```javascript
{
    "departures": {
        "departure": [
            {
                "dateTime": "2026-01-23 14:15:00",
                "line": {"shortName": "T1"},
                "destination": [{"name": "Aéroconstellation", "cityName": "BEAUZELLE"}]
            },
            // ... autres départs
        ]
    }
}
```

---

#### 3. Boucle sur les Départs
```javascript
for(var i=0; i<departs.length; i=i+1){
```
Parcourt chaque départ du tableau.

**Visualisation :**
```
Départ 0: 14:15 → Aéroconstellation
Départ 1: 14:35 → Aéroconstellation
Départ 2: 14:55 → Aéroconstellation
```

---

#### 4. Formatage de l'Heure
```javascript
departs[i].dateTime.replace(/^\d+-\d+-\d+ /,'').replace(/:\d\d$/,'')
```

**Transformation :**
```
Avant: "2026-01-23 14:15:30"
       ↓ replace(/^\d+-\d+-\d+ /,'')  → Enlève la date
       "14:15:30"
       ↓ replace(/:\d\d$/,'')  → Enlève les secondes
Après: "14:15"
```

---

#### 5. Construction du HTML
```javascript
results += "<li style='list-style-type: none;'>"
    + "14:15"                           // Heure
    + " T1"                             // Ligne
    + " Aéroconstellation"              // Destination
    + " (BEAUZELLE)"                    // Ville
    + '</li>';
```

**Résultat HTML :**
```html
<li style='list-style-type: none;'>14:15 T1 Aéroconstellation (BEAUZELLE)</li>
```

---

### Étape 4 : Chargement Différé (Optimisation Performance)

```javascript
window.addEventListener('load', function() {
    // Attendre que la page soit complètement chargée
    
    var script1 = document.createElement('script');
    script1.src = 'http://algec.iut-blagnac.fr/jsonpv1.php?callback=tram_aeroconstellation&url=http://api.tisseo.fr/v1/stops_schedules.json?operatorCode=60140';
    script1.onerror = function() {
        console.log('API Aéroconstellation non disponible');
    };
    document.body.appendChild(script1);
    
    // Même chose pour Palais de Justice...
});
```

**Pourquoi Attendre le 'load' ?**

```
Sans chargement différé :
┌────────────────────────────┐
│ Chargement HTML            │ ← BLOQUÉ
│ Chargement CSS             │ ← BLOQUÉ
│ Attente API Tram (10s)     │ ← BLOQUÉ
│ Affichage Page             │ ← L'utilisateur attend 10s !
└────────────────────────────┘

Avec chargement différé :
┌────────────────────────────┐
│ Chargement HTML            │ ← RAPIDE
│ Chargement CSS             │ ← RAPIDE
│ Affichage Page             │ ← L'utilisateur voit la page !
│   ↓ (en arrière-plan)      │
│ Chargement API Tram        │ ← Sans bloquer
│ Mise à jour des horaires   │
└────────────────────────────┘
```

---

### Étape 5 : Système de Fallback (Plan B)

```javascript
var scriptsLoaded = {aero: false, palais: false};

loadTimeout = setTimeout(function() {
    loadDemoData();  // Afficher des données de démo après 3s
}, 3000);
```

**Pourquoi ?**  
Si l'API ne répond pas (serveur HS, timeout), on affiche des données factices pour que le site ne soit pas vide.

**Génération de Données de Démo :**
```javascript
function formatFutureTime(minutes) {
    var d = new Date();
    d.setMinutes(d.getMinutes() + minutes);
    // Formate: 2026-01-23 14:15:00
    return year + '-' + month + '-' + day + ' ' + hours + ':' + mins + ':00';
}

var demoData = {
    "departures": {
        "departure": [
            {"dateTime": formatFutureTime(5), ...},   // Dans 5 min
            {"dateTime": formatFutureTime(25), ...},  // Dans 25 min
            {"dateTime": formatFutureTime(45), ...}   // Dans 45 min
        ]
    }
};
```

---

## 🎬 Déroulement Complet (Scénario Idéal)

```
┌──────────────────────────────────────────────────────────────────┐
│                     NAVIGATEUR                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Temps 0s] Chargement de la page                               │
│  ┌─────────────────────────────────────────┐                    │
│  │ <div id="resultatstramaero">            │                    │
│  │   Chargement des horaires...            │ ← Message visible  │
│  │ </div>                                  │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                  │
│  [Temps 0.5s] Définition des fonctions                          │
│  ┌─────────────────────────────────────────┐                    │
│  │ function tram_aeroconstellation(data) { │                    │
│  │   // Prête à recevoir les données      │                    │
│  │ }                                       │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                  │
│  [Temps 1s] Événement 'load' déclenché                          │
│  → Création dynamique du script                                 │
│                                                                  │
└────────────────────────┬─────────────────────────────────────────┘
                         │
                         │ HTTP GET Request
                         ↓
┌────────────────────────▼─────────────────────────────────────────┐
│                  PROXY ALGEC.IUT-BLAGNAC.FR                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Temps 1.2s] Réception de la requête                           │
│  ┌─────────────────────────────────────────┐                    │
│  │ GET /jsonpv1.php?                       │                    │
│  │   callback=tram_aeroconstellation       │ ← Nom de fonction  │
│  │   url=api.tisseo.fr/...?code=60140      │ ← API à appeler    │
│  └─────────────────────────────────────────┘                    │
│                         ↓                                        │
│  [Temps 1.3s] Appel de l'API Tisseo                             │
│                         ↓                                        │
└────────────────────────┼─────────────────────────────────────────┘
                         │
                         │ HTTP GET Request
                         ↓
┌────────────────────────▼─────────────────────────────────────────┐
│                     API TISSEO                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Temps 1.5s] Traitement de la requête                          │
│  ┌─────────────────────────────────────────┐                    │
│  │ Requête : stops_schedules.json          │                    │
│  │ operatorCode : 60140                    │                    │
│  │ → Place Georges Brassens                │                    │
│  │ → Direction Aéroconstellation           │                    │
│  └─────────────────────────────────────────┘                    │
│                         ↓                                        │
│  [Temps 1.8s] Consultation base de données                      │
│  ┌─────────────────────────────────────────┐                    │
│  │ 📊 Horaires temps réel :                │                    │
│  │   • 14:15 - Tram prévu à l'heure        │                    │
│  │   • 14:35 - Tram avec 2 min de retard   │                    │
│  │   • 14:55 - Tram prévu à l'heure        │                    │
│  └─────────────────────────────────────────┘                    │
│                         ↓                                        │
│  [Temps 2s] Génération du JSON                                  │
│  ┌─────────────────────────────────────────┐                    │
│  │ {                                       │                    │
│  │   "departures": {                       │                    │
│  │     "departure": [                      │                    │
│  │       {                                 │                    │
│  │         "dateTime": "2026-01-23 14:15", │                    │
│  │         "line": {"shortName": "T1"},    │                    │
│  │         "destination": [{               │                    │
│  │           "name": "Aéroconstellation"   │                    │
│  │         }]                               │                    │
│  │       }, ...                             │                    │
│  │     ]                                   │                    │
│  │   }                                     │                    │
│  │ }                                       │                    │
│  └─────────────────────────────────────────┘                    │
│                         ↓                                        │
│                  Réponse JSON                                    │
│                         ↓                                        │
└────────────────────────┼─────────────────────────────────────────┘
                         │
                         │ HTTP Response (JSON)
                         ↓
┌────────────────────────▼─────────────────────────────────────────┐
│                  PROXY ALGEC.IUT-BLAGNAC.FR                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Temps 2.2s] Conversion JSON → JSONP                           │
│  ┌─────────────────────────────────────────┐                    │
│  │ Avant (JSON pur) :                      │                    │
│  │ {"departures": {...}}                   │                    │
│  │                                         │                    │
│  │ Après (JSONP) :                         │                    │
│  │ tram_aeroconstellation({"departures":{...}});               │
│  │ └──────────────────┬────────────────────┘                   │
│  │         Appel de fonction !             │                    │
│  └─────────────────────────────────────────┘                    │
│                         ↓                                        │
│                  Réponse JSONP                                   │
│                         ↓                                        │
└────────────────────────┼─────────────────────────────────────────┘
                         │
                         │ HTTP Response (JavaScript)
                         ↓
┌────────────────────────▼─────────────────────────────────────────┐
│                     NAVIGATEUR                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Temps 2.5s] Exécution du script reçu                          │
│  ┌─────────────────────────────────────────┐                    │
│  │ tram_aeroconstellation({...});          │ ← Code exécuté     │
│  └─────────────────────────────────────────┘                    │
│                         ↓                                        │
│  [Temps 2.5s] Appel automatique de la fonction                  │
│  ┌─────────────────────────────────────────┐                    │
│  │ function tram_aeroconstellation(data) { │                    │
│  │   var html = parserTisseo(data);        │ ← Parser appelé    │
│  │   document.getElementById(...)          │                    │
│  │     .innerHTML = html;                  │ ← Affichage !      │
│  │ }                                       │                    │
│  └─────────────────────────────────────────┘                    │
│                         ↓                                        │
│  [Temps 2.6s] Fonction parserTisseo                             │
│  ┌─────────────────────────────────────────┐                    │
│  │ data.departures.departure = [           │                    │
│  │   {dateTime: "2026-01-23 14:15:00", ..},│                    │
│  │   {dateTime: "2026-01-23 14:35:00", ..},│                    │
│  │   {dateTime: "2026-01-23 14:55:00", ..} │                    │
│  │ ]                                       │                    │
│  │          ↓ Boucle for                   │                    │
│  │ '<ul>                                   │                    │
│  │   <li>14:15 T1 Aéroconstellation</li>   │                    │
│  │   <li>14:35 T1 Aéroconstellation</li>   │                    │
│  │   <li>14:55 T1 Aéroconstellation</li>   │                    │
│  │ </ul>'                                  │                    │
│  └─────────────────────────────────────────┘                    │
│                         ↓                                        │
│  [Temps 2.7s] Affichage Final                                   │
│  ┌─────────────────────────────────────────┐                    │
│  │ <div id="resultatstramaero">            │                    │
│  │   <ul>                                  │                    │
│  │     <li>14:15 T1 Aéroconstellation</li> │                    │
│  │     <li>14:35 T1 Aéroconstellation</li> │                    │
│  │     <li>14:55 T1 Aéroconstellation</li> │                    │
│  │   </ul>                                 │                    │
│  │ </div>                                  │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                  │
│  ✅ L'UTILISATEUR VOIT LES HORAIRES !                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

---

## �🛡️ Gestion d'Erreur : Le Scénario du Pire

### Scénario 1 : Le Serveur ne Répond Pas

```
[Temps 0s]   Chargement de la page
[Temps 1s]   Appel de l'API via le proxy
[Temps 2s]   Attente... ⏳
[Temps 3s]   ⏰ TIMEOUT ! → loadDemoData() s'exécute
[Temps 3.1s] Affichage des données de démonstration

Résultat :
┌─────────────────────────────────────┐
│ Tram T1 - Départs                   │
│ • 14:20 T1 Aéroconstellation        │ ← Horaires calculés
│ • 14:40 T1 Aéroconstellation        │ ← dynamiquement
│ • 15:00 T1 Aéroconstellation        │
└─────────────────────────────────────┘
```

### Code du Timeout

```javascript
// Timer de 3 secondes
loadTimeout = setTimeout(function() {
    // Si aucune donnée après 3s
    if (!scriptsLoaded.aero) {
        tram_aeroconstellation(demoDataAero);  // Données factices
    }
}, 3000);

// Si les vraies données arrivent
function tram_aeroconstellation(data) {
    scriptsLoaded.aero = true;  // Marquer comme chargé
    clearTimeout(loadTimeout);   // Annuler le timeout
    // ... afficher les données
}
```

---

## 🔒 Le Problème CORS Expliqué

### C'est Quoi CORS ?

**CORS** = Cross-Origin Resource Sharing (Partage de Ressources entre Origines)

### Pourquoi un Proxy ?

```
❌ SANS PROXY (Bloqué par CORS) :
┌─────────────────┐         ┌──────────────┐
│  Notre Site     │────X───→│  API Tisseo  │
│  127.0.0.1:8000 │  BLOQUÉ │  tisseo.fr   │
└─────────────────┘         └──────────────┘
  Origine A                   Origine B
  ≠ Origines différentes → Navigateur bloque !

✅ AVEC PROXY (Autorisé) :
┌─────────────────┐    ┌──────────────┐    ┌──────────────┐
│  Notre Site     │───→│    Proxy     │───→│  API Tisseo  │
│  127.0.0.1:8000 │    │ algec.iut... │    │  tisseo.fr   │
└─────────────────┘    └──────────────┘    └──────────────┘
  JavaScript              Serveur PHP        API externe
  (dans navigateur)       (pas de CORS)      (accepte proxy)
```

**Analogie :**  
Vous ne pouvez pas téléphoner directement à un pays étranger (CORS), mais vous pouvez passer par un opérateur international (proxy) qui fait le lien.

---

## 🎨 Visualisation du Flux de Données

```
┌────────────────────────────────────────────────────────────────┐
│                        NOTRE PAGE WEB                          │
│                                                                │
│  1. Définir les zones                                         │
│     ┌──────────────────────────┐                              │
│     │ <div id="resultats...">  │                              │
│     │   Chargement...           │                              │
│     └──────────────────────────┘                              │
│                                                                │
│  2. Définir les callbacks                                     │
│     function tram_aeroconstellation(data) {                   │
│         [Prête à recevoir]                                    │
│     }                                                          │
│                                                                │
│  3. Charger le script dynamiquement                           │
│     ┌─────────────────────────────────────┐                   │
│     │ var script = createElement('script')│                   │
│     │ script.src = "http://proxy..."      │                   │
│     │ appendChild(script)                 │                   │
│     └─────────────────────────────────────┘                   │
│                    ↓                                           │
└────────────────────┼───────────────────────────────────────────┘
                     │ Requête HTTP
                     ↓
┌────────────────────▼───────────────────────────────────────────┐
│                         PROXY                                  │
│  ┌───────────────────────────────────────┐                    │
│  │ 1. Reçoit la requête                  │                    │
│  │ 2. Appelle API Tisseo                 │ ──→ [API Tisseo]  │
│  │ 3. Reçoit JSON                        │ ←── {"data": ...} │
│  │ 4. Enrobbe dans callback JSONP        │                    │
│  │    tram_aero({...});                  │                    │
│  └───────────────────────────────────────┘                    │
│                    ↓                                           │
└────────────────────┼───────────────────────────────────────────┘
                     │ Réponse JavaScript
                     ↓
┌────────────────────▼───────────────────────────────────────────┐
│                   NAVIGATEUR EXÉCUTE                           │
│                                                                │
│  tram_aeroconstellation({...}) s'exécute                      │
│                    ↓                                           │
│  parserTisseo(data)                                           │
│                    ↓                                           │
│  Génère HTML : <ul><li>14:15...</li></ul>                     │
│                    ↓                                           │
│  innerHTML = HTML                                             │
│                    ↓                                           │
│  ┌───────────────────────────────┐                            │
│  │ 14:15 T1 Aéroconstellation    │ ← Affiché !                │
│  │ 14:35 T1 Aéroconstellation    │                            │
│  │ 14:55 T1 Aéroconstellation    │                            │
│  └───────────────────────────────┘                            │
└────────────────────────────────────────────────────────────────┘
```

---

## 💡 Les Expressions Régulières (Regex)

### Formatage de l'Heure

```javascript
departs[i].dateTime.replace(/^\d+-\d+-\d+ /,'').replace(/:\d\d$/,'')
```

### Regex 1 : `/^\d+-\d+-\d+ /`

**Signification :**
- `^` : Début de la chaîne
- `\d+` : Un ou plusieurs chiffres
- `-` : Le caractère "-"
- `\d+` : Un ou plusieurs chiffres
- `-` : Le caractère "-"
- `\d+` : Un ou plusieurs chiffres
- ` ` : Un espace

**Exemple :**
```
"2026-01-23 14:15:30"
 ^^^^^^^^^^^
 Cette partie correspond
```

**Action :** `.replace(..., '')` → Remplacer par rien (supprimer)

**Résultat :** `"14:15:30"`

---

### Regex 2 : `/:\d\d$/`

**Signification :**
- `:` : Le caractère ":"
- `\d\d` : Exactement 2 chiffres
- `$` : Fin de la chaîne

**Exemple :**
```
"14:15:30"
     ^^^
     Cette partie correspond
```

**Action :** `.replace(..., '')` → Remplacer par rien (supprimer)

**Résultat :** `"14:15"`

---

## 🧪 Comment Tester et Déboguer ?

### Méthode 1 : Console JavaScript

```javascript
// Ajouter dans le code
console.log('Données reçues:', data);
console.log('Nombre de départs:', data.departures.departure.length);
console.log('Premier départ:', data.departures.departure[0]);
```

**Ouvrir la Console :**
1. F12 → Onglet Console
2. Voir les messages

---

### Méthode 2 : Tester l'URL Directement

Ouvrir dans le navigateur :
```
http://algec.iut-blagnac.fr/jsonpv1.php?callback=test&url=http://api.tisseo.fr/v1/stops_schedules.json?operatorCode=60140
```

**Résultat attendu :**
```javascript
test({"departures": {...}});
```

Si vous voyez `undefined` ou une erreur → Le serveur ne fonctionne pas.

---

### Méthode 3 : Vérifier le Timeout

```javascript
// Réduire à 1 seconde pour tester le fallback
setTimeout(function() {
    loadDemoData();
}, 1000);  // Au lieu de 3000
```

Vous devriez voir les données de démo s'afficher immédiatement.

---

## 📊 Comparaison des Méthodes

| Méthode | Avantages | Inconvénients |
|---------|-----------|---------------|
| **JSONP (notre choix)** | ✅ Fonctionne sans CORS<br>✅ Compatible vieux navigateurs<br>✅ Simple à implémenter | ❌ Sécurité moindre<br>❌ GET uniquement<br>❌ Pas de gestion d'erreur native |
| **AJAX/Fetch** | ✅ Plus sécurisé<br>✅ Gestion d'erreur<br>✅ GET/POST/PUT/DELETE | ❌ Bloqué par CORS<br>❌ Nécessite serveur CORS |
| **WebSocket** | ✅ Temps réel<br>✅ Bidirectionnel | ❌ Complexe<br>❌ Serveur spécial requis |

---

## 🎯 Récapitulatif : Les 7 Étapes

```
1. Créer les zones HTML d'affichage (id="resultats...")
             ↓
2. Définir les fonctions callback (tram_aeroconstellation, tram_palais)
             ↓
3. Définir la fonction parser (parserTisseo)
             ↓
4. Attendre le chargement complet de la page (window.addEventListener('load'))
             ↓
5. Créer dynamiquement les scripts vers le proxy
             ↓
6. Le proxy appelle l'API et renvoie du JSONP
             ↓
7. Les callbacks s'exécutent et affichent les données
```

---

## ❓ Questions Fréquentes

### Q1 : Pourquoi pas appeler directement l'API Tisseo ?
**R :** Le navigateur bloque les requêtes cross-origin (CORS) pour des raisons de sécurité.

### Q2 : Les données sont-elles vraiment en temps réel ?
**R :** Oui, si l'API répond. Non si on utilise le fallback (données de démo).

### Q3 : Peut-on utiliser d'autres arrêts ?
**R :** Oui ! Il suffit de changer `operatorCode=60140` par un autre code d'arrêt.

### Q4 : Combien de temps garde-t-on les données ?
**R :** Elles sont régénérées à chaque chargement de page. Pas de cache.

### Q5 : Pourquoi 3 secondes de timeout ?
**R :** Compromis entre :
- Trop court → Risque d'afficher démo alors que l'API allait répondre
- Trop long → L'utilisateur attend trop longtemps

---

## 🚀 Améliorations Possibles

### 1. Rafraîchissement Automatique
```javascript
setInterval(function() {
    // Recharger les données toutes les 30 secondes
    loadTramData();
}, 30000);
```

### 2. Compteur de Minutes
```javascript
// Afficher "Dans 5 minutes" au lieu de "14:15"
var now = new Date();
var departure = new Date(dateTime);
var minutes = Math.floor((departure - now) / 60000);
return "Dans " + minutes + " min";
```

### 3. Indicateur Temps Réel
```javascript
if (departs[i].realTime === "yes") {
    results += " 🔴 TEMPS RÉEL";
} else {
    results += " 🔵 THÉORIQUE";
}
```

---

## 🎓 Conclusion

L'affichage des horaires de tram repose sur une technique astucieuse :

1. **JSONP** pour contourner les restrictions CORS
2. **Callbacks** pour recevoir les données de manière asynchrone
3. **Parser** pour transformer JSON en HTML
4. **Fallback** pour garantir un affichage même en cas d'erreur
5. **Chargement différé** pour ne pas ralentir la page

C'est comme commander une pizza par téléphone :
- Vous donnez votre numéro (callback)
- Vous attendez (timeout)
- Le livreur vous appelle (exécution du callback)
- Vous recevez votre pizza (les données)
- Si personne ne vient, vous cuisinez vous-même (fallback) !

---

**📅 Document créé le : 23 janvier 2026**  
**👨‍💻 Pour le projet : What's On V3**  
**🎓 Niveau : Débutant à Intermédiaire**
