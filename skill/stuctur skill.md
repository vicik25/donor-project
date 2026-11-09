---
name: layered-crud-architecture
description: Panduan arsitektur untuk membangun fitur CRUD (List, Detail, Add, Edit, Delete) dengan pemisahan lapisan Model, HTTP client, Service, Provider (state), dan Page triad (View/Ctrl/Data) plus Navigator terpusat. Bahasa dan framework bebas (Flutter, React, Vue, Kotlin, Swift, Angular, dll). Gunakan skill ini setiap kali user meminta membuat project CRUD baru, menambah entity/fitur CRUD baru, menambah halaman list/detail/add/edit, merapikan atau me-review struktur folder aplikasi yang memanggil REST API, atau menyebut "struktur crud", "domain service provider", "page view ctrl data", walaupun mereka tidak menyebut nama arsitekturnya.
---

# Layered CRUD Architecture

Pola ini diekstrak dari project referensi Flutter bernama `crud` (entity `User`, API `gorest.co.in`, state management `states_rebuilder`, routing `navigation_builder`). Polanya tidak bergantung bahasa. Dokumen ini memisahkan **aturan (wajib diikuti)** dari **detail implementasi (menyesuaikan bahasa)**.

Tujuan pola: setiap file hanya punya satu alasan untuk berubah. Agent yang menambah entity baru cukup menyalin bentuk yang sama, tanpa harus memutuskan ulang di mana sebuah kode diletakkan.

---

## 1. Lima lapisan

```
        ┌──────────────────────────────────────────────┐
 UI     │ pages/<feature>/<N>_<Screen>/                │
        │   View  : tampilan murni                     │
        │   Ctrl  : aksi / event handler               │
        │   Data  : pintu masuk state + form controller│
        └───────────────┬──────────────────────────────┘
                        │ (hanya lewat Data & Ctrl)
        ┌───────────────▼──────────────┐
 State  │ domains/<entity>/<entity>.prov│  state reaktif: list, detail,
        └───────────────┬──────────────┘  selectedId, input, create/update/delete
                        │ memanggil
        ┌───────────────▼──────────────┐
 Logic  │ domains/<entity>/<entity>.serv│  orkestrasi: panggil http → decode → Model
        └───────────────┬──────────────┘
                        │ memanggil
        ┌───────────────▼──────────────┐
 Infra  │ http client (satu file)       │  HTTP mentah, return body/status apa adanya
        └──────────────────────────────┘
 Model  │ domains/_models/<entity>      │  dipakai semua lapisan di atas
 Route  │ navigator (satu file)         │  tabel route terpusat + objek `nav`
```

### Aturan arah dependensi

1. View hanya mengimpor **Ctrl dan Data milik halaman itu sendiri**, plus widget/komponen UI. View tidak mengimpor Provider, Service, atau HTTP secara langsung.
2. Ctrl mengimpor Data (halaman sendiri), Provider, dan `nav`. Ctrl tidak membuat widget.
3. Data mengimpor Provider saja. Isinya alias state dan form controller.
4. Provider mengimpor Service. Service mengimpor HTTP client dan Model.
5. HTTP client tidak tahu Model, state, atau UI. Ia hanya mengembalikan string body atau status code.
6. Halaman tidak mengimpor file halaman lain. Kalau dua halaman butuh data yang sama, bagikan lewat Provider di `domains/`.

Alasannya: ganti library HTTP cukup menyentuh satu file, ganti tampilan tidak menyentuh logika, dan tiap lapisan bisa dites sendiri.

> **Catatan siklus impor.** Di project referensi, `serv` mengimpor `prov` (untuk memperbarui cache list setelah create/update/delete) sementara `prov` mengimpor `serv`. Itu siklus. Bahasa yang melarang impor siklik (Go, Rust, banyak setup TypeScript) tidak bisa memakainya. Versi bersih: Service **hanya mengembalikan hasil**, sedangkan sinkronisasi cache list dilakukan di Provider (callback sukses) atau di Ctrl. Gunakan versi bersih untuk project baru.

