


## Tutorial : Membuat API GraphQL *Star Wars* untuk Pemula

Selamat datang, Padawan! Tutorial ini akan memandu kamu membangun API GraphQL bertema *Star Wars* menggunakan **FastAPI**, **Ariadne**, dan **SQLite**. Kita akan membuat API untuk mengelola data karakter, planet, dan kapal luar angkasa, lengkap dengan fitur membaca (query) dan mengubah (mutation) data. Tutorial ini dirancang untuk pemula, dengan penjelasan sederhana dan langkah-langkah yang mudah diikuti.

### Apa Itu GraphQL?
Bayangkan GraphQL sebagai asisten yang membantu kamu mengambil *tepat* data yang kamu inginkan dari server. Tidak seperti REST, yang sering memberikan terlalu banyak atau terlalu sedikit data, GraphQL memungkinkan kamu meminta informasi spesifik—misalnya, nama Luke Skywalker, planet asalnya, dan kapal yang dia piloti—dalam satu permintaan. Ini efisien, fleksibel, dan cocok untuk galaksi data yang kompleks seperti *Star Wars*.

### Apa yang Akan Kita Bangun?
Kita akan membuat API yang memungkinkan pengguna:
- **Membaca** data tentang karakter (misalnya, Luke Skywalker), planet (Tatooine), dan kapal (Millennium Falcon).
- **Mengubah** data dengan menambah, memperbarui, atau menghapus entitas.
- Menjelajahi relasi, seperti karakter yang tinggal di planet tertentu atau kapal yang dipiloti oleh karakter.

### Persiapan
Sebelum memulai, pastikan kamu memiliki:
- **Python 3.8+** terinstal (cek dengan `python --version`).
- Editor kode seperti **VS Code**.
- Pengetahuan dasar Python (variabel, fungsi, dictionary).
- Terminal (Command Prompt, PowerShell, atau Terminal di macOS/Linux).

### Langkah 1: Menyiapkan Proyek 
Kita akan membuat folder proyek dan lingkungan virtual untuk menjaga semuanya rapi, seperti markas rahasia di Yavin IV.

1. **Buat dan masuk ke folder proyek:**
   ```bash
   mkdir starwars-graphql-api
   cd starwars-graphql-api
   ```

2. **Buat lingkungan virtual dan aktifkan:**
   ```bash
   python -m venv venv
   source venv/bin/activate  # Windows: venv\Scripts\activate
   ```

3. **Instal dependensi:**
   Kita akan menggunakan **FastAPI** (server), **Uvicorn** (menjalankan server), **Ariadne** (GraphQL), dan **SQLAlchemy** (opsional untuk masa depan, tapi kita tetap pakai `sqlite3` bawaan untuk pemula).
   ```bash
   pip install fastapi uvicorn ariadne
   ```

4. **Buat struktur proyek:**
   Buat file-file berikut di folder `starwars-graphql-api`:
   ```
   starwars-graphql-api/
   ├── venv/
   ├── main.py           # Logika utama FastAPI dan GraphQL
   ├── schema.graphql    # Definisi skema GraphQL
   ├── resolvers.py      # Fungsi untuk menangani query/mutation
   ├── database.py       # Setup dan koneksi database
   ├── seed.py           # Mengisi database dengan data Star Wars
   └── starwars.db       # Database SQLite (dibuat otomatis nanti)
   ```

### Langkah 2: Membuat Database SQLite 
Kita menggunakan **SQLite**, database ringan yang sempurna untuk pemula, untuk menyimpan data seperti arsip di Kuil Jedi.

