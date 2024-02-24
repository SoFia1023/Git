git   config   --global  user.name   "Your name"
-- Explicacion
git: comando 
--global: Que se hara
user.name: Especificacion 
"Your name":Atributo

# Comando para ver versiones de git
* git --version

# Configuración de git
* git config --global user.name "My name"

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
