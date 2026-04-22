# SysWatch

Projet Rust de supervision systeme en reseau.

Le depot contient deux executables :
- `syswatch` : agent a lancer sur chaque machine etudiante.
- `syswatch-master` : interface admin a lancer sur la machine maitre.

## Condition reseau importante

Pour manipuler les autres machines a partir de la machine admin, toutes les machines doivent etre connectees au meme reseau local.

Concretement :
- chaque machine etudiante lance `syswatch` et ecoute sur le port `7878`,
- la machine admin lance `syswatch-master`,
- les adresses IP configurees dans `src/master.rs` doivent correspondre aux IP reelles des machines du meme reseau,
- le pare-feu doit autoriser les connexions TCP sur le port `7878`.

## Lancer le projet

Dans un terminal a la racine du projet :

```powershell
& 'C:\Users\mpack\.cargo\bin\cargo.exe' run
```

Pour lancer l'interface admin :

```powershell
& 'C:\Users\mpack\.cargo\bin\cargo.exe' run --bin syswatch-master
```

## Correspondance avec le TP

Le projet respecte les grandes etapes demandees :
- Etape 1 : modelisation des donnees avec `CpuInfo`, `MemInfo`, `ProcessInfo` et `SystemSnapshot`.
- Etape 2 : collecte reelle des metriques avec `sysinfo` et retour en `Result`.
- Etape 3 : formatage des reponses avec `format_response`.
- Etape 4 : serveur TCP multi-thread sur le port `7878` avec `Arc<Mutex<SystemSnapshot>>`.
- Etape 5 : journalisation des connexions et commandes dans `syswatch.log`.

Le projet va un peu plus loin que le sujet en ajoutant une interface maitre dans `src/master.rs` pour piloter plusieurs machines.

## Reponses aux questions de comprehension

### Etape 1

1. `#[derive(Debug)]` permet surtout un affichage technique destine au debogage, alors que `fmt::Display` sert a produire un affichage lisible et personnalise pour l'utilisateur.
2. `Vec<ProcessInfo>` est utilise parce que le nombre de processus varie selon la machine et le moment, contrairement a un tableau de taille fixe.
3. `#[derive(Clone)]` sur `SystemSnapshot` permet de dupliquer proprement un instantane complet si on a besoin d'en conserver une copie ou de le transmettre ailleurs sans deplacer l'original.

### Etape 2

1. `collect_snapshot()` retourne un `Result` parce que la collecte systeme peut echouer et qu'il faut propager ou traiter cette erreur proprement.
2. Remplacer `.unwrap()` par `?` dans une fonction qui retourne `Result` evite de faire paniquer le programme et propage l'erreur a l'appelant.
3. `processes.sort_by(|a, b| b.cpu_usage.partial_cmp(&a.cpu_usage).unwrap())` trie les processus par CPU decroissant. On utilise `partial_cmp` parce que `f32` n'implemente pas `cmp` total a cause de cas speciaux comme `NaN`.

### Etape 3

1. `.collect::<Vec<_>>().join("")` transforme les elements produits par l'iterateur en vecteur de fragments, puis les concatene pour former une barre ASCII.
2. `cmd.as_str()` convertit la `String` en `&str`, ce qui est plus adapte au `match` sur des litteraux. `&cmd` donne une reference vers la `String` elle-meme.
3. On passe `snapshot` par reference `&SystemSnapshot` pour eviter une copie inutile et pour pouvoir le lire sans en transferer la propriete.

### Etape 4

1. `Arc<Mutex<T>>` est utilise parce qu'il faut a la fois partager la donnee entre plusieurs threads (`Arc`) et proteger les acces concurrents (`Mutex`).
2. Si un thread panique en tenant le verrou, le `Mutex` devient empoisonne. Les appels suivants a `lock()` renverront alors une erreur de poison.
3. `stream.try_clone()` permet d'avoir une poignee pour la lecture et une autre pour l'ecriture sur la meme connexion. Sans cela, on se heurterait aux regles d'emprunt de Rust.
4. La boucle infinie du thread de rafraichissement n'est pas un probleme tant que le serveur tourne. Pour l'arreter proprement, on pourrait ajouter un signal d'arret partage, par exemple un booleen atomique ou un canal.

### Etape 5

1. `OpenOptions::new().create(true).append(true)` ouvre ou cree un fichier sans ecraser son contenu et ajoute a la fin, tandis que `File::create()` recree le fichier et tronque generalement l'ancien contenu.
2. `if let Ok(mut file) = ...` est utilise pour garder une journalisation en mode best-effort : si l'ouverture echoue, le programme continue. Avec `unwrap()`, il paniquerait.
3. Oui, si plusieurs threads ecrivent en meme temps, les lignes peuvent theoriquement se chevaucher selon l'ordonnancement et le systeme. Pour garantir un log strictement serialize, il faudrait proteger l'ecriture avec un verrou partage ou un thread dedie au logging.
