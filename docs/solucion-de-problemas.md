# Solución de problemas

Busca el mensaje que ves (en el navegador o en la pestaña **Actions**) y sigue la solución.

| Síntoma | Causa probable | Solución |
| --- | --- | --- |
| El workflow aparece como **skipped** (gris) | Faltan las variables del Paso 5 | Crea `AWS_ROLE_ARN`, `AWS_REGION` y `S3_BUCKET` y ejecuta el workflow a mano con **Run workflow** |
| El workflow no aparece en **Actions** | Creaste el repo con *Fork* o Actions está desactivado | **Settings → Actions → General → Allow all actions**. Mejor aún: crea el repo con **Use this template** |
| `Could not load credentials from any providers` | Falta `permissions: id-token: write` en el workflow | Restaura el bloque `permissions` de `deploy.yml` |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | La trust policy del rol no coincide con tu repo o rama | IAM → tu rol → **Trust relationships**: revisa que diga exactamente `repo:TU-USUARIO/TU-REPO:ref:refs/heads/main`. Las mayúsculas cuentan |
| `No OpenIDConnect provider found` | No hiciste el Paso 3 en esta cuenta de AWS | Crea el proveedor OIDC (Paso 3) |
| `Request ARN is invalid` | El valor de `AWS_ROLE_ARN` está mal copiado | Copia de nuevo el ARN completo del rol, sin espacios |
| `AccessDenied` en el paso *Subir la carpeta site* | El nombre del bucket en la política del rol no coincide con `S3_BUCKET` | Revisa la política inline `s3-deploy`: `NOMBRE-BUCKET` debe estar cambiado **en las dos líneas** |
| `NoSuchBucket` | `S3_BUCKET` tiene un error de tipeo o la región no coincide | Compara con el nombre exacto en la consola de S3 y revisa `AWS_REGION` |
| **403 Forbidden** al abrir la web | Falta la bucket policy o sigue activo *Block Public Access* | Paso 2, puntos 5 y 9 |
| **404 Not Found** al abrir la web | No hay `index.html` en la raíz del bucket | Revisa que el workflow terminó en verde y que *Index document* es `index.html` |
| La web no muestra mis cambios | Caché del navegador | **Ctrl + F5**, o abre una ventana privada |
| `git push` pide usuario y contraseña y falla | GitHub ya no acepta contraseñas | Usa `gh auth login`, SSH o un token personal |

## Cómo leer un error en Actions

1. Pestaña **Actions** → clic en la ejecución en rojo.
2. Clic en el job **deploy**.
3. Abre el paso con la ❌ y lee las últimas líneas: casi siempre el mensaje está ahí.
4. Busca ese mensaje en la tabla de arriba.