---

## 2. Layout folder (generik)

```
<project>/
└── src/                              # lib/ di Dart, src/ di TS/Kotlin, dst.
    ├── main                          # entry point, hanya menjalankan App
    ├── app                           # root: tema + router config
    ├── navigator                     # tabel route + objek `nav`
    ├── home                          # halaman awal, tombol menuju /list
    ├── domains/
    │   ├── _models/
    │   │   └── {entity}              # Model (underscore: bukan fitur, dipakai bersama)
    │   └── {entity}/
    │       ├── {entity}.prov         # Provider
    │       └── {entity}.serv         # Service
    ├── infra/http/http               # HTTP client  (referensi menaruhnya di pages/serch/http/)
    └── pages/
        └── {entities}/               # plural
            ├── 1_{Entity}List/       {Entity}.List.{View,Ctrl,Data}
            ├── 2_{Entity}Detail/     {Entity}.Detail.{View,Ctrl,Data}
            ├── 3_{Entity}Edit/       {Entity}.Edit.{View,Ctrl,Data}
            └── 4_{Entity}Add/        {Entity}.Add.{View,Ctrl,Data}
```

Placeholder: `{Entity}` = PascalCase tunggal (`User`), `{entity}` = lowercase tunggal (`user`), `{entities}` = lowercase jamak (`users`).

- Awalan `1_ … 4_` pada folder halaman menunjukkan urutan alur pengguna (list → detail → edit → add). Pertahankan.
- Tiap halaman **selalu tiga file**: `View`, `Ctrl`, `Data`. Tidak boleh digabung, tidak boleh kurang, walaupun salah satunya hanya beberapa baris.
- **Penamaan file mengikuti konvensi bahasa target**, bukan bentuk `User.List.View.dart` di referensi. Dart memakai `user_list_view.dart`, TypeScript `user-list.view.ts` atau `UserList.view.tsx`, Kotlin/Swift `UserListView.kt`/`.swift`. Yang dipertahankan adalah **struktur peran** (`entity + halaman + View|Ctrl|Data`), bukan tanda titiknya. Referensi sendiri tidak konsisten (`User.List.View.dart` vs `user.edit.ctrl.dart`), jangan ditiru.

---

## 3. Kontrak tiap file

### 3.1 Model (`domains/_models/{entity}`)

Kelas data murni, tanpa logika jaringan atau UI. Wajib punya:

| Anggota | Fungsi |
|---|---|
| konstruktor dengan **nilai default** (id `0`, string `''`) | membuat objek kosong untuk form dan state awal |
| `copyWith(...)` | salinan dengan sebagian field diganti |
| `toMap()` | semua field, termasuk `id` |
| `toMapWithoutId()` | untuk body POST/PUT, karena server menentukan id |
| `fromMap(map)` | field hilang atau null jatuh ke default, angka dibaca aman |
| `toJson()` / `fromJson(str)` | pembungkus encode/decode |
| `==`, `hashCode`, `toString` | perbandingan nilai dan debug |

Di bahasa yang punya data class (Kotlin `data class`, TS `interface` + fungsi, Python `dataclass`, Swift `Codable struct`) sebagian anggota ini otomatis. Tetap sediakan `toMapWithoutId` secara eksplisit.

### 3.2 HTTP client (`infra/http/http`)

Satu fungsi per operasi, tanpa pengetahuan domain:

```
getMany()            -> body string      GET    /{entities}
getSingle(id)        -> body string      GET    /{entities}/{id}
post(map)            -> body string      POST   /{entities}
put(map, id)         -> body string      PUT    /{entities}/{id}
delete(id)           -> status code      DELETE /{entities}/{id}
```

