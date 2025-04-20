# Pipeline DevSecOps

Este repositorio implementa un **pipeline DevSecOps** utilizando **GitHub Actions** para realizar análisis de seguridad en diferentes etapas del ciclo de vida del desarrollo de software. El pipeline integra múltiples herramientas de seguridad para garantizar que el código, las dependencias, la infraestructura y los entornos en ejecución sean seguros. Los informes generados por estos análisis se suben como artefactos en GitHub Actions y se pueden importar a **DefectDojo** para una gestión centralizada de vulnerabilidades, siempre que DefectDojo esté expuesto a internet mediante **ngrok**. Este documento explica cómo funciona el pipeline, cómo se manejan los artefactos y cómo configurar DefectDojo con ngrok para la integración de informes.

## Tabla de Contenidos
- [Descripción General](#descripción-general)
- [Flujos de Trabajo del Pipeline](#flujos-de-trabajo-del-pipeline)
- [Gestión de Artefactos](#gestión-de-artefactos)
- [Integración con DefectDojo](#integración-con-defectdojo)
  - [Ejecutar DefectDojo](#ejecutar-defectdojo)
  - [Exponer DefectDojo con ngrok](#exponer-defectdojo-con-ngrok)
  - [Actualizar Secretos de GitHub](#actualizar-secretos-de-github)
- [Visualización de Informes en DefectDojo](#visualización-de-informes-en-defectdojo)
- [Prerrequisitos](#prerrequisitos)
- [Instrucciones de Configuración](#instrucciones-de-configuración)

## Descripción General
El pipeline DevSecOps automatiza las pruebas de seguridad mediante flujos de trabajo de GitHub Actions que se activan al realizar un push a la rama `main`. Incluye las siguientes prácticas de seguridad:

- **Pruebas Estáticas de Seguridad (SAST)**: Analiza el código fuente en busca de vulnerabilidades utilizando Bandit y SonarQube.
- **Análisis de Composición de Código (SCA)**: Examina las dependencias en busca de vulnerabilidades conocidas utilizando pip-audit.
- **Pruebas Dinámicas de Seguridad (DAST)**: Prueba la aplicación en ejecución para detectar vulnerabilidades utilizando OWASP ZAP.
- **Seguridad en Infraestructura como Código (IaC)**: Escanea Dockerfiles y archivos Compose utilizando Trivy.
- **Pruebas Interactivas de Seguridad (IAST)**: Realiza análisis de código en tiempo de ejecución utilizando Semgrep.

Cada flujo de trabajo genera un informe en formato JSON o XML, que se sube como artefacto en GitHub Actions. Estos informes pueden importarse a DefectDojo si está expuesto a internet mediante ngrok, lo que permite a los flujos de trabajo enviar los resultados. El pipeline está diseñado para ser resistente, continuando con la ejecución incluso si DefectDojo no está disponible, con los informes siempre accesibles como artefactos.

## Flujos de Trabajo del Pipeline
El pipeline consta de cinco flujos de trabajo de GitHub Actions, cada uno enfocado en un aspecto específico de la seguridad:

1. **SAST (sast.yml)**:
   - **Herramientas**: Bandit (para código Python), SonarQube (para calidad y seguridad del código).
   - **Activador**: Push a la rama `main`.
   - **Salida**: `sast_report.json` (informe de Bandit).
   - **Detalles**: Escanea archivos Python, excluyendo directorios de pruebas, y genera un informe JSON de vulnerabilidades.

2. **SCA (sca.yml)**:
   - **Herramienta**: pip-audit.
   - **Activador**: Push a la rama `main` o programación diaria (`0 0 * * *`).
   - **Salida**: `sca_report.json`.
   - **Detalles**: Analiza `requirements.txt` en busca de dependencias vulnerables.

3. **DAST (dast.yml)**:
   - **Herramienta**: OWASP ZAP.
   - **Activador**: Push a la rama `main`.
   - **Salida**: `report_xml.xml`, `report_html.html`, `report_md.md`.
   - **Detalles**: Ejecuta una aplicación Flask localmente y realiza un escaneo base para identificar vulnerabilidades en tiempo de ejecución.

4. **IaC (iac.yml)**:
   - **Herramienta**: Trivy.
   - **Activador**: Push a la rama `main`, específicamente para cambios en `Dockerfile`, `docker/docker-compose.dev.yml` o el archivo del flujo de trabajo.
   - **Salida**: `trivy_report.json`.
   - **Detalles**: Escanea Dockerfiles y archivos Compose en busca de configuraciones erróneas y vulnerabilidades.

5. **IAST (iast.yml)**:
   - **Herramienta**: Semgrep.
   - **Activador**: Push a la rama `main`.
   - **Salida**: `semgrep_report.json`.
   - **Detalles**: Realiza análisis de código en tiempo de ejecución, excluyendo directorios de pruebas.

Cada flujo de trabajo está configurado para:
- Ejecutar análisis de seguridad con las herramientas especificadas.
- Generar un informe en formato JSON o XML.
- Subir el informe como artefacto en GitHub Actions.
- Intentar importar el informe a DefectDojo si está disponible a través de ngrok, continuando la ejecución si no lo está.

## Gestión de Artefactos
Los informes generados por cada flujo de trabajo se suben como artefactos en GitHub Actions. Estos artefactos están disponibles en la interfaz de GitHub Actions, en la sección de cada ejecución del flujo de trabajo. Los artefactos incluyen:

- **SAST**: `sast_report.json`
- **SCA**: `sca_report.json`
- **DAST**: `report_xml.xml`, `report_html.html`, `report_md.md`
- **IaC**: `trivy_report.json`
- **IAST**: `semgrep_report.json`

Para descargar un artefacto:
1. Ve a la pestaña **Actions** en el repositorio de GitHub.
2. Selecciona la ejecución del flujo de trabajo deseada.
3. En la sección **Artifacts**, haz clic en el nombre del artefacto para descargarlo.

Estos artefactos permiten revisar los resultados de los análisis incluso si DefectDojo no está configurado o disponible.

## Integración con DefectDojo
Los flujos de trabajo están configurados para importar los informes a **DefectDojo**, una plataforma de gestión de vulnerabilidades. Para que los flujos de trabajo puedan enviar los informes, DefectDojo debe estar expuesto a internet mediante **ngrok**, ya que una instancia local sin exposición pública (por ejemplo, `http://localhost:8080`) no es accesible desde GitHub Actions. Si DefectDojo no está disponible, los flujos de trabajo continúan ejecutándose y los informes se guardan como artefactos.

### Ejecutar DefectDojo
1. **Clonar el repositorio de DefectDojo**:
   ```bash
   git clone https://github.com/DefectDojo/django-DefectDojo.git
   cd django-DefectDojo
   ```

2. **Configurar el entorno**:
   - Instala Docker y Docker Compose.
   - Configura las variables de entorno en el archivo `.env` según la documentación de DefectDojo.

3. **Iniciar DefectDojo**:
   ```bash
   docker-compose up -d
   ```
   DefectDojo se ejecutará localmente, pero no será accesible desde GitHub Actions hasta que se exponga con ngrok.

4. **Obtener la contraseña del administrador**:
   ```bash
   sudo docker compose logs initializer | grep "Admin password:"
   ```
  Con esto obtendremos la contraseña para el usuario de admin.

5. **Obtener un token de API**:
   - Inicia sesión en DefectDojo a través de la URL de ngrok (ver siguiente sección).
   - Ve a **Configuración > API v2 > Generar token** y copia el token generado.

6. **Crear un producto**:
   - En DefectDojo, crea un nuevo producto (por ejemplo, `DevSecOps-Pipeline`).
   - Anota el **ID del producto**, que se encuentra en la URL del producto (por ejemplo, `/product/1` indica `1` como ID).

### Exponer DefectDojo con ngrok
Para que los flujos de trabajo de GitHub Actions puedan comunicarse con DefectDojo, debes exponerlo a internet usando **ngrok**:

1. **Instalar ngrok**:
   - Descarga e instala ngrok desde [ngrok.com](https://ngrok.com/download).
   - Autentica tu cuenta de ngrok:
     ```bash
     ngrok authtoken <tu-token-de-ngrok>
     ```

2. **Exponer DefectDojo**:
   - Con DefectDojo ejecutándose en `http://0.0.0.0:8080`, inicia ngrok:
     ```bash
     ngrok http 8080
     ```
   - ngrok generará una URL pública, por ejemplo, `https://abcd1234.ngrok.io`. Copia esta URL.

3. **Verificar la conexión**:
   - Accede a la URL de ngrok en tu navegador (por ejemplo, `https://abcd1234.ngrok.io`) para confirmar que DefectDojo es accesible.

### Actualizar Secretos de GitHub
Para que los flujos de trabajo se comuniquen con tu instancia de DefectDojo expuesta vía ngrok, actualiza los secretos en GitHub:

1. **Navega a los secretos del repositorio**:
   - En GitHub, ve a tu repositorio > **Settings** > **Secrets and variables** > **Actions** > **Secrets**.

2. **Añadir los siguientes secretos**:
   - `DEFECTDOJO_TOKEN`: El token de API generado en DefectDojo.
   - `DEFECTDOJO_URL`: La URL de ngrok (por ejemplo, `https://abcd1234.ngrok.io`).
   - `DEFECTDOJO_PRODUCT_ID`: El ID del producto creado en DefectDojo (por ejemplo, `1`).
   - (Opcional) `SONAR_TOKEN` y `SONAR_HOST_URL`: Para el flujo de trabajo SAST si usas SonarQube.

3. **Actualizar secretos**:
   - Cada vez que reinicies ngrok, se generará una nueva URL. Actualiza el secreto `DEFECTDOJO_URL` en GitHub con la nueva URL para mantener la conectividad.

## Visualización de Informes en DefectDojo
Una vez configurado DefectDojo con ngrok y actualizados los secretos, los informes generados por los flujos de trabajo se importarán automáticamente a DefectDojo:

1. **Ejecutar el pipeline**:
   - Realiza un push a la rama `main` para activar los flujos de trabajo.
   - Los flujos de trabajo ejecutarán los análisis y enviarán los informes a DefectDojo si está accesible a través de ngrok.

2. **Acceder a los informes en DefectDojo**:
   - Inicia sesión en DefectDojo a través de la URL de ngrok (por ejemplo, `https://abcd1234.ngrok.io`).
   - Ve a **Productos** > Selecciona el producto configurado (por ejemplo, `DevSecOps-Pipeline`).
   - En la pestaña **Engagements**, encontrarás un nuevo engagement para cada ejecución del pipeline (por ejemplo, `SAST-Pipeline-<run_id>`).
   - Haz clic en el engagement para ver los detalles del escaneo (por ejemplo, vulnerabilidades de Bandit, pip-audit, OWASP ZAP, Trivy o Semgrep).

3. **Si DefectDojo no está disponible**:
   - Si ngrok no está activo o la URL es incorrecta, los flujos de trabajo continuarán ejecutándose y los informes se subirán como artefactos en GitHub Actions.
   - Descarga los artefactos desde la interfaz de GitHub Actions para revisarlos manualmente.

## Prerrequisitos
- **GitHub**: Un repositorio con permisos para configurar GitHub Actions y secretos.
- **Docker**: Para ejecutar DefectDojo.
- **ngrok**: Para exponer DefectDojo a internet (requerido para la integración con GitHub Actions).
- **Python 3.9+**: Para las herramientas de los flujos de trabajo (instalado automáticamente por GitHub Actions).
- **Dependencias**: Un archivo `requirements.txt` para SCA y DAST, y archivos como `Dockerfile` para IaC.

## Instrucciones de Configuración
1. **Clonar el repositorio**:
   ```bash
   git clone <url-del-repositorio>
   cd <nombre-del-repositorio>
   ```

2. **Configurar los flujos de trabajo**:
   - Asegúrate de que los archivos `.github/workflows/*.yml` (sast.yml, sca.yml, dast.yml, iac.yml, iast.yml) estén en el repositorio.
   - Verifica que las herramientas (Bandit, pip-audit, OWASP ZAP, Trivy, Semgrep) sean compatibles con tu proyecto.

3. **Configurar DefectDojo con ngrok**:
   - Sigue las instrucciones en [Ejecutar DefectDojo](#ejecutar-defectdojo) y [Exponer DefectDojo con ngrok](#exponer-defectdojo-con-ngrok).

4. **Actualizar secretos de GitHub**:
   - Configura los secretos según [Actualizar Secretos de GitHub](#actualizar-secretos-de-github).

5. **Probar el pipeline**:
   - Realiza un push a la rama `main`:
     ```bash
     git add .
     git commit -m "Iniciar pipeline DevSecOps"
     git push origin main
     ```
   - Monitorea las ejecuciones en la pestaña **Actions** de GitHub.