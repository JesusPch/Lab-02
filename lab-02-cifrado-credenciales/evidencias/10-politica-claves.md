### Política de Gestión y Ciclo de Vida de Claves Criptográficas

1. **Generación Criptográficamente Segura**
   Toda clave criptográfica debe generarse mediante un generador de números pseudoaleatorios criptográficamente seguro (CSPRNG) con una longitud mínima de 256 bits (AES-256 o equivalente), prohibiendo el uso de contraseñas de texto simple o algoritmos de generación débiles.

2. **Almacenamiento Aislado y Fuera de Código**
   Las claves criptográficas nunca deben escribirse directamente en el código fuente, scripts SQL, archivos de configuración de PostgreSQL ni repositorios de Git. Deben resguardarse exclusivamente en gestores de secretos dedicados (como HashiCorp Vault, AWS KMS o GCP Secret Manager) e inyectarse en tiempo de ejecución como variables de entorno.

3. **Acceso Basado en el Principio de Mínimo Privilegio**
   El acceso a las claves de cifrado debe limitarse de forma estricta a las aplicaciones y servicios que lo requieran para su operación. Todo intento de acceso a la clave debe autenticarse, autorizarse y registrarse en una bitácora de auditoría inmutable.

4. **Rotación Programada de Claves**
   Las claves de cifrado de datos (DEK) deben rotarse automáticamente en un periodo máximo de 12 meses (o 90 días en entornos de alto riesgo). Se debe implementar un procedimiento de re-cifrado para actualizar los datos almacenados con la nueva clave sin interrumpir la disponibilidad del servicio.

5. **Respaldo Cifrado y Distribuido**
   Los respaldos de las claves deben guardarse en un almacenamiento secundario físicamente separado de la base de datos principal, cifrados mediante una clave maestra (KEK) y protegidos bajo un esquema de custodia compartida (división de conocimiento).

6. **Revocación Inmediata por Emergencia**
   Ante cualquier evento confirmado o sospecha de brecha de seguridad, la clave comprometida debe ser revocada inmediatamente mediante un plan de respuesta a incidentes automatizado, sustituyéndola por una nueva clave limpia y re-cifrando los datos afectados.

7. **Monitoreo y Prevención de Exposición Accidental**
   Se deben integrar herramientas automáticas de escaneo de secretos (como *Gitleaks* o *Trufflehog*) en las canalizaciones de CI/CD para impedir *commits* con claves en texto claro, además de configurar el servidor PostgreSQL para enmascarar o evitar el registro de funciones de cifrado en los archivos de *log* y vistas de monitoreo (`pg_stat_activity`).

8. **Eliminación y Destrucción Segura (*Zeroization*)**
   Cuando una clave cumpla su ciclo de vida o los datos que protegía deban ser purgados, la clave debe destruirse de forma definitiva e irreversible mediante sobreescritura de memoria (*zeroization*) y eliminación verificada de todos sus respaldos asociados, garantizando el borrado criptográfico de la información.