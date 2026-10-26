# Despliegue de la app PokeDex en Azure

## 1. Introducción

En este documento se explica el proceso seguido para publicar la aplicación **PokeDex**, una SPA construida con Angular, en la nube de Microsoft Azure mediante el servicio **Azure App Service**. Además, se detalla cómo se establecieron las cabeceras de seguridad HTTP a través de un archivo `web.config`.

## 2. Requisitos previos

- Una cuenta de Azure (Azure for Students).
- Node.js y npm instalados en el equipo.
- Angular CLI instalado de forma global (`npm install -g @angular/cli`).
- El código fuente de la aplicación PokeDex.

## 3. Creación de la Web App en Azure

Desde el portal de Azure se creó un recurso de tipo **Web App** con los siguientes valores de configuración:

| Parámetro | Valor |
|---|---|
| Grupo de recursos | `rg-pokedex-prod` |
| Nombre de la aplicación | `app-pokedex-web-prod-la` |
| Publicar | Código |
| Runtime stack | .NET 10 (LTS) |
| Sistema operativo | Windows |
| Región | Mexico Central |
| Plan de precios | Free F1 |

> Para nombrar los recursos se siguió la convención `prefijo-proyecto-tipo-entorno` definida en el curso.

![Creación de la Web App](img/01-crear-webapp.png)

## 4. Preparación del proyecto en el entorno local

1. Se descargó el proyecto comprimido en un archivo `.zip` y se extrajo su contenido.
2. Se instalaron las dependencias necesarias:

   ```bash
   npm i
   ```

3. Se levantó la aplicación en local para comprobar que funcionaba correctamente:

   ```bash
   npm start
   ```

## 5. Configuración de la seguridad (web.config)

Con el fin de añadir las cabeceras de seguridad HTTP, se creó un archivo `web.config` en la raíz de la carpeta `src` del proyecto.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>

    <!-- Reescribe las rutas de la SPA (Angular) a index.html -->
    <rewrite>
      <rules>
        <rule name="Angular SPA" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
          </conditions>
          <action type="Rewrite" url="/index.html" />
        </rule>
      </rules>
    </rewrite>

    <!-- Cabeceras de seguridad -->
    <httpProtocol>
      <customHeaders>
        <remove name="X-Powered-By" />
        <add name="Strict-Transport-Security" value="max-age=31536000; includeSubDomains" />
        <add name="X-Content-Type-Options" value="nosniff" />
        <add name="X-Frame-Options" value="DENY" />
        <add name="Referrer-Policy" value="strict-origin-when-cross-origin" />
        <add name="Permissions-Policy" value="camera=(), microphone=(), geolocation=()" />
        <add name="Content-Security-Policy" value="default-src 'self'; script-src 'self' 'sha256-QmIi/LDJnOEYAO9CVS/+JVvnKMiVsY6fn0FVoHnkPfg='; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: https://raw.githubusercontent.com https://pokeapi.co https://assets.pokemon.com; font-src 'self' data: https://fonts.gstatic.com; connect-src 'self' https://beta.pokeapi.co https://pokeapi.co; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'" />
      </customHeaders>
    </httpProtocol>

  </system.webServer>
</configuration>
```

Este archivo cumple con dos tareas distintas:

**a) Redirección de rutas de la SPA.** Cuando la ruta solicitada no corresponde a un archivo ni a un directorio real, el servidor responde con `index.html` y deja que el enrutador de Angular se encargue de interpretar la URL. Si no existiera esta regla, al recargar la página estando en una ruta interna se obtendría un error 404.

**b) Cabeceras de seguridad.**

| Cabecera | Valor | Función |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Hace que el navegador utilice siempre HTTPS durante un año. |
| `X-Content-Type-Options` | `nosniff` | Impide que el navegador intente deducir por su cuenta el tipo de contenido de los archivos. |
| `X-Frame-Options` | `DENY` | No permite que la página se cargue dentro de un `iframe`, lo que protege frente al clickjacking. |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Reduce la información de origen que se comparte con otros sitios. |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Desactiva el uso de la cámara, el micrófono y la geolocalización. |
| `Content-Security-Policy` | Ver abajo | Define desde qué orígenes se pueden cargar scripts, estilos, imágenes y conexiones. |
| `X-Powered-By` | *(eliminada)* | Evita revelar datos sobre la tecnología que usa el servidor. |

La política `Content-Security-Policy` autoriza únicamente lo siguiente:

- **Scripts:** los del propio sitio y el script en línea que se identifica mediante su hash `sha256`.
- **Estilos y fuentes:** los del propio sitio y los provenientes de Google Fonts (`fonts.googleapis.com`, `fonts.gstatic.com`).
- **Imágenes:** las del propio sitio, las de tipo `data:` y las de los dominios que alojan imágenes de Pokémon (`raw.githubusercontent.com`, `pokeapi.co`, `assets.pokemon.com`).
- **Conexiones (API):** las dirigidas a `pokeapi.co` y `beta.pokeapi.co`.
- Asimismo, bloquea el uso de `iframes` (`frame-ancestors 'none'`) y de objetos incrustados (`object-src 'none'`), y restringe `base-uri` y `form-action` al mismo sitio.

### Registro del archivo en `angular.json`

Para que Angular incluya el `web.config` en la carpeta de salida al compilar, se añadió su ruta en la sección `assets` de `architect > build > options`:

```json
"assets": [
  "src/favicon.ico",
  "src/og-image.jpg",
  "src/assets",
  "src/404.html",
  "src/web.config"
]
```

## 6. Compilación del proyecto

```bash
ng build
```

Este comando produce los archivos listos para producción (la configuración `production` se aplica por defecto) dentro de la carpeta `dist/pokedex-angular`, que es la ruta indicada en `outputPath` de `angular.json`. Gracias a la sección `assets`, el `web.config` también queda copiado en esa carpeta.

## 7. Publicación de los archivos con Kudu

1. Desde el portal de Azure, se abrió la Web App y se accedió a **Kudu** (Herramientas avanzadas).
2. Se navegó hasta la carpeta `/site/wwwroot`.
3. Se borró el archivo `hostingstart.html`, que Azure genera de forma predeterminada.
4. Se cargaron todos los archivos resultantes de la compilación (el **contenido** de `dist/pokedex-angular`, y no la carpeta como tal), de manera que `index.html` y `web.config` quedaran directamente dentro de `/site/wwwroot`.

![Kudu - wwwroot](img/02-kudu-wwwroot.png)

## 8. Comprobación del despliegue

1. Se ingresó a la URL pública de la Web App para confirmar que la aplicación se cargaba sin problemas.
2. Se evaluó el sitio en [securityheaders.com](https://securityheaders.com) con el fin de validar las cabeceras de seguridad.

| Momento | Calificación |
|---|---|
| Antes del `web.config` | F |
| Después del `web.config` | A+ |

![Resultado en securityheaders.com](img/03-securityheaders.png)

## 9. Resultado final

La aplicación PokeDex quedó desplegada en Azure App Service y disponible públicamente en internet. Con las cabeceras de seguridad definidas en el `web.config`, el sitio alcanzó la calificación **A+** en securityheaders.com.
