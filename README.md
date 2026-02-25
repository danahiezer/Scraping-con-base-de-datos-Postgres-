# 📚 Scraping de Books to Scrape + PostgreSQL

Proyecto para scrapear libros del sitio [books.toscrape.com](https://books.toscrape.com) y guardar los datos en una base de datos PostgreSQL.

---

## ¿Qué hace este proyecto?

Básicamente scrapea los 1000 libros del sitio, extrae datos de cada uno, los guarda en un archivo JSON y luego los inserta en una base de datos PostgreSQL con varias tablas relacionadas entre sí.

---

## Estructura de la base de datos

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│     AUTORES     │    │  LIBROS_AUTORES  │    │     LIBROS      │
├─────────────────┤    │   (Tabla Pivot)  │    ├─────────────────┤
│ id (PK)         │◄───┤ autor_id (FK)    │───►│ id (PK)         │
│ nombre          │    │ libro_id (FK)    │    │ titulo          │
└─────────────────┘    └──────────────────┘    │ precio          │
                                               │ rating          │
┌─────────────────┐                            │ categoria_id(FK)│
│   CATEGORIAS    │◄───────────────────────────│ en_stock        │
├─────────────────┤                            │ url             │
│ id (PK)         │                            └─────────────────┘
│ nombre          │
└─────────────────┘
```

**Relaciones:**
- `AUTORES` ↔ `LIBROS`: Muchos a Muchos (un autor puede tener varios libros y un libro puede tener varios autores)
- `CATEGORIAS` → `LIBROS`: Uno a Muchos (una categoría tiene muchos libros)

---

## Datos que se scrapean por libro

- Título
- Precio (se limpia el símbolo £ antes de insertar)
- Rating (se convierte de texto a número: One→1, Two→2, etc.)
- Categoría (se obtiene entrando a la URL de cada libro)
- Stock
- URL del libro
- Autor (asignado aleatoriamente de una lista de 6 nombres)

---

## Librerías necesarias

```bash
pip install requests beautifulsoup4 psycopg2
```

---

## ¿Cómo se usa?

**1. Crear las tablas en PostgreSQL**

Correr la celda de creación de tablas. Solo hace falta hacerlo una vez.

**2. Correr el scraping**

Se scrapean las 50 páginas del sitio usando 5 hilos en paralelo para que sea más rápido. Al terminar genera un archivo `libros.json` con todos los datos.

**3. Insertar los datos en la base de datos**

Lee el archivo JSON e inserta los datos en las tablas respetando el orden: primero autores, luego categorías, luego libros y finalmente la tabla intermedia `libros_autores`. Los autores y categorías no se duplican gracias a un SELECT previo que verifica si ya existen.

**4. Crear los índices**

Se crean tres índices para optimizar las consultas más comunes. Solo se crean una vez gracias al `IF NOT EXISTS`.

- `idx_libros_categoria` sobre `libros(categoria_id)` → acelera las búsquedas por categoría
- `idx_autores_nombre` sobre `autores(nombre)` → acelera las búsquedas por nombre de autor
- `idx_libros_autores_autor` sobre `libros_autores(autor_id)` → acelera los JOINs entre autores y libros

**5. Correr las consultas**

Hay dos tipos de consultas para ver la diferencia entre hacer una búsqueda con y sin índice:

*Sin índice (Sequential Scan)* — PostgreSQL recorre fila por fila toda la tabla. En 1000 libros no se nota tanto, pero en bases de datos grandes sería muy lento.

```sql
-- Libros con precio menor a £15
SELECT titulo, precio FROM libros
WHERE precio < 15.00
ORDER BY precio LIMIT 5;

-- Libros con rating 5 estrellas
SELECT titulo, precio, rating FROM libros
WHERE rating = 5
ORDER BY precio LIMIT 3;
```

*Con índice (Index Scan)* — PostgreSQL navega directo al resultado usando el árbol B-Tree del índice, sin revisar registros innecesarios.

```sql
-- Libros de una categoría específica
SELECT l.titulo, c.nombre AS categoria
FROM libros l
JOIN categorias c ON c.id = l.categoria_id
WHERE l.categoria_id = 1 LIMIT 3;

-- Buscar autor por nombre
SELECT id, nombre FROM autores
WHERE nombre = 'María García';

-- Todos los libros de un autor (JOIN de 3 tablas)
SELECT a.nombre, l.titulo
FROM autores a
JOIN libros_autores la ON a.id = la.autor_id
JOIN libros l ON l.id = la.libro_id
WHERE a.nombre = 'María García';
```

---

## Notas

- Los autores son ficticios y se asignan aleatoriamente ya que el sitio no tiene datos de autores reales
- El scraping puede tardar varios minutos porque entra a la URL de cada libro para obtener la categoría (1000 peticiones adicionales)
- El archivo `libros.json` sirve como respaldo por si algo falla en la inserción, así no hay que volver a scrapear todo
- El rating no tiene índice porque solo tiene 5 valores posibles (1 al 5), lo que hace que un índice B-Tree tenga poco sentido ahí

