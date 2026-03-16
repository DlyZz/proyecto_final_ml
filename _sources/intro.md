# Análisis de la Plataforma Steam — Proyecto de Machine Learning

## Contexto

Steam es la plataforma de distribución de videojuegos PC más grande del mundo. Este proyecto analiza datos reales recolectados de Steam, PlayStation y Xbox, trabajando exclusivamente con el subconjunto de Steam, que comprende nueve archivos con información de ~424k usuarios, ~97k juegos, precios en 5 monedas, historial de logros desbloqueados, reseñas, listas de amigos y bibliotecas de juegos comprados.

El dataset cubre un período de actividad desde 2008 hasta 2024, con snapshots de precios concentrados en noviembre–diciembre 2024.

---

## Estructura del proyecto

El análisis se organiza en tres etapas:

**1. Exploración (EDA)** — un notebook por cada archivo fuente, donde se documentan la calidad de los datos, patrones relevantes, decisiones de limpieza y variables derivadas. Cada EDA exporta su dataframe limpio como CSV.

**2. Feature Engineering** — un notebook que carga los CSVs de los EDAs y construye mediante joins las dos tablas maestras que alimentan los modelos.

---

## Archivos del dataset

| Archivo | Descripción | Filas aprox. |
|---|---|---|
| `games` | Catálogo de juegos con género, publisher, fecha de lanzamiento | 97k juegos |
| `achievements` | Logros disponibles en cada juego | ~2M logros |
| `history` | Logros desbloqueados por usuarios (timestamp exacto) | ~47k usuarios |
| `players` | Usuarios con país y fecha de creación de cuenta | 424k usuarios |
| `friends` | Lista de amigos por usuario | 642 MB |
| `purchased_games` | Biblioteca de juegos por usuario | 89 MB |
| `prices` | Historial de precios en USD, EUR, GBP, JPY, RUB | 179 MB |
| `reviews` | Reseñas escritas por jugadores | ~1.2M reseñas |
| `private_steamids` | IDs con perfil privado | 227k IDs |

---

## Enfoques de Machine Learning

### Modelo 1 — Regresión: ¿Cuántos logros desbloqueará un usuario en un juego?

**Pregunta:** dado un par (usuario, juego), ¿qué tan profundo llegará el usuario en ese juego?

**Target:** `n_logros_par` — número de logros desbloqueados por el usuario en ese juego, calculado desde `history` excluyendo farming automatizado. Como variable derivada se calcula también `completion_rate = n_logros_par / n_logros_totales_del_juego`.

**Por qué este target:** el dataset no contiene horas jugadas. Los logros desbloqueados son el mejor proxy disponible de engagement real — capturan tanto el tiempo invertido como la intención activa de jugar.

**Features:**

| Grupo | Variables |
|---|---|
| Usuario | `account_age_years`, `country`, `is_private`, `n_friends_log`, `social_tier`, `lib_size`, `user_type` |
| Juego | `primary_genre`, `release_year`, `n_achievements`, `is_spam_game`, `usd_log`, `price_tier`, `n_reviews_log`, `avg_engagement` |

---

### Modelo 2 — Clasificación: ¿Será este juego popular?

**Pregunta:** dadas las características de un juego antes o al momento de su lanzamiento, ¿llegará al top 25% de juegos más comprados?

**Target:** `is_popular` — variable binaria, `True` si `n_owners_public` ≥ percentil 75 de la distribución. Se usa `n_owners_public` (solo usuarios con perfil público) para evitar el sesgo de inflación que introducen los perfiles privados.

**Por qué este target:** popularidad medida por compradores reales es más honesta que ratings declarados o ventas reportadas por publishers. El umbral P75 genera un split 75/25 útil para clasificación.

**Features:**

| Grupo | Variables |
|---|---|
| Características del juego | `primary_genre`, `release_year`, `genres_count`, `supported_languages_count` |
| Precio | `usd_log`, `price_tier`, `has_price`, `is_f2p`, `no_eur_region` |
| Logros | `n_achievements`, `is_spam_game`, `pct_generic` |
| Reseñas | `n_reviews_log`, `avg_engagement`, `pct_with_text` |
| Engagement | `avg_completion_rate` (promedio de logros completados por jugador) |

---

## Relación entre los dos modelos

Los modelos son complementarios y permiten una conclusión de negocio con tensión real:

> *Un juego puede ser popular (muchos compradores) sin ser muy enganchador (pocos logros completados), y viceversa. La intersección — juegos populares con alto completion rate — identifica los títulos que el mercado valora y que los jugadores realmente terminan.*