Aturan:
- Base URL dan token autentikasi **dibaca dari konfigurasi/environment**, tidak ditulis di source. (Lihat bagian 7, temuan #1.)
- Header autentikasi dipasang di satu tempat (helper), bukan disalin di tiap fungsi.
- Cek status code sebelum mengembalikan body. Status non-2xx harus melempar error yang bisa ditangkap lapisan state, supaya UI masuk ke kondisi error.
- Fungsi log boleh ada, tapi hanya aktif pada mode debug dan tidak mencetak token.

### 3.3 Service (`domains/{entity}/{entity}.serv`)

Fungsi bebas (atau kelas stateless) yang menjembatani HTTP dan Model:

```
readList()            -> List<Entity>      getMany → decode array → fromMap per elemen
readSingle(id)        -> Entity            getSingle → decode → fromMap
create(input?)        -> Entity?           null jika input null; post(toMapWithoutId) → fromMap
update(input?, id)    -> Entity?           null jika input null; put(toMapWithoutId, id) → fromMap
delete(id)            -> void | bool       delete(id) → sukses jika status 2xx (DELETE biasanya 204)
```

Service tidak menyentuh widget, navigasi, atau controller form.

### 3.4 Provider (`domains/{entity}/{entity}.prov`)

Kumpulan **penampung state reaktif** (dalam referensi diawali `rx`). Ada dua jenis:

| Jenis | Nama (contoh) | Isi |
|---|---|---|
| state sinkron | `selectedId` (int, 0) | id item yang sedang dibuka |
| state sinkron | `input` (Entity?, null) | data form yang akan dikirim |
| state async, **dibaca** | `list` (List<Entity>) | hasil `readList()`, dimuat saat pertama dipakai |
| state async, **dibaca** | `detail` (Entity) | hasil `readSingle(selectedId)` |
| state async, **perintah** | `create`, `update`, `delete` | menjalankan service saat di-refresh/di-trigger |

Setiap state async punya tiga kondisi: **waiting**, **error**, **data**. View memetakan ketiganya ke spinner, pesan error, dan konten.

**Pola perintah (command state)** untuk create/update/delete:
1. Ctrl mengisi `input` dengan objek hasil form.
2. Ctrl men-trigger state perintah (`create.refresh()`).
3. View mengamati state itu: waiting → spinner di tombol, selesai dengan data tidak null → efek samping (navigasi + bersihkan form).
4. Setelah sukses, cache `list` dan `detail` disinkronkan (lihat bagian 4).

### 3.5 Page triad

**Data** (`{Entity}.{Screen}.Data`): satu-satunya tempat halaman mengenal Provider.
- Alias state yang dipakai halaman (`listX = list`, `detailX = detail`, `selectedIdX = selectedId`). Akhiran `X` di referensi menandai alias; boleh diganti nama lain asal konsisten.
- Form controller (`name`, `email`, `gender`, `status`) untuk halaman Add dan Edit. Halaman Edit juga punya controller `id` yang `enabled: false`.

**Ctrl** (`{Entity}.{Screen}.Ctrl`): fungsi, bukan widget.
- List: `onTap(id)` → `nav.to('/detail')` dan set `selectedId`.
- Detail: `delete()` → panggil service/state delete, lalu refresh `list` dan `detail`.
- Edit: `prepare()` (di referensi bernama `reset`) mengisi semua controller dari `detail`; `submit()` membaca controller → `Entity` → `input` → trigger `update`.
- Add: `submit()` membaca controller → `input` → trigger `create`; `clear()` mengosongkan form; `fillRandom()` mengisi data acak untuk uji cepat (opsional).

**View** (`{Entity}.{Screen}.View`): hanya tampilan. Tidak ada `if` bisnis, tidak ada pemanggilan HTTP.
- List: AppBar, daftar kartu (`name` sebagai judul, `email` sebagai subjudul), FAB `+` menuju `/add`.
- Detail: AppBar dengan tombol hapus (dialog konfirmasi), isi field, FAB edit.
- Edit/Add: kolom input per field, tombol submit yang menampilkan spinner saat waiting.

### 3.6 Navigator, App, Main, Home

```
nav = Router.create(routes: {
  '/'       : Home
  '/list'   : {Entity}List
  '/detail' : {Entity}Detail
  '/edit'   : {Entity}Edit
  '/add'    : {Entity}Add
})
```

- `nav` adalah **satu objek global**. Navigasi dari Ctrl atau View selalu lewat `nav.to(path)`, `nav.back()`, `nav.backUntil(path)`. Tidak ada pemanggilan navigator bawaan framework di tempat lain.
- Id item tidak dikirim lewat parameter route. Ia disimpan di state `selectedId` sebelum pindah halaman.
- `app` memasang tema dan `nav` ke root. `main` hanya menjalankan `app`.
- Entity kedua menambah route dengan prefiks sendiri (`/products`, `/products/detail`, dst.), tidak menimpa route lama.

---

## 4. Alur data per operasi

**List** — `list` async dimuat saat halaman pertama dibuka → View menampilkan spinner → data → `ListView`. Tap item: Ctrl set `selectedId` lalu `nav.to('/detail')`.

**Detail** — `detail` memuat `readSingle(selectedId)`. Tombol hapus membuka dialog konfirmasi; ya → Ctrl menjalankan delete.

**Add** — form → `submit()` → `input` → trigger `create` → sukses: sisipkan item baru di **awal** `list`, `nav.backUntil('/list')`, `clear()` form.

**Edit** — tombol edit di Detail memanggil `prepare()` **sebelum** `nav.to('/edit')` supaya form terisi data terbaru → `submit()` → trigger `update` → sukses: ganti `detail` dengan hasil server, ganti elemen yang id-nya sama di `list` (jaga kasus index tidak ketemu), `nav.backUntil('/detail')`.

**Delete** — status sukses (2xx) → hapus item dari `list` secara lokal → `nav.backUntil('/list')`. Spinner dialog ditampilkan selama menunggu, lalu **ditutup setelah hasil diketahui**, bukan sebelum.

Prinsip sinkronisasi cache: setelah mutasi sukses, perbarui `list` dan `detail` **dari respons server** (bukan dari input form), supaya field yang diisi server (id, timestamp) benar.

---

## 5. Peta ke bahasa dan framework

Konsep tetap, nama komponen berubah:

| Konsep | Flutter (referensi) | React / Next | Vue / Nuxt | Android (Kotlin) | SwiftUI | Angular |
|---|---|---|---|---|---|---|
| Provider (state) | `states_rebuilder` injector, atau Riverpod/Bloc | Zustand / Redux Toolkit / TanStack Query | Pinia store | `ViewModel` + `StateFlow` | `ObservableObject` | service + signals |
| Service | fungsi di `*.serv.dart` | modul `*.service.ts` | `*.service.ts` | `Repository` | `Service` class | `@Injectable` |
| HTTP client | paket `http` | `fetch`/axios | `fetch`/ofetch | Retrofit/Ktor | `URLSession` | `HttpClient` |
| View | `StatelessWidget` | komponen `.tsx` | `.vue` SFC | `@Composable` | `View` | komponen template |
| Ctrl | fungsi top-level | fungsi/hook handler | `composable` | fungsi di ViewModel | method di ViewModel | method komponen |
| Data | file alias + `TextEditingController` | hook `use{Entity}ListData` + form state | `composable` | state holder halaman | `@State`/binding | `FormGroup` |
| Navigator | `nav` global (`navigation_builder`) | router + wrapper `nav` | vue-router + wrapper `nav` | Navigation Compose + wrapper | `NavigationStack` + wrapper | `Router` + wrapper |
| waiting/error/data | `OnBuilder.all` | `isLoading/isError/data` | `status` ref | sealed `UiState` | enum `LoadState` | signal `status` |

Aturan memilih:
- Jika project sudah memakai library state, **pakai library itu**. Jangan menambah library kedua.
- Jika project kosong, pilih yang idiomatik untuk framework (tabel di atas) dan sebutkan pilihannya ke user dalam satu kalimat.
- Untuk bahasa tanpa UI deklaratif (CLI, backend), padanan: View = handler/route, Ctrl = use-case, Service = repository, Model = DTO. Lapisan Provider boleh dihilangkan, lapisan lain tetap.
- Bentuk tetap: **tiga file per halaman, satu alur dependensi searah**.

---

## 6. Prosedur untuk agent

### 6.1 Menambah entity baru (mis. `Product`)

Kerjakan berurutan dari lapisan terbawah supaya tiap langkah bisa dikompilasi:

1. Tanyakan atau baca skema API: nama field, tipe, endpoint, autentikasi. Jika tidak ada, pakai asumsi paling umum dan sebutkan.
2. `domains/_models/product`: field + semua anggota di 3.1.
3. HTTP: tambah lima fungsi di 3.2 (atau satu helper generik berparameter path).
4. `product.serv`: lima fungsi di 3.3.
5. `product.prov`: `selectedId`, `input`, `list`, `detail`, `create`, `update`, `delete`.
6. Halaman `pages/products/1_ProductList … 4_ProductAdd`, masing-masing tiga file.
7. Daftarkan empat route di `navigator`; tambahkan tombol atau tautan masuk di `home`.
8. Jalankan analyzer/linter/compiler bahasa itu, lalu jalankan aplikasi atau tes dasar bila memungkinkan.

### 6.2 Membuat project baru

Buat kerangka (`main`, `app`, `navigator`, `home`) lebih dulu, lalu satu entity penuh lewat 6.1. Pastikan alur Home → List → Detail → Edit/Delete → Add berjalan sebelum menambah entity lain.

### 6.3 Mereview atau merapikan project yang ada

Cocokkan dengan bagian 1 (arah dependensi) dan bagian 7 (temuan). Laporkan pelanggaran dulu, perbaiki setelah user setuju bila perubahannya besar. Jangan mengganti state library atau struktur folder hanya karena selera.

### 6.4 Kebiasaan yang harus dijaga

- Jangan taruh logika bisnis atau panggilan jaringan di View.
- Jangan membuat Data berisi logika. Ia hanya alias dan controller.
- Jangan membagi satu halaman ke lebih dari tiga file, kecuali widget kecil yang dipakai ulang diletakkan di folder komponen bersama.
- Jangan menaruh rahasia (token, API key, password) di source atau commit.
- Satu perubahan, satu lapisan: jika sebuah tugas memaksa mengedit View dan HTTP sekaligus, periksa apakah ada Ctrl, Provider, atau Service yang terlewat.

---

## 7. Temuan di project referensi (jangan ditiru)

Project `crud` adalah project latihan, dan beberapa hal di dalamnya sebaiknya tidak dibawa ke project baru. Urut dari yang paling penting:

1. **Bearer token di-hardcode** di setiap fungsi `http.dart` dan ikut ter-commit di riwayat git. Token itu harus dianggap bocor: cabut dan buat ulang di penyedia API, lalu pindahkan ke environment/config (Flutter: `--dart-define` atau `flutter_dotenv`; web: variabel env build; mobile native: file konfigurasi yang di-gitignore). Hapus dari riwayat git jika repo pernah dipublikasikan.
2. **Pengecekan hasil delete salah**: kode membandingkan status dengan `'240'`, padahal DELETE sukses mengembalikan `204`. Akibatnya item tidak pernah dihapus dari list lokal. Pakai pemeriksaan rentang 2xx. Selain itu `deletx()` memanggil `delet()` tanpa `await` lalu langsung me-refresh dan menutup halaman, sehingga hasilnya belum tentu terbaca.
3. **Siklus impor `prov` ↔ `serv`** (lihat catatan di bagian 1).
4. **Form controller Edit dibuat saat file dimuat** dari `detail.state` (nilai basi), lalu diperbaiki dengan `reset()` sebelum navigasi. Isi controller secara eksplisit lewat `prepare()` dan jangan mengandalkan inisialisasi global.
5. **Antar-halaman saling impor**: View Detail mengimpor `User.List.Data` dan `user.edit.ctrl`. Ini melanggar aturan 1 dan 6. Detail seharusnya punya Data sendiri (dan memang ada `user.Detail.Data.dart`, tapi tidak dipakai View).
6. **Duplikasi alias**: `rxUserDetailX` didefinisikan di dua file Data; `userselec` menyimpan snapshot `selectedId.state` yang tidak ikut berubah.
7. **Tidak ada validasi form** dan `int.parse(idctrl.text)` bisa melempar error. Tambahkan validasi sebelum men-trigger perintah.
8. **Error ditampilkan sebagai teks `'error'`** tanpa pesan. Tampilkan pesan dari server dan sediakan tombol coba lagi.
9. **Penamaan tidak konsisten** (`delet`, `deletx`, `editx`, `createx`, `httpput`, folder `serch`, `respons`) dan campuran huruf besar-kecil pada nama file. Gunakan nama utuh dan satu konvensi.
10. **Isi folder platform ikut ke `lib/`** (`lib/linux/` berisi CMake/C++). Folder `lib/` hanya untuk kode Dart; file platform tetap di `linux/`, `android/`, `ios/`, dst.
11. **Artefak build ikut diarsipkan** (`build/`, `.dart_tool/`, `windows/flutter/ephemeral/`, total ratusan MB). Pastikan `.gitignore` dan arsip yang dibagikan mengecualikannya.

---

## 8. Contoh minimal (Dart/Flutter, bentuk bersih)

Hanya sebagai ilustrasi bentuk. Terjemahkan ke bahasa target, jangan salin mentah ke bahasa lain.

```dart
// infra/http/http.dart
Future<String> httpGetMany() async {
  final res = await http.get(Uri.parse('$baseUrl/users'), headers: authHeaders());
  if (res.statusCode < 200 || res.statusCode >= 300) throw HttpError(res.statusCode, res.body);
  return res.body;
}

// domains/user/user.serv.dart  (tidak mengimpor prov)
Future<List<User>> readList() async =>
    (jsonDecode(await httpGetMany()) as List).map((e) => User.fromMap(e)).toList();

// domains/user/user.prov.dart
final selectedId = RM.inject<int>(() => 0);
final userInput  = RM.inject<User?>(() => null);
final userList   = RM.injectFuture<List<User>>(() => readList());
final userDetail = RM.injectFuture<User>(() => readSingle(selectedId.state.toString()));
final userCreate = RM.injectFuture<User?>(() async {
  final created = await create(userInput.state);
  if (created != null) userList.state = [created, ...userList.state]; // sinkron cache di sini
  return created;
});

// pages/users/1_UserList/user_list_data.dart
final listX = userList;
final selectedIdX = selectedId;

// pages/users/1_UserList/user_list_ctrl.dart
void onTap(int id) { selectedIdX.state = id; nav.to('/detail'); }

// pages/users/1_UserList/user_list_view.dart
//   OnBuilder.all(listenTo: listX, onWaiting: spinner, onError: errorView, onData: listView)
```

---

## 9. Daftar periksa sebelum selesai

- [ ] Tiap halaman punya tepat tiga file `View`, `Ctrl`, `Data`.
- [ ] View tidak mengimpor Provider/Service/HTTP; halaman tidak mengimpor halaman lain.
- [ ] Tidak ada siklus impor antara Provider dan Service.
- [ ] Lima operasi (list, detail, add, edit, delete) berjalan, dan `list`/`detail` ter-update tanpa reload.
- [ ] Status sukses delete dicek dengan rentang 2xx.
- [ ] Spinner untuk waiting, pesan untuk error, konten untuk data, di semua halaman.
- [ ] Tidak ada token/rahasia di source; `.gitignore` mengecualikan folder build.
- [ ] Linter/analyzer/compiler lolos tanpa error.
- [ ] Route baru terdaftar di `navigator` dan bisa dicapai dari `home`.
