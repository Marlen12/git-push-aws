# Limpieza: borrar todo al terminar

Sigue estos pasos si no vas a mantener tu web publicada. Así te aseguras de no pagar nada.

## En AWS

1. **S3** → selecciona tu bucket → **Empty** → escribe `permanently delete` → **Empty**.
2. Con el bucket ya vacío: selecciónalo → **Delete** → escribe el nombre del bucket → **Delete bucket**.
3. **IAM** → **Roles** → selecciona `deploy-TU-REPO` → **Delete**.
4. **IAM** → **Identity providers** → selecciona `token.actions.githubusercontent.com` → **Delete**.
   Hazlo solo si ningún otro proyecto tuyo usa GitHub Actions con AWS.

## En GitHub (opcional)

- Para que el workflow deje de ejecutarse: **Actions** → **Publicar web en S3** → **⋯** → **Disable workflow**.
- O borra las variables en **Settings → Secrets and variables → Actions**: el workflow se saltará solo.
- Tu repositorio puede quedarse como recuerdo del taller.
