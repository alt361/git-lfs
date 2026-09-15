# Git LFS Specification

This is a general guide for Git LFS clients.  Typically it should be
implemented by a command line `git-lfs` tool, but the details may be useful
for other tools.

## The Pointer

The core Git LFS idea is that instead of writing large blobs to a Git repository,
only a pointer file is written.

* Pointer files are text files which MUST contain only UTF-8 characters.
* Each line MUST be of the format `{key} {value}\n` (trailing unix newline).
* Only a single space character between `{key}` and `{value}`.
* Keys MUST only use the characters `[a-z] [0-9] . -`.
* The first key is _always_ `version`.
* Lines of key/value pairs MUST be sorted alphabetically in ascending order
(with the exception of `version`, which is always first).
* Values MUST NOT contain return or newline characters.
* Pointer files MUST be stored in Git with their executable bit matching that
of the replaced file.
* Pointer files must be less than 1024 bytes in size, including any pointer
  extension lines.
* Pointer files are unique: that is, there is exactly one valid encoding for a
  pointer file.

An empty file is the pointer for an empty file. That is, empty files are
passed through LFS without any change.

The required keys are:

* `version` is a URL that identifies the pointer file spec.  Parsers MUST use
simple string comparison on the version, without any URL parsing or
normalization.  It is case sensitive, and %-encoding is discouraged.
* `oid` tracks the unique object id for the file, prefixed by its hashing
method: `{hash-method}:{hash}`.  Currently, only `sha256` is supported.  The
hash is lower case hexadecimal.
* `size` is in bytes.

Example of a v1 text pointer:

```
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
size 12345
(ending \n)
```

Blobs created with the pre-release version of the tool generated files with
a different version URL.  Git LFS can read these files, but writes them using
the version URL above.

```
version https://hawser.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
size 12345
(ending \n)
```

For testing compliance of any tool generating its own pointer files, the
reference is this official Git LFS tool:

**NOTE:** exact pointer command behavior TBD!

* Tools that parse and regenerate pointer files MUST preserve keys that they
don't know or care about.
* Run the `pointer` command to generate a pointer file for the given local
file:

    ```
    $ git lfs pointer --file=path/to/file
    Git LFS pointer for path/to/file:

    version https://git-lfs.github.com/spec/v1
    oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345
    ```

* Run `pointer` to compare the blob OID of a pointer file built by Git LFS with
a pointer built by another tool.

  * Write the other implementation's pointer to "other/pointer/file":

    ```
    $ git lfs pointer --file=path/to/file --pointer=other/pointer/file
    Git LFS pointer for path/to/file:

    version https://git-lfs.github.com/spec/v1
    oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345

    Blob OID: 60c8d8ab2adcf57a391163a7eeb0cdb8bf348e44

    Pointer from other/pointer/file
    version https://git-lfs.github.com/spec/v1
    oid sha256 4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345

    Blob OID: 08e593eeaa1b6032e971684825b4b60517e0638d

    Pointers do not match
    ```

  * It can also read STDIN to get the other implementation's pointer:

    ```
    $ cat other/pointer/file | git lfs pointer --file=path/to/file --stdin
    Git LFS pointer for path/to/file:

    version https://git-lfs.github.com/spec/v1
    oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345

    Blob OID: 60c8d8ab2adcf57a391163a7eeb0cdb8bf348e44

    Pointer from STDIN
    version https://git-lfs.github.com/spec/v1
    oid sha256 4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345

    Blob OID: 08e593eeaa1b6032e971684825b4b60517e0638d

    Pointers do not match
    ```

## Intercepting Git

Git LFS uses the `clean` and `smudge` filters to decide which files use it.  The
global filters can be set up with `git lfs install`:

```
$ git lfs install
```

These filters ensure that large files aren't written into the repository proper,
instead being stored locally at `.git/lfs/objects/{OID-PATH}` (where `{OID-PATH}`
is a sharded filepath of the form `OID[0:2]/OID[2:4]/OID`), synchronized with
the Git LFS server as necessary.  Here is a sample path to a file:

    .git/lfs/objects/4d/7a/4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393