1. **Buat file `database.py`:**
   ```python
   import sqlite3

   DATABASE_NAME = "starwars.db"

   def get_db_connection():
       """Membuka koneksi ke database dengan row factory untuk akses kolom berdasarkan nama."""
       conn = sqlite3.connect(DATABASE_NAME)
       conn.row_factory = sqlite3.Row
       return conn

   def init_db():
       """Membuat tabel jika belum ada."""
       conn = get_db_connection()
       c = conn.cursor()

       # Tabel planets
       c.execute("""
           CREATE TABLE IF NOT EXISTS planets (
               id INTEGER PRIMARY KEY AUTOINCREMENT,
               name TEXT UNIQUE NOT NULL,
               climate TEXT,
               terrain TEXT
           )
       """)

       # Tabel characters
       c.execute("""
           CREATE TABLE IF NOT EXISTS characters (
               id INTEGER PRIMARY KEY AUTOINCREMENT,
               name TEXT UNIQUE NOT NULL,
               species TEXT,
               home_planet_id INTEGER,
               FOREIGN KEY (home_planet_id) REFERENCES planets (id)
           )
       """)

       # Tabel starships
       c.execute("""
           CREATE TABLE IF NOT EXISTS starships (
               id INTEGER PRIMARY KEY AUTOINCREMENT,
               name TEXT UNIQUE NOT NULL,
               model TEXT,
               manufacturer TEXT
           )
       """)

       # Tabel penghubung characters-starships (many-to-many)
       c.execute("""
           CREATE TABLE IF NOT EXISTS character_starships (
               character_id INTEGER,
               starship_id INTEGER,
               PRIMARY KEY (character_id, starship_id),
               FOREIGN KEY (character_id) REFERENCES characters (id),
               FOREIGN KEY (starship_id) REFERENCES starships (id)
           )
       """)

       conn.commit()
       conn.close()
       print("Tabel berhasil dibuat.")

   if __name__ == "__main__":
       init_db()
   ```

   **Penjelasan:**
   - Fungsi `get_db_connection()` memastikan koneksi konsisten.
   - `conn.row_factory = sqlite3.Row` memungkinkan akses data seperti `row["name"]`.
   - Tabel-tabel mendukung relasi (foreign keys) dan hubungan many-to-many.
   - Kolom `name` di setiap tabel adalah `UNIQUE` untuk mencegah duplikasi.
   - Fungsi `init_db()` membuat tabel saat file dijalankan langsung.

2. **Jalankan `database.py` untuk membuat tabel:**
   ```bash
   python database.py
   ```

### Langkah 3: Mengisi Database 
Kita akan membuat file `seed.py` untuk mengisi database dengan data *Star Wars* yang menarik.

1. **Buat file `seed.py`:**
   ```python
   from database import get_db_connection, init_db

   def seed_data():
       """Mengisi database dengan data awal Star Wars."""
       conn = get_db_connection()
       c = conn.cursor()

       # Bersihkan data lama
       c.execute("DELETE FROM character_starships")
       c.execute("DELETE FROM characters")
       c.execute("DELETE FROM starships")
       c.execute("DELETE FROM planets")
       conn.commit()

       # Data planet
       planets = [
           ("Tatooine", "Arid", "Desert"),
           ("Alderaan", "Temperate", "Grasslands, Mountains"),
           ("Yavin IV", "Temperate, Humid", "Jungle, Rainforests"),
           ("Naboo", "Temperate", "Grassy Hills, Swamps"),
           ("Coruscant", "Temperate", "Cityscape"),
       ]
       c.executemany("INSERT INTO planets (name, climate, terrain) VALUES (?, ?, ?)", planets)
       planet_ids = {row["name"]: row["id"] for row in c.execute("SELECT id, name FROM planets").fetchall()}

       # Data karakter
       characters = [
           ("Luke Skywalker", "Human", planet_ids["Tatooine"]),
           ("Leia Organa", "Human", planet_ids["Alderaan"]),
           ("Han Solo", "Human", None),
           ("C-3PO", "Droid", None),
           ("Yoda", "Unknown", None),
       ]
       c.executemany("INSERT INTO characters (name, species, home_planet_id) VALUES (?, ?, ?)", characters)
       character_ids = {row["name"]: row["id"] for row in c.execute("SELECT id, name FROM characters").fetchall()}

       # Data kapal
       starships = [
           ("Millennium Falcon", "YT-1300 light freighter", "Corellian Engineering"),
           ("X-wing", "T-65 X-wing starfighter", "Incom Corporation"),
           ("TIE Fighter", "TIE/LN starfighter", "Sienar Fleet Systems"),
       ]
       c.executemany("INSERT INTO starships (name, model, manufacturer) VALUES (?, ?, ?)", starships)
       starship_ids = {row["name"]: row["id"] for row in c.execute("SELECT id, name FROM starships").fetchall()}

       # Relasi karakter-kapal
       character_starships = [
           (character_ids["Han Solo"], starship_ids["Millennium Falcon"]),
           (character_ids["Luke Skywalker"], starship_ids["X-wing"]),
       ]
       c.executemany("INSERT INTO character_starships (character_id, starship_id) VALUES (?, ?)", character_starships)

       conn.commit()
       conn.close()
       print("Database berhasil diisi dengan data Star Wars!")

   if __name__ == "__main__":
       init_db()  # Pastikan tabel ada
       seed_data()
   ```

