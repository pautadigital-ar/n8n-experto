# Google Sheets desde n8n: cuenta de servicio, no OAuth

## El problema

Las credenciales OAuth2 de Google (`googleSheetsOAuth2Api`, `googleDriveOAuth2Api`) dependen de una app OAuth en Google Cloud. Si esa app está en estado **"Prueba" / "Testing"**, Google caduca el refresh token a los **7 días**. El síntoma aparece solo, sin que nadie toque nada:

> The provided authorization grant (e.g., authorization code, resource owner credentials) or refresh token is invalid, expired, revoked…

Reconectar la credencial lo arregla por otros 7 días. Publicar la app ("En producción") lo arregla del todo, pero Google puede bloquear el botón "Publicar app" hasta completar Página principal, Política de Privacidad y Condiciones del Servicio bajo un dominio verificado en Search Console. Para automatizaciones internas no vale la pena.

**Cómo confirmar la causa:** Google Cloud → Google Auth Platform → **Público** (`console.cloud.google.com/auth/audience`) → "Estado de publicación". La pantalla "Descripción general" no muestra este dato.

## La solución: cuenta de servicio

No tiene pantalla de consentimiento, ni tokens que caduquen, ni advertencias de verificación.

**Lo que se puede hacer desde el navegador:**

1. Verificar que el proyecto tenga habilitada **Google Sheets API** (y Drive API si se van a listar o crear archivos): `console.cloud.google.com/apis/dashboard?project=<id>`.
2. Crear la cuenta de servicio: IAM y administración → Cuentas de servicio → Crear. **Sin roles de IAM**: el acceso a una planilla no viene de IAM sino de compartirla, y darle roles sería regalar permisos sobre todo el proyecto.
3. Compartir cada planilla con el mail de la cuenta (`<nombre>@<proyecto>.iam.gserviceaccount.com`) como **Editor**, **destildando "Notificar"**: una cuenta de servicio no tiene buzón.

**Lo que hace el usuario** (es un secreto, no se toca):

4. En la cuenta de servicio → Claves → Agregar clave → JSON. Se descarga un archivo.
5. En n8n: Credentials → New → **Google Service Account API**. `client_email` y `private_key` completo, incluidas las líneas `-----BEGIN PRIVATE KEY-----` y `-----END PRIVATE KEY-----`. La credencial tiene que decir "Connection tested successfully".
6. Borrar el JSON descargado una vez cargado.

**Lo que se hace por MCP:**

7. En el nodo de Sheets: `parameters.authentication: "serviceAccount"` y `credentials` reemplazado completo por `{ "googleApi": { "id": "<id>", "name": "<nombre>" } }`. El ID de la credencial se lee en la URL de n8n al abrirla: `/home/credentials/<id>`.
8. Dejar el `documentId` por ID en vez de por selector de lista, que es lo más seguro con una cuenta de servicio (no tiene archivos propios en Drive).
9. Actualizar la nota del nodo: que diga que usa cuenta de servicio y que, si da **403**, lo primero es revisar que la planilla siga compartida.
10. Probar de punta a punta y leer la ejecución.

## Cuidado con credenciales compartidas

Una misma credencial OAuth suele estar en muchos workflows. Antes de reconectarla o reemplazarla, fijate cuáles la usan: migrar uno no arregla los demás.