The `clean` filter runs as files are added to repositories.  Git sends the
content of the file being added as STDIN, and expects the content to write
to Git as STDOUT.

* Stream binary content from STDIN to a temp file, while calculating its SHA-256
signature.
* Atomically move the temp file to `.git/lfs/objects/{OID-PATH}` if it does not
exist, and the sha-256 signature of the contents matches the given OID.
* Delete the temp file.
* Write the pointer file to STDOUT.

Note that the `clean` filter does not push the file to the server.  Use the
`git push` command to do that (lfs files are pushed before commits in a pre-push hook).

The `smudge` filter runs as files are being checked out from the Git repository
to the working directory.  Git sends the content of the Git blob as STDIN, and
expects the content to write to the working directory as STDOUT.

* Read 100 bytes.
* If the content is ASCII and matches the pointer file format:
  * Look for the file in `.git/lfs/objects/{OID-PATH}`.
  * If it's not there, download it from the server.
  * Write its contents to STDOUT
* Otherwise, simply pass the STDIN out through STDOUT.

The `.gitattributes` file controls when the filters run.  Here's a sample file that
runs all mp3 and zip files through Git LFS:

```
$ cat .gitattributes
*.mp3 filter=lfs -text
*.zip filter=lfs -text
```

Use the `git lfs track` command to view and add to `.gitattributes`.

# Les spécifications de GIT LFS

Ceci est un manuel pour les utilisateurs de Git LFS. Si quelqu'un peut ajouter la ligne de commande `git-lfs-spec` dans le package, pour afficher son contenu.

## Le pointeur

L'idée centrale de Git LFS, est de créé des fichiers pointer plutôt que de télécharger tout un repository.

* Les fichiers pointeurs sont des fichiers textes normé par UTF-8.
* Chaque ligne utilise le format {clef} {valeur}
* Les balises clef et valeur sont séparés par un espace.
* Les clefs utilisent les caractères suivant : `[a-z] [0-9] . -`.
* La première clefs d'un fichier pointer est toujours la clef : `version`.
* Les lignes de clef/valeur doivent être rangé par ordre alphabétique croissant avec comme exceptions la clef version qui devra toujours être la première clef. 
* Les clefs ne doivent pas être associé as des valeurs contenant des "newline caractère".
* Les fichiers pointeur doivent être rangé dans Git avec leurs exécutables "bit matching that of the replaced file" 
* Les fichier pointeurs doivent être d'une taille inférieur à 1024 bits, en incluant n'importe quel "pointer extension lines"
* Les fichiers pointeurs sont unique: ce qui signifie qu'il existe un fichier encodé pour chaque fichier pointeur créé

Pour ne pas créé de fichier pointer inutile, nous avons décidez qu'un "fichier vide" ne nécessite pas de fichier pointer DONC il ne sera pas traité par git LFS. Dès lors que ce fichier vide, devient volumineux, il pourra être considéré par l'extension git LFS. 

Les clefs nécessaire à la création d'un fichier pointeur sont :

* version 
  La clef version est associé à un URL donnant accès à une ressource dans une base de données privé.
* oid
  La clef oid est associé à un ID crypté, au format {hash-method}:{hash}. Pour l'instant il n'y à que le sha256 qui est      supporté pour crypté l'oid. Le hash(cryptage) utilise des caractère hexadécimaux en minuscule.
* size
  taille en bits.
  
Exemple d'une version 1 d'un pointer de texte:

```
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
size 12345
(ending \n)
```

Les fichiers créé avec l'outil de génération de fichier avec une version de pré-publication et différent URL, peuvent être lus par Git LFS. Mais il est déconseillé d'ecrire ces fichiers sans passé par l'outil de génération.

```
version https://hawser.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
size 12345
(ending \n)
```

Pour tester la conformité de n'importe quel outils générant soit même ces fichiers pointeur, la référence est le package Git LFS.