2. **Jalankan `seed.py`:**
   ```bash
   python seed.py
   ```
   File `starwars.db` sekarang berisi data awal.

   **Peningkatan:**
   - Data lebih terfokus untuk menjaga tutorial sederhana.
   - Menggunakan dictionary comprehension untuk ID mapping.
   - Membersihkan data lama sebelum seeding untuk menghindari duplikasi.

### Langkah 4: Membuat Skema GraphQL 
Skema GraphQL adalah peta yang mendefinisikan data apa yang bisa diminta dan struktur hasilnya.

1. **Buat file `schema.graphql`:**
   ```graphql
   type Query {
     allCharacters: [Character!]!
     character(id: ID!): Character
     allPlanets: [Planet!]!
     planet(id: ID!): Planet
     allStarships: [Starship!]!
     starship(id: ID!): Starship
   }

   type Mutation {
     createPlanet(input: CreatePlanetInput!): Planet
     updatePlanet(input: UpdatePlanetInput!): Planet
     deletePlanet(id: ID!): Boolean
     createCharacter(input: CreateCharacterInput!): Character
     assignStarship(input: AssignStarshipInput!): Character
   }

   input CreatePlanetInput {
     name: String!
     climate: String
     terrain: String
   }

   input UpdatePlanetInput {
     id: ID!
     name: String
     climate: String
     terrain: String
   }

   input CreateCharacterInput {
     name: String!
     species: String
     homePlanetId: Int
   }

   input AssignStarshipInput {
     characterId: ID!
     starshipId: ID!
   }

   type Character {
     id: ID!
     name: String!
     species: String
     homePlanet: Planet
     pilotedStarships: [Starship!]!
   }

   type Planet {
     id: ID!
     name: String!
     climate: String
     terrain: String
     residents: [Character!]!
   }

   type Starship {
     id: ID!
     name: String!
     model: String
     manufacturer: String
     pilots: [Character!]!
   }
   ```

   **Penjelasan:**
   - **Query**: Mendefinisikan cara mengambil data (misalnya, `allCharacters` atau `character(id: ID!)`).
   - **Mutation**: Mendefinisikan cara mengubah data (misalnya, `createPlanet`).
   - **Input Types**: Menyederhanakan struktur input untuk mutation.
   - **Types**: `Character`, `Planet`, `Starship` mendefinisikan struktur data dengan relasi (misalnya, `homePlanet` atau `pilotedStarships`).
   - `!` menandakan field wajib (non-null).

   **Peningkatan:**
   - Mutation `deletePlanet` mengembalikan `Boolean` untuk kesederhanaan.
   - Hanya menyertakan mutation esensial untuk fokus pada pembelajaran.

### Langkah 5: Membuat Resolver 
Resolver adalah fungsi Python yang mengambil data dari database saat query atau mutation dipanggil.

