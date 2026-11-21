# git push a la nube: publica tu web con AWS S3 y GitHub Actions

Taller práctico de 90 minutos. Al terminar, tu web estará publicada en **Amazon S3** y se actualizará sola cada vez que hagas `git push` a la rama `main`, gracias a **GitHub Actions**.

```
Tu laptop  ──git push──▶  GitHub Actions  ──token OIDC──▶  Rol IAM  ──credenciales temporales──▶  Bucket S3  ──▶  Navegador
```

> Puedes seguir esta guía por tu cuenta, a tu ritmo, aunque no hayas estado en la explicación.

---

## Contenido

0. [Antes de empezar](#0-antes-de-empezar)
1. [Crea tu copia del repositorio](#1-crea-tu-copia-del-repositorio)
2. [Crea el bucket S3 con hosting estático](#2-crea-el-bucket-s3-con-hosting-estático)
3. [Registra a GitHub como proveedor de identidad (OIDC)](#3-registra-a-github-como-proveedor-de-identidad-oidc)
4. [Crea el rol IAM que usará GitHub](#4-crea-el-rol-iam-que-usará-github)
5. [Configura las variables en GitHub](#5-configura-las-variables-en-github)
6. [Primer despliegue automático](#6-primer-despliegue-automático)
7. [La prueba: cambia tu web con git push](#7-la-prueba-cambia-tu-web-con-git-push)
8. [Retos extra](#8-retos-extra)
9. [Limpieza](#9-limpieza)

¿Algo falló? Revisa [docs/solucion-de-problemas.md](docs/solucion-de-problemas.md).

---

## Qué hay en este repositorio

| Ruta | Para qué sirve |
| --- | --- |
| `site/` | Tu web. **Solo esta carpeta** se publica en S3. |
| `.github/workflows/deploy.yml` | El workflow que publica la web en cada push a `main`. |
| `docs/politicas/` | Las políticas JSON que vas a copiar y pegar en la consola de AWS. |
| `docs/solucion-de-problemas.md` | Errores comunes y cómo resolverlos. |
| `docs/limpieza.md` | Cómo borrar todo al terminar para no pagar nada. |

El workflow **no hace nada hasta que configures las variables del Paso 5**. Por eso, aunque hagas push antes, no vas a ver errores en rojo.

---

## 0. Antes de empezar

Necesitas:

- [ ] Una cuenta de **GitHub**.
- [ ] Una cuenta de **AWS** verificada. Las cuentas nuevas entran al plan gratuito con créditos; este taller cuesta centavos de dólar o nada.
- [ ] **Git** instalado y configurado:
  ```bash
  git config --global user.name "Tu Nombre"
  git config --global user.email "tu@correo.com"
  ```
- [ ] Poder hacer push a GitHub desde tu computadora (GitHub CLI con `gh auth login`, SSH o un token).
- [ ] Un editor de código, por ejemplo VS Code.

Durante la guía vas a reemplazar estos textos por tus propios valores:

| Texto en la guía | Qué poner | Ejemplo |
| --- | --- | --- |
| `TU-USUARIO` | Tu usuario de GitHub | `anaquispe` |
| `TU-REPO` | El nombre de tu repositorio | `mi-web` |
| `NOMBRE-BUCKET` | Un nombre único para tu bucket, en minúsculas | `taller-push-anaquispe-2026` |
| `REGION` | La región de AWS que indique el tallerista | `us-east-1` |

Anota tus valores en un papel o archivo de texto: los vas a usar varias veces.

---

## 1. Crea tu copia del repositorio

⏱️ 5 minutos

1. Arriba en esta página haz clic en **Use this template** → **Create a new repository**.
2. **Owner**: tu usuario. **Repository name**: `TU-REPO` (por ejemplo `mi-web`). Elige **Public**.
3. Clic en **Create repository**.
4. Clona tu nuevo repositorio en tu computadora:

   ```bash
   git clone https://github.com/TU-USUARIO/TU-REPO.git
   cd TU-REPO
   ```

5. Abre `site/index.html` y cambia `Tu Nombre` por tu nombre. Todavía no hagas push; primero vamos a preparar AWS.

> ⚠️ Usa **Use this template**, no **Fork**. En un fork, GitHub Actions viene desactivado por defecto.

---

## 2. Crea el bucket S3 con hosting estático

⏱️ 15 minutos · 🖥️ Consola de AWS

Un **bucket** es una "carpeta" en la nube de AWS. Lo vamos a configurar para que sirva páginas web.

1. Entra a la [consola de AWS](https://console.aws.amazon.com/) y, arriba a la derecha, elige la región `REGION`.
2. Busca **S3** → **Create bucket**.
3. **Bucket name**: `NOMBRE-BUCKET`. Debe ser único en todo el mundo, en minúsculas y sin espacios.
4. **Object Ownership**: deja *ACLs disabled*.
5. **Block Public Access settings for this bucket**: **desmarca** *Block all public access* y marca la casilla que confirma que entiendes que el bucket será público.
6. Deja todo lo demás como está → **Create bucket**.
7. Entra a tu bucket → pestaña **Properties** → baja hasta **Static website hosting** → **Edit**:
   - **Static website hosting**: *Enable*
   - **Hosting type**: *Host a static website*
   - **Index document**: `index.html`
   - **Error document**: `error.html`
   - **Save changes**
8. En esa misma sección copia el **Bucket website endpoint**. Se ve así:
   `http://NOMBRE-BUCKET.s3-website-REGION.amazonaws.com`
   👉 Esa es la dirección pública de tu web. Guárdala.
9. Pestaña **Permissions** → **Bucket policy** → **Edit**. Pega el contenido de [`docs/politicas/bucket-policy.json`](docs/politicas/bucket-policy.json), cambia `NOMBRE-BUCKET` por el nombre de tu bucket y haz clic en **Save changes**.

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "LecturaPublica",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::NOMBRE-BUCKET/*"
       }
     ]
   }
   ```

10. **Prueba rápida** (opcional): pestaña **Objects** → **Upload** → sube `site/index.html` → abre el endpoint en tu navegador. Si ves la página, S3 está listo. Es la última vez que subes algo a mano.

✅ **Checkpoint:** tienes el endpoint de tu web anotado.

---

## 3. Registra a GitHub como proveedor de identidad (OIDC)

⏱️ 5 minutos · 🖥️ Consola de AWS

Esto permite que GitHub Actions pida **credenciales temporales** a AWS sin que tengas que guardar claves en ningún lado. Se hace **una sola vez por cuenta de AWS**.

> Si el tallerista te dijo que usas una cuenta compartida, este paso ya está hecho: salta al Paso 4.
> Si AWS dice que el proveedor ya existe, también puedes saltar al Paso 4.

1. Busca **IAM** → en el menú izquierdo, **Identity providers** → **Add provider**.
2. **Provider type**: *OpenID Connect*.
3. **Provider URL**: `https://token.actions.githubusercontent.com`
4. **Audience**: `sts.amazonaws.com`
5. Clic en **Add provider**.

✅ **Checkpoint:** en la lista de *Identity providers* aparece `token.actions.githubusercontent.com`.

---

## 4. Crea el rol IAM que usará GitHub

⏱️ 10 minutos · 🖥️ Consola de AWS

El **rol** define *quién* puede entrar a AWS (solo tu repositorio, desde la rama `main`) y *qué* puede hacer (solo escribir en tu bucket).

1. En IAM → **Roles** → **Create role**.
2. **Trusted entity type**: *Web identity*.
3. **Identity provider**: `token.actions.githubusercontent.com`
   **Audience**: `sts.amazonaws.com`
4. **GitHub organization**: `TU-USUARIO`
   **GitHub repository**: `TU-REPO`
   **GitHub branch**: `main`
   → **Next**.
5. En la pantalla de permisos **no marques ninguna política** → **Next**.
6. **Role name**: `deploy-TU-REPO` (por ejemplo `deploy-mi-web`). No lo llames `GitHubActions`. → **Create role**.
7. Abre el rol que acabas de crear → **Add permissions** → **Create inline policy** → pestaña **JSON**. Pega el contenido de [`docs/politicas/permisos-rol.json`](docs/politicas/permisos-rol.json), cambia **las dos veces** que aparece `NOMBRE-BUCKET`, y continúa:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "ListarBucket",
         "Effect": "Allow",
         "Action": "s3:ListBucket",
         "Resource": "arn:aws:s3:::NOMBRE-BUCKET"
       },
       {
         "Sid": "SubirYBorrarArchivos",
         "Effect": "Allow",
         "Action": ["s3:PutObject", "s3:DeleteObject"],
         "Resource": "arn:aws:s3:::NOMBRE-BUCKET/*"
       }
     ]
   }
   ```

   **Policy name**: `s3-deploy` → **Create policy**.
8. Copia el **ARN** del rol (aparece arriba en la página del rol). Se ve así:
   `arn:aws:iam::123456789012:role/deploy-mi-web`
9. Verifica la pestaña **Trust relationships**. Debe tener una condición como la de [`docs/politicas/trust-policy.json`](docs/politicas/trust-policy.json), con **tu** usuario y **tu** repo:

   ```json
   "token.actions.githubusercontent.com:sub": "repo:TU-USUARIO/TU-REPO:ref:refs/heads/main"
   ```

   Esta línea es la que impide que otro repositorio use tu rol. Si el usuario o el repo están mal escritos (las mayúsculas cuentan), el despliegue va a fallar.

✅ **Checkpoint:** tienes el ARN del rol anotado. **Ya terminaste con la consola de AWS.**

---

## 5. Configura las variables en GitHub

⏱️ 5 minutos · 🌐 github.com

1. En tu repositorio → **Settings** → **Secrets and variables** → **Actions**.
2. Pestaña **Variables** → **New repository variable**. Crea estas tres:

| Name | Value (ejemplo) |
| --- | --- |
| `AWS_ROLE_ARN` | `arn:aws:iam::123456789012:role/deploy-mi-web` |
| `AWS_REGION` | `us-east-1` |
| `S3_BUCKET` | `taller-push-anaquispe-2026` |

> ¿Por qué *variables* y no *secrets*? Porque ninguno de estos valores es una clave. Sin la confianza OIDC del Paso 4, el ARN no le sirve a nadie más.

✅ **Checkpoint:** ves las tres variables en la pestaña *Variables*.

---

## 6. Primer despliegue automático

⏱️ 10 minutos

1. Abre [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) y léelo. Las partes clave son:

   | Parte | Qué hace |
   | --- | --- |
   | `on: push: branches: [main]` | Se ejecuta cada vez que haces push a `main`. |
   | `workflow_dispatch` | Te deja ejecutarlo a mano desde la pestaña Actions. |
   | `permissions: id-token: write` | Permite que GitHub emita el token OIDC. Sin esto, AWS no te deja entrar. |
   | `if: vars.S3_BUCKET != ''` | Si no configuraste las variables, el workflow se salta sin fallar. |
   | `configure-aws-credentials` | Cambia el token de GitHub por credenciales temporales del rol. |
   | `aws s3 sync site/ ... --delete` | Sube solo lo que cambió y borra del bucket lo que ya no existe en `site/`. |

2. Haz commit del cambio de tu nombre (Paso 1) y súbelo:

   ```bash
   git add site/index.html
   git commit -m "Pongo mi nombre en la web"
   git push
   ```

3. En GitHub abre la pestaña **Actions**. Vas a ver la ejecución **Publicar web en S3**. Entra para ver cada paso en vivo.
4. Cuando esté en verde ✅, abre el endpoint de tu bucket (Paso 2). Tu web ya está en la nube.

> Si ya habías hecho push antes de configurar las variables, ve a **Actions** → **Publicar web en S3** → **Run workflow** para ejecutarlo a mano.

---

## 7. La prueba: cambia tu web con git push

⏱️ 5 minutos

Este es el momento clave del taller: **publicar sin tocar AWS**.

1. Abre `site/index.html` y cambia `Versión 1` por `Versión 2`.
2. Ejecuta:

   ```bash
   git commit -am "Versión 2"
   git push
   ```

3. Mira la ejecución en **Actions**. Cuando esté en verde, recarga tu web con **Ctrl + F5** (o **Cmd + Shift + R** en Mac).

🎉 **Lo lograste.** Cada push a `main` publica tu web automáticamente.

---

## 8. Retos extra

Para quien termine antes:

- [ ] Agrega una segunda página `site/sobre-mi.html` y enlázala desde `index.html`.
- [ ] Cambia los colores en `site/style.css`.
- [ ] Borra un archivo de `site/`, haz push y comprueba que también desaparece del bucket (eso hace `--delete`).
- [ ] Agrega un badge de estado al README:
  `![Deploy](https://github.com/TU-USUARIO/TU-REPO/actions/workflows/deploy.yml/badge.svg)`
- [ ] Rompe algo a propósito (por ejemplo, borra `id-token: write`), haz push y lee el error en Actions. Luego arréglalo.

**Para seguir aprendiendo después del taller:**

- **HTTPS y dominio propio:** el endpoint de S3 solo usa HTTP. Para HTTPS se pone **Amazon CloudFront** delante del bucket, con un certificado gratuito de **AWS Certificate Manager**.
- **Sitios con build** (React, Vue, Astro): agrega un paso `npm ci && npm run build` antes del sync y sincroniza `dist/` en vez de `site/`.
- **Aprobación antes de publicar:** usa *Environments* de GitHub con revisores obligatorios.

---

## 9. Limpieza

Si no vas a mantener la web publicada, sigue [docs/limpieza.md](docs/limpieza.md) para borrar todo. Si la dejas publicada, una web pequeña como esta cuesta centavos de dólar al mes.

---

## Créditos

Material del taller **"git push a la nube: publica tu web con AWS S3 y GitHub Actions"**.
Si encuentras un error en la guía, abre un *issue* en el repositorio original.
