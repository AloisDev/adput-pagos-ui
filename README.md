# ADPUT Pagos UI

Documentación técnica: [backend, migraciones y operación](../adput-pagos-backend/DOCUMENTACION_TECNICA.md).

Interfaz Vue responsive para socios y Administración de ADPUT.

Manual de uso: [socios y Administración](../adput-pagos-backend/MANUAL_DE_USO.md).

## Uso actual

- Socios: inicio de sesión, cuotas, saldo, vencimientos, perfil y contraseña.
- Administración: configuración de cuota mensual futura, cargos extra primero en borrador y aplicados solo tras confirmación, extracto de cobranzas, filtros por fecha/situación/búsqueda, paginación y exportación XLSX.
- Los filtros de fecha y situación se aplican automáticamente; la búsqueda espera 400 ms después de la última tecla.

La API se ejecuta en `http://127.0.0.1:3000` y Vite reenvía `/api` a ese servicio durante el desarrollo.

## Ejecutar

```powershell
pnpm install
pnpm dev
```

Abrir `http://127.0.0.1:5173`.

## Compartir por ngrok

Con el backend y el frontend locales en ejecución:

```powershell
ngrok http 5173
```

Solo se expone el frontend: el proxy de Vite mantiene la API en el equipo local. Si ngrok entrega un dominio nuevo y Vite lo bloquea, agregalo temporalmente antes de iniciar Vite:

```powershell
$env:__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS = "tu-dominio.ngrok-free.dev"
pnpm dev
```

Usar datos de demostración al compartir un túnel público.

## Validación

```powershell
pnpm build
```

La documentación completa de arquitectura, datos, seguridad y próximos opcionales está en el [README del backend](../adput-pagos-backend/README.md).