1. **Buat file `resolvers.py`:**
   ```python
   from ariadne import QueryType, MutationType, ObjectType
   from database import get_db_connection

   query = QueryType()
   mutation = MutationType()
   character_type = ObjectType("Character")
   planet_type = ObjectType("Planet")
   starship_type = ObjectType("Starship")

   # --- Query Resolvers ---
   @query.field("allCharacters")
   def resolve_all_characters(_, info):
       conn = get_db_connection()
       try:
           characters = conn.execute("SELECT id, name, species, home_planet_id FROM characters").fetchall()
           return [dict(char) for char in characters]
       finally:
           conn.close()

   @query.field("character")
   def resolve_character(_, info, id):
       conn = get_db_connection()
       try:
           character = conn.execute("SELECT id, name, species, home_planet_id FROM characters WHERE id = ?", (id,)).fetchone()
           return dict(character) if character else None
       finally:
           conn.close()

   @query.field("allPlanets")
   def resolve_all_planets(_, info):
       conn = get_db_connection()
       try:
           planets = conn.execute("SELECT id, name, climate, terrain FROM planets").fetchall()
           return [dict(p) for p in planets]
       finally:
           conn.close()

   @query.field("planet")
   def resolve_planet(_, info, id):
       conn = get_db_connection()
       try:
           planet = conn.execute("SELECT id, name, climate, terrain FROM planets WHERE id = ?", (id,)).fetchone()
           return dict(planet) if planet else None
       finally:
           conn.close()

   @query.field("allStarships")
   def resolve_all_starships(_, info):
       conn = get_db_connection()
       try:
           starships = conn.execute("SELECT id, name, model, manufacturer FROM starships").fetchall()
           return [dict(s) for s in starships]
       finally:
           conn.close()

   @query.field("starship")
   def resolve_starship(_, info, id):
       conn = get_db_connection()
       try:
           starship = conn.execute("SELECT id, name, model, manufacturer FROM starships WHERE id = ?", (id,)).fetchone()
           return dict(starship) if starship else None
       finally:
           conn.close()

   # --- Nested Resolvers ---
   @character_type.field("homePlanet")
   def resolve_character_home_planet(character_obj, info):
       home_planet_id = character_obj.get("home_planet_id")
       if not home_planet_id:
           return None
       conn = get_db_connection()
       try:
           planet = conn.execute("SELECT id, name, climate, terrain FROM planets WHERE id = ?", (home_planet_id,)).fetchone()
           return dict(planet) if planet else None
       finally:
           conn.close()

   @character_type.field("pilotedStarships")
   def resolve_character_piloted_starships(character_obj, info):
       character_id = character_obj.get("id")
       conn = get_db_connection()
       try:
           starships = conn.execute(
               """
               SELECT s.id, s.name, s.model, s.manufacturer
               FROM starships s
               JOIN character_starships cs ON s.id = cs.starship_id
               WHERE cs.character_id = ?
               """,
               (character_id,),
           ).fetchall()
           return [dict(s) for s in starships]
       finally:
           conn.close()

   @planet_type.field("residents")
   def resolve_planet_residents(planet_obj, info):
       planet_id = planet_obj.get("id")
       conn = get_db_connection()
       try:
           characters = conn.execute(
               "SELECT id, name, species, home_planet_id FROM characters WHERE home_planet_id = ?",
               (planet_id,),
           ).fetchall()
           return [dict(char) for char in characters]
       finally:
           conn.close()

   @starship_type.field("pilots")
   def resolve_starship_pilots(starship_obj, info):
       starship_id = starship_obj.get("id")
       conn = get_db_connection()
       try:
           characters = conn.execute(
               """
               SELECT c.id, c.name, c.species, c.home_planet_id
               FROM characters c
               JOIN character_starships cs ON c.id = cs.character_id
               WHERE cs.starship_id = ?
               """,
               (starship_id,),
           ).fetchall()
           return [dict(char) for char in characters]
       finally:
           conn.close()

   # --- Mutation Resolvers ---
   @mutation.field("createPlanet")
   def resolve_create_planet(_, info, input):
       conn = get_db_connection()
       try:
           conn.execute(
               "INSERT INTO planets (name, climate, terrain) VALUES (?, ?, ?)",
               (input["name"], input.get("climate"), input.get("terrain")),
           )
           conn.commit()
           planet_id = conn.execute("SELECT last_insert_rowid()").fetchone()[0]
           planet = conn.execute("SELECT id, name, climate, terrain FROM planets WHERE id = ?", (planet_id,)).fetchone()
           return dict(planet)
       except sqlite3.IntegrityError:
           conn.rollback()
           raise Exception(f"Planet '{input['name']}' sudah ada.")
       finally:
           conn.close()

   @mutation.field("updatePlanet")
   def resolve_update_planet(_, info, input):
       conn = get_db_connection()
       try:
           planet = conn.execute("SELECT id, name, climate, terrain FROM planets WHERE id = ?", (input["id"],)).fetchone()
           if not planet:
               raise Exception(f"Planet dengan ID {input['id']} tidak ditemukan.")
           conn.execute(
               "UPDATE planets SET name = ?, climate = ?, terrain = ? WHERE id = ?",
               (
                   input.get("name", planet["name"]),
                   input.get("climate", planet["climate"]),
                   input.get("terrain", planet["terrain"]),
                   input["id"],
               ),
           )
           conn.commit()
           updated_planet = conn.execute("SELECT id, name, climate, terrain FROM planets WHERE id = ?", (input["id"],)).fetchone()
           return dict(updated_planet)
       except sqlite3.IntegrityError:
           conn.rollback()
           raise Exception("Nama planet sudah digunakan.")
       finally:
           conn.close()

   @mutation.field("deletePlanet")
   def resolve_delete_planet(_, info, id):
       conn = get_db_connection()
       try:
           planet = conn.execute("SELECT id FROM planets WHERE id = ?", (id,)).fetchone()
           if not planet:
               raise Exception(f"Planet dengan ID {id} tidak ditemukan.")
           residents = conn.execute("SELECT COUNT(*) FROM characters WHERE home_planet_id = ?", (id,)).fetchone()[0]
           if residents > 0:
               raise Exception(f"Tidak dapat menghapus planet dengan {residents} penduduk.")
           conn.execute("DELETE FROM planets WHERE id = ?", (id,))
           conn.commit()
           return True
       finally:
           conn.close()

   @mutation.field("createCharacter")
   def resolve_create_character(_, info, input):
       conn = get_db_connection()
       try:
           if input.get("homePlanetId"):
               planet = conn.execute("SELECT id FROM planets WHERE id = ?", (input["homePlanetId"],)).fetchone()
               if not planet:
                   raise Exception(f"Planet dengan ID {input['homePlanetId']} tidak ditemukan.")
           conn.execute(
               "INSERT INTO characters (name, species, home_planet_id) VALUES (?, ?, ?)",
               (input["name"], input.get("species"), input.get("homePlanetId")),
           )
           conn.commit()
           char_id = conn.execute("SELECT last_insert_rowid()").fetchone()[0]
           character = conn.execute(
               "SELECT id, name, species, home_planet_id FROM characters WHERE id = ?", (char_id,)
           ).fetchone()
           return dict(character)
       except sqlite3.IntegrityError:
           conn.rollback()
           raise Exception(f"Karakter '{input['name']}' sudah ada.")
       finally:
           conn.close()

   @mutation.field("assignStarship")
   def resolve_assign_starship(_, info, input):
       conn = get_db_connection()
       try:
           character = conn.execute("SELECT id FROM characters WHERE id = ?", (input["characterId"],)).fetchone()
           starship = conn.execute("SELECT id FROM starships WHERE id = ?", (input["starshipId"],)).fetchone()
           if not character:
               raise Exception(f"Karakter dengan ID {input['characterId']} tidak ditemukan.")
           if not starship:
               raise Exception(f"Kapal dengan ID {input['starshipId']} tidak ditemukan.")
           conn.execute(
               "INSERT OR IGNORE INTO character_starships (character_id, starship_id) VALUES (?, ?)",
               (input["characterId"], input["starshipId"]),
           )
           conn.commit()
           character = conn.execute(
               "SELECT id, name, species, home_planet_id FROM characters WHERE id = ?",
               (input["characterId"],),
           ).fetchone()
           return dict(character)
       finally:
           conn.close()

   resolvers = [query, mutation, character_type, planet_type, starship_type]
   ```

   - Menggunakan `try...finally` untuk memastikan koneksi ditutup.
   - Error handling spesifik untuk `IntegrityError` (duplikasi nama).
   - Validasi relasi (misalnya, cek planet sebelum assign).
   - Mengembalikan `Boolean` untuk `deletePlanet` agar lebih sederhana.

