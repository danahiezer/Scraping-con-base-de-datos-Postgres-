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

---

## Notas

- Los autores son ficticios y se asignan aleatoriamente ya que el sitio no tiene datos de autores reales
- El scraping puede tardar varios minutos porque entra a la URL de cada libro para obtener la categoría (1000 peticiones adicionales)
- El archivo `libros.json` sirve como respaldo por si algo falla en la inserción, así no hay que volver a scrapear todo