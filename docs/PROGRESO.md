# DockScan — Progreso

> Estado vivo del proyecto. La referencia estable está en [ARQUITECTURA.md](ARQUITECTURA.md).
> **Última actualización:** 2026-11-02

## Estado actual

Repositorio creado, `main` protegida (solo por PR, verificado con un push rechazado). Todavía no hay código.

## Roadmap (cortes verticales; cada uno funciona y se sube antes del siguiente)

| # | Corte | Estado |
|---|---|---|
| 1 | Repositorio, documentación y protección de `main` | ✅ Hecho |
| 2 | CI con GitHub Actions | ⏳ Siguiente |
| 3 | Base de datos Supabase: esquema, RLS, RPC y datos de prueba inventados | ⏳ |
| 4 | Esqueleto Android con Hilt (verificar compatibilidad de versiones antes) | ⏳ |
| 5 | Login (Google y correo) | ⏳ |
| 6 | Escáner (barcode + OCR, reutilizando lo ya validado) | ⏳ |
| 7 | Recepción de compra con colores y cuadre | ⏳ |
| 8 | Tiempo real entre celulares | ⏳ |
| 9 | Cambio de equipo, repotenciación y fotos | ⏳ |

## Decisiones tomadas

- **Nombre:** DockScan (sin referencias a ninguna institución; el repositorio es público).
- **Cuenta de GitHub:** JoseMoreno7. Identidad de Git fijada solo en este repositorio, con correo noreply.
- **Versiones:** siempre la última versión **estable** (nunca alfa, beta ni RC), verificada contra fuentes oficiales antes de fijarla.
- **Guías de compra:** normalmente traen serie por equipo; las líneas por cantidad (granel) se tratarán con un código interno asignado por la app.
- **Terminal:** PowerShell. Archivos con acentos se leen con `Get-Content -Encoding utf8`.

## Decisiones abiertas

1. Paquete definitivo (hoy `dev.morenojos.dockscan`): confirmar antes de crear el proyecto Android.
2. Proyecto de Supabase (nombre y región) en la cuenta personal.
3. Cuenta de Google Cloud para el inicio de sesión con Google.
4. Celular físico para probar el escáner con etiquetas reales.
5. Cómo identificar el modelo cuando una guía trae solo cantidades por modelo (se define al diseñar la función de escaneo).