### Langkah 6: Mengintegrasikan FastAPI 
FastAPI akan menjadi pusat operasi API kita, seperti *Executor*-class Star Destroyer.

1. **Buat file `main.py`:**
   ```python
   from fastapi import FastAPI
   from ariadne import load_schema_from_path, make_executable_schema
   from ariadne.asgi import GraphQL
   from database import init_db
   from resolvers import resolvers

   app = FastAPI(title="Star Wars GraphQL API")

   type_defs = load_schema_from_path("schema.graphql")
   schema = make_executable_schema(type_defs, resolvers)
   graphql_app = GraphQL(schema, debug=True)

   app.mount("/graphql", graphql_app)

   @app.on_event("startup")
   async def startup_event():
       init_db()
       print("API siap! Akses GraphiQL di http://localhost:8000/graphql")

   @app.get("/")
   async def root():
       return {"message": "Selamat datang di Star Wars GraphQL API! Buka /graphql untuk mulai."}
   ```

2. **Jalankan server:**
   ```bash
   uvicorn main:app --reload
   ```

   Buka `http://localhost:8000/graphql` untuk mengakses **GraphiQL**, antarmuka interaktif untuk menguji API.

   **Peningkatan:**
   - Dokumentasi otomatis FastAPI di `/docs`.
   - Pesan sambutan di endpoint `/`.

### Langkah 7: Menguji API (Menjelajahi Galaksi)
Gunakan GraphiQL untuk menguji query dan mutation.