**A noté que : ** Les commande induise des comportements invisible par l'utilisateur, tel que :

* Les outils qui tirent et régénère les fichiers pointers, doivent être programmé pour préserver ou pour effectuer un        ensemble de test pour les clefs qu'ils ne connaissent pas.
* Une comparaison entre OID stocké en mémoire, pour les chemins identiques. 
* Exécuter la commande `pointer` permet de générer un fichier pointer pour le chemin local donné.

    ```
    $ git lfs pointer --file=path/to/file
    Git LFS pointer for path/to/file:

    version https://git-lfs.github.com/spec/v1
    oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345
    ```

* Run `pointer` to compare the blob OID of a pointer file built by Git LFS with
a pointer built by another tool.

  * Write the other implementation's pointer to "other/pointer/file":

    ```
    $ git lfs pointer --file=path/to/file --pointer=other/pointer/file
    Git LFS pointer for path/to/file:

    version https://git-lfs.github.com/spec/v1
    oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345

    Blob OID: 60c8d8ab2adcf57a391163a7eeb0cdb8bf348e44

    Pointer from other/pointer/file
    version https://git-lfs.github.com/spec/v1
    oid sha256 4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345

    Blob OID: 08e593eeaa1b6032e971684825b4b60517e0638d

    Pointers do not match
    ```

  * It can also read STDIN to get the other implementation's pointer:

    ```
    $ cat other/pointer/file | git lfs pointer --file=path/to/file --stdin
    Git LFS pointer for path/to/file:

    version https://git-lfs.github.com/spec/v1
    oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345

    Blob OID: 60c8d8ab2adcf57a391163a7eeb0cdb8bf348e44

    Pointer from STDIN
    version https://git-lfs.github.com/spec/v1
    oid sha256 4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
    size 12345

    Blob OID: 08e593eeaa1b6032e971684825b4b60517e0638d

    Pointers do not match
    ```

## Intercepting Git

Git LFS utilise les filtres clean et smudge pour décider quel fichier Git LFS pointera.
Les filtres globaux peuvent être mis en fonction avec la commande `git lfs install`:

```
$ git lfs install
```

Ces filtres assure que des fichiers volumineux ne sont pas écrit dans des dossier personnels, à la place ils sont stocké localement dans `.git/lfs/objects/{OID-PATH}` ou {OID-PATH} est le chemin partagé avec la forme : `OID[0:2]/OID[2:4]/OID`, synchroniser avec le serveur Git LFS pour pouvoir accèder au fichier depuis le serveur.

    .git/lfs/objects/4d/7a/4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393

Le filtre `clean` s'exécute comme des fichier ajouté à un repository. Git envoie le contenue du fichier qui est entrain d'être ajouté comme une entrée STDIN, et attends que le contenue envoyé soit reçus en sortie STDOUT.

* Le contenu du flux binaire provenant de STDIN vers un fichier temp, lorsqu'il calcul ça signature sha-256
* Atomically move the temp file to `.git/lfs/objects/{OID-PATH}` if it does not
exist, and the sha-256 signature of the contents matches the given OID.
* Delete the temp file.
* Write the pointer file to STDOUT.

Note that the `clean` filter does not push the file to the server.  Use the
`git push` command to do that (lfs files are pushed before commits in a pre-push hook).

The `smudge` filter runs as files are being checked out from the Git repository
to the working directory.  Git sends the content of the Git blob as STDIN, and
expects the content to write to the working directory as STDOUT.

* Read 100 bytes.
* If the content is ASCII and matches the pointer file format:
  * Look for the file in `.git/lfs/objects/{OID-PATH}`.
  * If it's not there, download it from the server.
  * Write its contents to STDOUT
* Otherwise, simply pass the STDIN out through STDOUT.

The `.gitattributes` file controls when the filters run.  Here's a sample file that
runs all mp3 and zip files through Git LFS:

```
$ cat .gitattributes
*.mp3 filter=lfs -text
*.zip filter=lfs -text
```

Use the `git lfs track` command to view and add to `.gitattributes`.
