# Prompt para Setear un Vault Local de Obsidian

Usa este prompt con un agente que tenga acceso local a tu vault de Obsidian.

````text
Quiero que inspecciones y dejes seteado mi vault local de Obsidian según esta arquitectura.

Regla canónica:

El usuario captura y organiza su vida en `inbox/`, `daily/` y PARA. Solo cuando decide que una fuente puede ser leída por IA, la mueve o copia a `raw/`. Librarian nunca procesa `inbox/`, `daily/` ni PARA directamente.

Objetivo:

Dejar el vault organizado con una capa humana clara y una capa de IA separada. `raw/` debe funcionar como frontera explícita de consentimiento para procesamiento con IA.

Estructura objetivo:

```text
vault/
  home.md
  inbox/
  daily/
  1-proyectos/
  2-areas/
  3-recursos/
  4-archivo/
  raw/
  wiki/
  reports/
  reviews/
  configs/
  memory/
  templates/
  .librarian/
  _assets/
```

Significado de cada carpeta:

- `inbox/`: captura humana temporal. Librarian nunca la lee directamente.
- `daily/`: notas diarias humanas. No las procesa Librarian por defecto.
- `1-proyectos/`, `2-areas/`, `3-recursos/`, `4-archivo/`: capa PARA humana.
- `raw/`: fuentes inmutables aprobadas explícitamente para procesamiento con IA.
- `wiki/`: conocimiento estructurado generado o curado por Librarian.
- `reviews/`: propuestas legibles para revisión humana.
- `reports/`: diagnósticos del vault generados bajo demanda.
- `configs/`: reglas visibles y editables para Librarian.
- `memory/`: continuidad del agente entre sesiones.
- `.librarian/`: estado técnico interno, índices, propuestas, cache y locks.
- `templates/`: templates de Obsidian.
- `_assets/`: adjuntos, imágenes y binarios. No es carpeta de fuentes para IA.

Tareas:

1. Inspecciona la estructura actual del vault.
2. Reporta qué carpetas existen y cuáles faltan.
3. Detecta si existe alguna estructura vieja como:
   - `raw/inbox/`
   - `raw/daily/`
   - `raw/1-proyectos/`
   - `raw/2-areas/`
   - `raw/3-recursos/`
   - `raw/4-archivo/`
4. Si existen esas carpetas viejas, proponé moverlas a la raíz del vault:
   - `raw/inbox/` → `inbox/`
   - `raw/daily/` → `daily/`
   - `raw/1-proyectos/` → `1-proyectos/`
   - `raw/2-areas/` → `2-areas/`
   - `raw/3-recursos/` → `3-recursos/`
   - `raw/4-archivo/` → `4-archivo/`
5. Si existe `raw/assets/`, tratalo como corpus legacy congelado: no lo muevas, no lo renombres, no lo borres y no lo proceses automáticamente salvo instrucción explícita.
6. Si existe una carpeta accidental tipo `Users/`, inspeccionala y preguntame antes de moverla o borrarla.
7. Crea las carpetas faltantes de la estructura objetivo, pero no borres nada.
8. Configura o verifica Daily Notes para que use:
   - new file location: `daily`
   - template file location: `templates/daily-template`
   - date format: `YYYY-MM-DD`
9. Verifica que `raw/` contenga solo fuentes aprobadas para IA o subcarpetas de fuentes aprobadas.
10. Verifica que `inbox/`, `daily/` y PARA estén fuera de `raw/`.

Reglas de seguridad:

- No borres archivos ni carpetas.
- No sobrescribas archivos existentes.
- No muevas contenido si hay conflicto de nombres.
- Si hay conflicto, detenete y preguntame.
- No muevas contenido de `inbox/` a `raw/` automáticamente.
- No muevas contenido de `daily/` o PARA a `raw/` automáticamente.
- No proceses contenido con IA. Esta tarea es solo de estructura local.
- No toques `raw/assets/` si existe; es una excepción legacy que requiere aprobación explícita.
- No modifiques notas existentes salvo que sea necesario para crear `_README.md` explicativos, y preguntame antes.

Antes de hacer cambios, mostrame un plan con:

- carpetas encontradas
- carpetas faltantes
- movimientos propuestos
- posibles conflictos
- qué quedaría dentro de `raw/`
- qué configuración de Obsidian vas a tocar

Esperá mi aprobación antes de ejecutar movimientos.

Después de ejecutar los cambios, entregame un resumen final con:

- estructura final del vault
- movimientos realizados
- carpetas creadas
- conflictos encontrados, si hubo
- configuración de Daily Notes verificada
- lista de cosas que requieren revisión humana
````