#### Contoh Query:
1. **Ambil semua karakter:**
   ```graphql
   query {
     allCharacters {
       id
       name
       species
       homePlanet {
         name
       }
     }
   }
   ```

2. **Ambil planet tertentu:**
   ```graphql
   query {
     planet(id: "1") {
       id
       name
       climate
       residents {
         name
       }
     }
   }
   ```

#### Contoh Mutation:
1. **Buat planet baru:**
   ```graphql
   mutation {
     createPlanet(input: { name: "Mustafar", climate: "Hot", terrain: "Volcanic" }) {
       id
       name
       climate
     }
   }
   ```

2. **Tambah karakter dan assign kapal:**
   ```graphql
   mutation {
     createCharacter(input: { name: "Mace Windu", species: "Human", homePlanetId: 5 }) {
       id
       name
     }
   }
   ```
   ```graphql
   mutation {
     assignStarship(input: { characterId: "6", starshipId: "2" }) {
       id
       name
       pilotedStarships {
         name
       }
     }
   }
   ```


### Penutup
Selamat, kamu telah membangun API GraphQL *Star Wars*! Kamu sekarang bisa membaca dan mengubah data galaksi dengan kekuatan GraphQL. Eksperimenlah dengan query dan mutation baru, dan jelajahi dokumentasi skema di GraphiQL.

**Tugas 3 Berikutnya:**
- Tambahkan mutation untuk update/delete karakter dan kapal.



## Glosarium Istilah API GraphQL

