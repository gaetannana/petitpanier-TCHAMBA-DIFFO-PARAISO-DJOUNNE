# Image de base : serveur web nginx léger qui écoute sur le port 8080
FROM nginxinc/nginx-unprivileged:1.27-alpine

# Copier notre page dans le dossier servi par nginx
COPY index.html /usr/share/nginx/html/index.html

# Documenter le port d'écoute
EXPOSE 8080
