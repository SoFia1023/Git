git   config   --global  user.name   "Your name"
-- Explicacion
config: comando 
--global: Que se hara
user.name: Especificacion 
"Your name":Atributo

# Comando para ver versiones de git
* git --version

# Configuración de git
* git config --global user.name "My name" // nombre del que hizo el commit

* git config --global user.email "My email"

## Comandos para editar o ver la configuracion de Git

- git config --global --edit 
(Me lleva la configuración del sistema y me muestra los datos ingresados)
PD: Para salir del edit si es VIM es Esc y :wq . 

- git config --global --list (Este ayuda a listar) 
(Muestra información ingresada)

# Comando para iniciar git en un repositorio
* git init
(Se hace una vez por cada carpeta)

- git config --global init.defaultBranch main 
Este comando ayuda a nombrar la rama main por defecto


#   Comando para saber el estado de tus archivos
- git status 

# Importante: Pasos para crear versiones en nuestro código
1. Agregar todos los archivos al commit
* git add . 
(Agrega todo lo que no esté aún en commit)

* git add *py
(Todo lo que tenga esta extensión lo va a guardar en un commmit)

* git add nombre.py
(Sólo estoy agregando el archivo que estoy mencionando en el nombre)

# Pasos para crear un commmit
2.  Crear uan nueva version (Tomar la foto del código)
* git commit -m "Escribir comentario de lo que se está agregando al commmit"
(ANTES DE HACER UN COMMIT SE DEBE DE HACER UN ADD)

## Comando para listar las versiones de mi proyecto

- git log 
- git log --oneline

## Comando para cambiar el nombre de las ramas 

git branch -m "nombre antiguo" "nombre nuevo"

## Comando para cambiar de versión

- git checkout <Id del commit o nombre de la rama>
- git switch   <Id del commit o nombre de la rama>

##

- git config --global core.autocrlf true ("Sirve para darle el caracter que necesita window para su salto de linea)

## Comando para eliminar archivos desde git, se debe hacer commit cuando no se elimine correctamente

- git rm --cached nombre_archivo  -> "Este cached tiene  como funcion de recargar el VS"
- git rm nombre_archivo 

## Comando para recuperar/descartar (cambios) archivos de un commit anterior o una eliminacion 

- git restore nombre_archivo 
- git restore --staged nombre_archivo ("Tiene como funcion para descartar los cambios que se hayan ido para el unstage <commit>")

## REMOTA
# Si te traers un repositorio mediante el enlace es asi:
git clone enlace_del_repositorio

# Añadir a repositorio
git remote add origin URL_DEL_REPOSITORIO

# Montar tus commits al repositorio (antes de hacer un push debes estar actualizado con un pull)
git push origin nombre_de_la_rama

# Traeer cambios del repositorio 
git pull origin nombre_de_la_rama