| Istilah                 | Penjelasan Singkat                                                                                                                               | Contoh dalam Tutorial                                                                                                                                                                      |
| :---------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **GraphQL**             | Bahasa query untuk API yang memungkinkan klien meminta *tepat* data yang mereka butuhkan, tidak lebih, tidak kurang, dalam satu permintaan.         | Meminta nama karakter, planet asalnya, dan kapal yang dipilotinya sekaligus.                                                                                                               |
| **API** (Application Programming Interface) | Cara bagi berbagai perangkat lunak untuk berkomunikasi dan bertukar informasi.                                                      | API GraphQL kita memungkinkan aplikasi lain (seperti website atau aplikasi mobile) untuk mengakses data.                                                                                  |
| **FastAPI**             | Kerangka kerja web Python modern dan cepat untuk membangun API.                                                                                    | Digunakan untuk menjalankan server API kita dan menangani permintaan HTTP.                                                                                                                  |
| **Ariadne**             | Pustaka Python yang membantu mengimplementasikan server GraphQL, menghubungkan skema dengan resolver.                                               | Digunakan untuk membuat skema GraphQL bisa dieksekusi dan menghubungkannya ke fungsi Python (resolver).                                                                                |
| **Uvicorn**             | Server ASGI (Asynchronous Server Gateway Interface) berperforma tinggi yang digunakan untuk menjalankan aplikasi FastAPI.                            | Perintah `uvicorn main:app --reload` menjalankan server kita.                                                                                                                                |
| **SQLite**              | Sistem manajemen database berbasis file yang ringan, tidak memerlukan server terpisah, dan cocok untuk proyek kecil hingga menengah atau prototipe. | File `starwars.db` tempat kita menyimpan data karakter, planet, dan kapal.                                                                                                                 |
| **Skema (Schema)**      | Kontrak antara klien dan server yang mendefinisikan semua tipe data, query, dan mutation yang tersedia di API GraphQL.                             | File `schema.graphql` yang berisi definisi `type Query`, `type Mutation`, `type Character`, `type Planet`, `type Starship`, dan `input` types.                                        |
| **Tipe (Type)**         | Mendefinisikan struktur data tertentu, termasuk field-field yang dimilikinya dan tipe data dari setiap field.                                    | `type Character { id: ID!, name: String!, species: String }`                                                                                                                               |
| **Query**               | Operasi untuk *membaca* atau mengambil data dari server.                                                                                         | `query { allCharacters { name } }` atau `planet(id: "1") { name climate }`                                                                                                                   |
| **Mutation**            | Operasi untuk *mengubah* data di server (membuat, memperbarui, atau menghapus).                                                                    | `mutation { createPlanet(input: {name: "Dagobah"}) { id name } }`                                                                                                                           |
| **Resolver**            | Fungsi Python yang bertanggung jawab untuk mengambil data (untuk query) atau melakukan perubahan data (untuk mutation) untuk field tertentu dalam skema. | Fungsi `resolve_all_characters()` atau `resolve_create_planet()` di file `resolvers.py`.                                                                                                    |
| **Nested Resolver**     | Resolver untuk field di dalam sebuah tipe, yang dipanggil ketika field tersebut diminta. Memungkinkan pengambilan data relasional.                     | `resolve_character_home_planet()` untuk mengambil data `homePlanet` dari `Character`.                                                                                                        |
| **Field**               | Bagian dari sebuah Tipe yang menyimpan nilai data.                                                                                               | `name`, `climate`, `terrain` adalah field dari `type Planet`.                                                                                                                               |
| **Argumen (Argument)**  | Nilai yang diberikan ke sebuah field atau mutation untuk memfilter atau menyediakan data.                                                        | `id: ID!` pada query `character(id: ID!)` atau `input` pada mutation `createPlanet(input: CreatePlanetInput!)`.                                                                           |
| **Input Type**          | Tipe khusus yang digunakan untuk mengelompokkan argumen-argumen yang kompleks untuk mutation, menjaga agar mutation tetap bersih.                  | `input CreatePlanetInput { name: String!, climate: String }`                                                                                                                                |
| **GraphiQL**            | Alat interaktif berbasis browser yang memungkinkan pengembang untuk menulis, menguji, dan menjelajahi API GraphQL.                               | Antarmuka yang muncul saat kita membuka `http://localhost:8000/graphql`.                                                                                                                     |
| **Lingkungan Virtual (venv)** | Folder terisolasi yang berisi instalasi Python dan pustaka-pustaka khusus untuk proyek tertentu, mencegah konflik antar proyek.                | Folder `venv/` yang kita buat di awal.                                                                                                                                                     |
| **`seed.py`**           | Skrip Python yang kita buat untuk mengisi database dengan data awal agar API kita punya sesuatu untuk ditampilkan dan diuji.                      | File `seed.py` yang berisi data awal.                                                                                                                                                    |
| **`database.py`**       | Skrip Python yang bertanggung jawab untuk membuat koneksi ke database dan mendefinisikan/membuat struktur tabel (skema database).               | File `database.py` dengan fungsi `get_db_connection()` dan `init_db()`.                                                                                                                    |
| **`conn.row_factory = sqlite3.Row`** | Pengaturan pada koneksi SQLite yang memungkinkan kita mengakses kolom data berdasarkan namanya (seperti dictionary, misal `row['name']`) daripada hanya indeks numerik. | Digunakan di `get_db_connection()` di `database.py`.                                                                                                                                     |
| **Foreign Key**         | Kolom dalam satu tabel yang merujuk ke primary key di tabel lain, menciptakan hubungan antar tabel.                                                | `home_planet_id` di tabel `characters` merujuk ke `id` di tabel `planets`.                                                                                                                   |
| **Many-to-Many Relationship** | Jenis hubungan antar dua entitas di mana satu entitas dapat berhubungan dengan banyak entitas lain, dan sebaliknya. Biasanya diimplementasikan dengan tabel perantara/penghubung. | Hubungan antara `characters` dan `starships` melalui tabel `character_starships`.                                                                                                           |
| **`try...finally`**     | Konstruksi pemrograman yang memastikan blok kode `finally` selalu dieksekusi, baik terjadi error di blok `try` maupun tidak. Berguna untuk membersihkan resource. | Digunakan di `resolvers.py` untuk memastikan koneksi database (`conn.close()`) selalu ditutup.                                                                                             |
| **`sqlite3.IntegrityError`** | Jenis error yang muncul di SQLite (dan database lain) ketika ada pelanggaran batasan integritas, seperti mencoba memasukkan nilai duplikat ke kolom yang `UNIQUE`. | Ditangani di resolver mutation untuk mencegah pembuatan planet atau karakter dengan nama yang sudah ada.                                                                                   |
| **cURL**                | Alat baris perintah untuk mentransfer data dengan URL. Bisa digunakan untuk mengirim permintaan HTTP ke API kita dari terminal.                 | `curl -X POST http://localhost:8000/graphql ...`                                                                                                                                          |
| **Postman**             | Aplikasi populer untuk merancang, membangun, menguji, dan mendokumentasikan API. Menyediakan antarmuka grafis untuk mengirim permintaan HTTP.       | Disebut sebagai alternatif untuk menguji API selain GraphiQL dan cURL.                                                                                                                    |

---

