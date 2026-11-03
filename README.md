# DockScan

App Android para **recepcionar equipos comprados y registrar cambios de equipo** escaneando códigos, en lugar de verificar y anotar a mano.

> Estado: en desarrollo temprano. Este repositorio documenta el proceso paso a paso.

## Qué resuelve

- **Recepción de compra:** se carga la guía de compra, se escanea cada equipo (código de barras u OCR del número de serie) y la app responde al instante con un color: recibido, duplicado, no pertenece a la compra o pendiente.
- **Cambio de equipo:** se escanea el equipo que sale y el que entra, y se guarda todo como un único movimiento con trazabilidad.

## Stack

Kotlin, Jetpack Compose, MVVM, Hilt, CameraX, ML Kit (barcode y OCR), Supabase (Postgres, Auth, Realtime, Storage, RLS).

## Documentación

- [Arquitectura](docs/ARQUITECTURA.md): qué es el proyecto, el stack, la estructura y las reglas de trabajo.
- Progreso: `docs/PROGRESO.md` (próximamente).

## Datos

Todos los datos de ejemplo son sintéticos e inventados.