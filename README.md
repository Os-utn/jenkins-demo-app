# jenkins-demo-app — Práctica CI/CD con Jenkins (Act2_P3)

Pipeline declarativo con parámetros, webhook y notificaciones.

## Estructura

```
jenkins-demo-app/
├── index.js              # función suma()
├── package.json          # scripts test y build
├── test/
│   └── suma.test.js      # pruebas con assert
├── Jenkinsfile           # pipeline: Checkout, Instalar, Test, Build, Deploy
└── .gitignore
```

## Probar en local

```bash
npm test          # ✅ Todas las pruebas pasaron
npm run build     # genera dist/index.js
```

## Paso 1 — Levantar Jenkins

```bash
docker volume create jenkins_home

docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk21

docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

> La práctica indica `lts-jdk17`, pero las versiones LTS actuales exigen **Java 21** y entran en loop de reinicio con JDK 17. Usa `lts-jdk21`.

Abre http://localhost:8080 → pega la contraseña → *Install suggested plugins* → crea tu usuario admin.

## Paso 2 — Plugins

Manage Jenkins → Plugins → Available: **Pipeline**, **Git**, **GitHub Integration**, **Blue Ocean**, **Credentials Binding**, **Email Extension**. Reinicia si lo pide.

Nota: el `Jenkinsfile` usa `sh 'node -v'` y `npm`, así que Node debe existir donde corra el build. La imagen `jenkins/jenkins:lts` **no trae Node**. Opciones:

- Instalarlo dentro del contenedor: `docker exec -u root jenkins bash -c "curl -fsSL https://deb.nodesource.com/setup_18.x | bash - && apt-get install -y nodejs"`
- O usar el plugin **NodeJS** (Manage Jenkins → Tools → NodeJS installations) y envolver los stages con `nodejs('node18') { ... }`.

## Paso 3 — Subir a GitHub

```bash
git init
git add .
git commit -m "Proyecto base para práctica de Jenkins"
git branch -M main
git remote add origin https://github.com/TU_USUARIO/jenkins-demo-app.git
git push -u origin main
```

## Paso 5 — Job Multibranch Pipeline

New Item → nombre `jenkins-demo-app` → **Multibranch Pipeline**
Branch Sources → Git (o GitHub) → URL del repo → credenciales si es privado.
Guardar: Jenkins escanea, detecta el `Jenkinsfile` y lanza el primer build.

## Paso 6 — Ejecutar con parámetros

Entra a la rama `main` → **Build with Parameters** → elige `ENTORNO` (staging/produccion) y marca `EJECUTAR_DEPLOY`. Observa Stage View / Blue Ocean y el Console Output.

> En Multibranch, el primer build de una rama nueva se ejecuta con los valores por defecto (Jenkins aún no conoce los parámetros); a partir del segundo aparece *Build with Parameters*.

## Paso 7 — Webhook

GitHub → repo → Settings → Webhooks → Add webhook

- Payload URL: `http://TU_IP:8080/github-webhook/` (con barra final)
- Content type: `application/json`
- Evento: *Just the push event*

Si Jenkins es local: `ngrok http 8080` y usa la URL pública.
En el job (o en la carpeta multibranch): marca **GitHub hook trigger for GITScm polling** / *Scan Repository Triggers*.

Prueba: cambia algo en `index.js`, haz push, el build debe dispararse solo.

## Paso 8 — Notificación por correo (reto)

Configura SMTP en Manage Jenkins → System → **Extended E-mail Notification**, luego descomenta el bloque `mail` en el `post { failure { ... } }` del `Jenkinsfile` y pon tu correo.

## Checklist

- [ ] Jenkins corriendo en localhost:8080
- [ ] Plugins de Git y Pipeline instalados
- [ ] Repositorio con `Jenkinsfile` en GitHub
- [ ] Job Multibranch creado y ejecutado sin errores
- [ ] Etapas Checkout, Test, Build en verde
- [ ] Parámetros ENTORNO y EJECUTAR_DEPLOY funcionando
- [ ] Webhook configurado y probado con un push real
- [ ] (Reto) Notificación por correo configurada
