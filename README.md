# Litro Vegas

Casino web interactivo desarrollado con Vue 3, TypeScript y Vite. Permite jugar a varios juegos de azar desde una misma interfaz, con saldo compartido entre partidas y persistencia local en el navegador.

## Juegos disponibles

- **BlackJack**: acércate a 21 sin pasarte y vence al crupier.
- **Carrera de caballos**: elige un caballo, apuesta y comprueba quién llega primero a la meta.
- **Tragaperras**: introduce fichas y tira de la palanca para intentar conseguir un premio o el jackpot.
- **Ruleta**: apuesta por números, colores, docenas, columnas y apuestas anunciadas antes de girar la ruleta.

Todos los juegos utilizan el mismo saldo virtual. El saldo inicial es de **1.000 €** y se guarda en `localStorage`, por lo que se conserva al recargar la página en el mismo navegador. Cuando el saldo llega a cero, puede reiniciarse desde la cabecera.

> Este proyecto utiliza únicamente dinero virtual y no realiza apuestas reales.

## Requisitos

- [Node.js](https://nodejs.org/) `22.18+` o `24.12+`
- [pnpm](https://pnpm.io/)

## Instalación

Clona el repositorio y ejecuta:

```sh
pnpm install
```

## Desarrollo

Inicia el servidor de desarrollo con recarga en caliente:

```sh
pnpm dev
```

Después, abre la URL que muestre Vite, normalmente `http://localhost:5173`.

## Scripts disponibles

| Comando | Descripción |
| --- | --- |
| `pnpm dev` | Inicia el servidor de desarrollo. |
| `pnpm build` | Comprueba los tipos y genera la compilación de producción. |
| `pnpm build-only` | Genera únicamente la compilación de producción. |
| `pnpm preview` | Sirve localmente la compilación generada. |
| `pnpm type-check` | Ejecuta la comprobación de tipos con `vue-tsc`. |
| `pnpm test:unit` | Ejecuta las pruebas unitarias con Vitest. |
| `pnpm format` | Formatea el código de `src/` con Prettier. |

Para validar el proyecto antes de distribuirlo:

```sh
pnpm build
```

## Tecnologías

- [Vue 3](https://vuejs.org/) con `<script setup>`
- [TypeScript](https://www.typescriptlang.org/)
- [Vite](https://vite.dev/)
- [Pinia](https://pinia.vuejs.org/) para el saldo compartido
- [Vue Router](https://router.vuejs.org/) para la navegación
- [Vitest](https://vitest.dev/) y Vue Test Utils para pruebas

## Estructura del proyecto

```text
src/
├── assets/       # Estilos específicos de cada pantalla
├── components/   # Menú, cabecera y componentes de los juegos
├── composables/  # Lógica reutilizable, como las apuestas
├── data/         # Configuración de los juegos disponibles
├── models/       # Tipos de dominio
├── router/       # Rutas de la aplicación
├── stores/       # Estado global, incluido el saldo
└── utils/        # Utilidades para fichas y apuestas
public/
└── ...           # Cartas, caballos y otros recursos estáticos
```

## Añadir un juego

1. Crea el componente del juego en `src/components/`.
2. Añade su ruta en `src/router/index.ts`.
3. Registra el juego en `src/data/juegos.ts` para que aparezca en el menú.
4. Añade los estilos y recursos estáticos que necesite.
5. Ejecuta `pnpm build` para comprobar tipos y compilación.

## Licencia

Este proyecto no declara actualmente una licencia de distribución.
