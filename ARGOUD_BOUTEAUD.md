# Rapport TP Hadoop — Analyse du fichier `tags.csv`

## Environnement technique

| Élément | Version |
|---------|---------|
| Hadoop | 2.7.3 (HDP Sandbox) |
| Python | 2.7.5 |
| mrjob | 0.7.4 |
| Système | CentOS (sandbox-hdp.hortonworks.com) |

> **Remarque importante** : Python 2.7 impose l'utilisation de `from StringIO import StringIO` et non `from io import StringIO`. Cette correction a été nécessaire sur tous les scripts car les mappers ne produisaient aucune sortie avec l'import incorrect.

---

## Fichier d'échantillon de test

Avant d'appliquer les traitements sur le fichier complet `tags.csv` (38 810 332 bytes ≈ 37 Mo, 1 093 361 lignes), un fichier `tags_sample.csv` de faible volume a été utilisé pour valider chaque script localement :

```bash
python 1.py tags_sample.csv
python 2.py tags_sample.csv
python 4.py tags_sample.csv
python 5.py tags_sample.csv
```

Chaque script intègre un bloc `try/except` dans le mapper pour ignorer la ligne d'en-tête et les éventuelles lignes malformées :

```python
def mapper(self, _, line):
    try:
        row = next(csv.reader(StringIO(line)))
        if row[0] == 'userId':
            return
        userId, movieId, tag, timestamp = row
        yield movieId, 1
    except Exception:
        pass
```

---

## Partie 1 — Configuration par défaut de Hadoop

### Question 1 — Nombre de tags par film

**Objectif** : Compter combien de tags chaque film (`movieId`) possède.

**Script `1.py`** : le mapper émet `(movieId, 1)` pour chaque ligne ; le reducer somme les valeurs par `movieId`.

```python
from mrjob.job import MRJob
import csv
from StringIO import StringIO

class TagsParFilm(MRJob):
    def mapper(self, _, line):
        try:
            row = next(csv.reader(StringIO(line)))
            if row[0] == 'userId':
                return
            userId, movieId, tag, timestamp = row
            yield movieId, 1
        except Exception:
            pass
    def reducer(self, movieId, counts):
        yield movieId, sum(counts)

if __name__ == '__main__':
    TagsParFilm.run()
```

**Commande Hadoop** :

```bash
python 1.py -r hadoop \
  --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar \
  hdfs:///user/maria_dev/exam_hadoop/tags.csv \
  -o hdfs:///user/maria_dev/exam_hadoop/output_q1
```

**Récupération des résultats** :

```bash
hdfs dfs -get /user/maria_dev/exam_hadoop/output_q1/part-00000 q1_result.txt
```

**Compteurs clés du job** :

| Compteur | Valeur |
|----------|--------|
| Map input records | 1 093 361 |
| Map output records | 1 093 360 |
| Reduce output records | **45 251** |

➡️ **45 251 films distincts** ont au moins un tag dans le dataset.

**Extrait des résultats** :

```
"1"       697
"10"      137
"100"     18
"1000"    10
"100001"  1
"100003"  3
"100008"  9
"100017"  9
"100032"  2
"100034"  19
```

