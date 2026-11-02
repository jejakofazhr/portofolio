# FigCraft Integration

Project Portfolio ini telah terhubung dengan **FigCraft** yang berlokasi di `D:\artcraft\FigCraft`.

## 1. Integrasi MCP Server (AI Assistant)
FigCraft telah didaftarkan pada konfigurasi MCP Antigravity (`~/.gemini/config/mcp_config.json`).
AI Assistant dapat langsung menjalankan perintah headless FigCraft untuk membaca, menginspeksi struktur layer, dan mengekspor aset visual ke dalam direktori project ini.

## 2. Dev Mode Code Connect (`figcraft.connect.json`)
File [figcraft.connect.json](file:///d:/1mycode/Portfolio/figcraft.connect.json) memetakan komponen desain FigCraft ke komponen Svelte di project ini:
- `Nav` -> `src/lib/components/Nav.svelte`
- `Hero` -> `src/lib/components/Hero.svelte`
- `About` -> `src/lib/components/About.svelte`
- `Work` -> `src/lib/components/Work.svelte`
- `Testimonials` -> `src/lib/components/Testimonials.svelte`
- `Footer` -> `src/lib/components/Footer.svelte`
- `Trail` -> `src/lib/components/Trail.svelte`

Untuk mempublikasikan koneksi komponen ke file `.figcraft`:
```bash
npm run figcraft:connect -- path/to/design.figcraft
```

## 3. NPM Helper Scripts
Di [package.json](file:///d:/1mycode/Portfolio/package.json) telah ditambahkan script:
- `npm run figcraft` : Membuka aplikasi FigCraft GUI (`figcraft.exe`)
- `npm run figcraft:cli -- <command>` : Menjalankan FigCraft CLI (contoh: `export`, `render`, `info`)
- `npm run figcraft:connect -- <file.figcraft>` : Menghubungkan kode komponen ke file desain FigCraft
- `npm run figcraft:info -- <file.figcraft>` : Menampilkan informasi nodes, frame, dan style dari file desain