📎 [Résultats complets — q1_result.txt](https://github.com/Bioteau/TP-Not-/blob/main/q1_result.txt)

---

### Question 2 — Nombre de tags par utilisateur

**Objectif** : Compter combien de tags chaque utilisateur (`userId`) a ajoutés.

**Script `2.py`** : le mapper émet `(userId, 1)` pour chaque ligne ; le reducer somme les valeurs par `userId`.

```python
from mrjob.job import MRJob
import csv
from StringIO import StringIO

class TagsParUtilisateur(MRJob):
    def mapper(self, _, line):
        try:
            row = next(csv.reader(StringIO(line)))
            if row[0] == 'userId':
                return
            userId, movieId, tag, timestamp = row
            yield userId, 1
        except Exception:
            pass
    def reducer(self, userId, counts):
        yield userId, sum(counts)

if __name__ == '__main__':
    TagsParUtilisateur.run()
```

**Commande Hadoop** :

```bash
python 2.py -r hadoop \
  --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar \
  hdfs:///user/maria_dev/exam_hadoop/tags.csv \
  -o hdfs:///user/maria_dev/exam_hadoop/output_q2
```

**Récupération des résultats** :

```bash
hdfs dfs -get /user/maria_dev/exam_hadoop/output_q2/part-00000 q2_result.txt
```

**Compteurs clés du job** :

| Compteur | Valeur |
|----------|--------|
| Map input records | 1 093 361 |
| Map output records | 1 093 360 |
| Reduce output records | **14 592** |

➡️ **14 592 utilisateurs distincts** ont ajouté au moins un tag.

**Extrait des résultats** :

```
"100001"  9
"100016"  50
"100028"  4
"100029"  1
"100033"  1
"100046"  133
"100051"  19
"100058"  5
"100065"  2
"100068"  19
```

📎 [Résultats complets — q2_result.txt](https://github.com/Bioteau/TP-Not-/blob/main/q2_result.txt)

---

## Partie 2 — Configuration personnalisée (blocs HDFS)

### Question — Nombre de blocs occupés par `tags.csv`

**Objectif** : Comparer le nombre de blocs occupés selon deux configurations de taille de bloc.

#### Configuration par défaut (128 Mo)

```bash
hdfs fsck /user/maria_dev/exam_hadoop/tags.csv -files -blocks
```

**Résultat** :
```
/user/maria_dev/exam_hadoop/tags.csv 38810332 bytes, 1 block(s): OK
  BP-243674277-172.17.0.2-1529333510191:blk_1073743080_2262 len=38810332 repl=1
Total blocks (validated): 1 (avg. block size 38810332 B)
```

#### Configuration avec blocs de 64 Mo

```bash
hdfs dfs -Ddfs.blocksize=67108864 -put -f tags.csv \
  /user/maria_dev/exam_hadoop/tags_64mb.csv

hdfs fsck /user/maria_dev/exam_hadoop/tags_64mb.csv -files -blocks
```

**Résultat** :
```
/user/maria_dev/exam_hadoop/tags_64mb.csv 38810332 bytes, 1 block(s): OK
  BP-243674277-172.17.0.2-1529333510191:blk_1073743156_2338 len=38810332 repl=1
Total blocks (validated): 1 (avg. block size 38810332 B)
```

#### Tableau comparatif

| Configuration | Taille de bloc | Taille du fichier | Nombre de blocs |
|---------------|---------------|-------------------|-----------------|
| Par défaut    | 128 Mo        | ~37 Mo            | **1 bloc**      |
| Personnalisée | 64 Mo         | ~37 Mo            | **1 bloc**      |

**Explication** : le fichier `tags.csv` pèse **38 810 332 bytes ≈ 37 Mo**, ce qui est inférieur aux deux tailles de blocs testées (64 Mo et 128 Mo). Il occupe donc **1 seul bloc dans les deux cas**. Si le fichier dépassait 64 Mo (ex. 200 Mo), il aurait occupé **4 blocs** en configuration 64 Mo et **2 blocs** en configuration 128 Mo. C'est d'ailleurs pourquoi Hadoop a lancé **2 splits** lors des jobs MapReduce (il utilise les splits logiques, pas uniquement les blocs physiques).

---

### Question 4 — Nombre d'occurrences de chaque tag

**Objectif** : Compter combien de fois chaque tag a été utilisé pour taguer un film.

**Script `4.py`** : le mapper émet `(tag, 1)` (normalisé en minuscules) ; le reducer somme par tag.

```python
from mrjob.job import MRJob
import csv
from StringIO import StringIO

class OccurrencesTag(MRJob):
    def mapper(self, _, line):
        try:
            row = next(csv.reader(StringIO(line)))
            if row[0] == 'userId':
                return
            userId, movieId, tag, timestamp = row
            yield tag.strip().lower(), 1
        except Exception:
            pass
    def reducer(self, tag, counts):
        yield tag, sum(counts)

if __name__ == '__main__':
    OccurrencesTag.run()
```

**Commande Hadoop** :

```bash
python 4.py -r hadoop \
  --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar \
  hdfs:///user/maria_dev/exam_hadoop/tags.csv \
  -o hdfs:///user/maria_dev/exam_hadoop/output_q4
```

**Récupération des résultats** :

```bash
hdfs dfs -get /user/maria_dev/exam_hadoop/output_q4/part-00000 q4_result.txt
```

**Compteurs clés du job** :

| Compteur | Valeur |
|----------|--------|
| Map input records | 1 093 361 |
| Map output records | 1 093 360 |
| Reduce output records | **65 414** |

➡️ **65 414 tags distincts** ont été utilisés dans le dataset.

**Extrait des résultats** :

```
"!950's superman tv show"          1
"#1 prediction"                    3
"#adventure"                       1
"#antichrist"                      1
"#boring #lukeiamyourfather"       1
"#boring"                          1
"#danish"                          2
"#documentary"                     1
"#entertaining"                    1
"#exorcism"                        1
```

📎 [Résultats complets — q4_result.txt](https://github.com/Bioteau/TP-Not-/blob/main/q4_result.txt)

---

### Question 5 — Nombre de tags introduits par un même utilisateur pour chaque film

**Objectif** : Pour chaque film, compter combien de tags le même utilisateur a introduits (clé composite `(movieId, userId)`).

**Script `5.py`** : le mapper émet `((movieId, userId), 1)` ; le reducer somme par paire.

```python
from mrjob.job import MRJob
import csv
from StringIO import StringIO

class TagsParFilmEtUtilisateur(MRJob):
    def mapper(self, _, line):
        try:
            row = next(csv.reader(StringIO(line)))
            if row[0] == 'userId':
                return
            userId, movieId, tag, timestamp = row
            yield (movieId, userId), 1
        except Exception:
            pass
    def reducer(self, key, counts):
        movieId, userId = key
        yield (movieId, userId), sum(counts)

if __name__ == '__main__':
    TagsParFilmEtUtilisateur.run()
```

**Commande Hadoop** :

```bash
python 5.py -r hadoop \
  --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar \
  hdfs:///user/maria_dev/exam_hadoop/tags.csv \
  -o hdfs:///user/maria_dev/exam_hadoop/output_q5
```

**Récupération des résultats** :

```bash
hdfs dfs -get /user/maria_dev/exam_hadoop/output_q5/part-00000 q5_result.txt
```

**Compteurs clés du job** :

| Compteur | Valeur |
|----------|--------|
| Map input records | 1 093 361 |
| Map output records | 1 093 360 |
| Reduce output records | **305 356** |

➡️ **305 356 paires (film, utilisateur)** distinctes dans le dataset.

**Extrait des résultats** :

```
["1", "100538"]   4
["1", "10231"]    2
["1", "102568"]   4
["1", "102901"]   1
["1", "103368"]   1
["1", "103371"]   1
["1", "103883"]   3
["1", "104394"]   9
["1", "1048"]     1
["1", "105717"]   1
```

*Lecture : l'utilisateur `104394` a introduit **9 tags** pour le film `1`.*

📎 [Résultats complets — q5_result.txt](https://github.com/Bioteau/TP-Not-/blob/main/q5_result.txt)

---

## Récapitulatif général

| Question | Description | Résultat |
|----------|-------------|----------|
| Q1 | Tags par film | 45 251 films distincts |
| Q2 | Tags par utilisateur | 14 592 utilisateurs distincts |
| Q blocs (128 Mo) | Blocs HDFS config par défaut | **1 bloc** |
| Q blocs (64 Mo) | Blocs HDFS config personnalisée | **1 bloc** |
| Q4 | Occurrences de chaque tag | 65 414 tags distincts |
| Q5 | Tags par (film, utilisateur) | 305 356 paires distinctes |

---

## Difficultés rencontrées et solutions

| Problème | Cause | Solution |
|----------|-------|----------|
| `ImportError: No module named mrjob.job` | mrjob non installé sur Python 2.7 | `pip install mrjob==0.7.4` |
| `Map output records=0` (job vide) | `from io import StringIO` incompatible Python 2.7 | Remplacé par `from StringIO import StringIO` |
| `nano: command not found` | Éditeur non disponible sur le sandbox | Utilisation de `cat > fichier << 'EOF'` |
| `pip: command not found` | pip absent par défaut | Installation via `get-pip.py` |