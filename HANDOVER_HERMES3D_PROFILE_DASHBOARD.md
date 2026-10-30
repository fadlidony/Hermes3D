# Handover: Hermes3D Profile Dashboard Integration

Waktu handover: 2026-10-30 12:26 UTC
Repo lokal: `/opt/hermes3d`
Branch awal: `main`
Remote: `https://github.com/iamlukethedev/Hermes3D.git`

## Tujuan pekerjaan

Membuat pengelolaan profil Hermes Agent dari dashboard web Hermes3D benar-benar bekerja dan persisten:

- membuat dan menghapus profil;
- menyimpan nama tampilan, SOUL/persona, file profil, skills, tools, approvals, model, dan reasoning;
- menampilkan katalog model Hermes yang sebenarnya, bukan pilihan palsu `hermes`;
- menjaga operasi destruktif aman dan tidak melaporkan sukses palsu.

## Kondisi penting

Semua perubahan masih berada di working tree `/opt/hermes3d` dan belum menjadi commit fitur. Jangan menjalankan `git reset --hard`, `git clean`, checkout paksa, atau membuang working tree.

Status file saat handover:

```text
 M server/hermes-agent/bridge.js
 M tests/unit/hermesAgentBridge.test.ts
?? server/hermes-agent/agent-files.js
?? server/hermes-agent/api.js
?? server/hermes-agent/profiles.js
?? server/hermes-agent/settings.js
?? tests/unit/hermesAgentFilesPersistence.test.ts
?? tests/unit/hermesAgentSettingsPersistence.test.ts
```

## Yang sudah selesai

### 1. Create profile

- `agents.create` di adapter Hermes3D diterjemahkan ke RPC native `profiles.create`.
- Path konfigurasi/workspace diambil dari roster backend Hermes, bukan filesystem host Studio.
- Duplicate profile ditolak.
- Roster diperbarui setelah create.
- Create telah diverifikasi secara live dan persisten setelah reconnect.

### 2. Delete profile

- Dibuat `server/hermes-agent/profiles.js`.
- Delete diteruskan ke endpoint native Hermes `DELETE /api/profiles/{name}`.
- Profil `default` dilindungi.
- Identifier target harus persis; input seperti `" luna "` tidak dinormalisasi menjadi `luna`.
- Profil yang memiliki run aktif tidak boleh dihapus.
- Shared deletion marker mencegah chat/delete race.
- Adapter membaca ulang roster dan hanya melaporkan sukses jika profil benar-benar hilang.
- Delete telah diverifikasi secara live.

### 3. Profile settings dan files

Dibuat:

- `server/hermes-agent/api.js`: authenticated native HTTP JSON client.
- `server/hermes-agent/agent-files.js`: persistence file profil.
- `server/hermes-agent/settings.js`: translasi settings dashboard ke native Hermes.

File yang dikelola:

- `AGENTS.md`
- `SOUL.md`
- `IDENTITY.md`
- `USER.md`
- `TOOLS.md`
- `HEARTBEAT.md`
- `MEMORY.md`

Catatan:

- `SOUL.md` memakai `profiles.configure`/`profiles.describe`.
- `USER.md` dan `MEMORY.md` dipetakan ke `memories/USER.md` dan `memories/MEMORY.md`.
- Nama tampilan disimpan pada `ui_meta.hermes3d`, tanpa mengganti canonical profile ID.
- Skills disimpan melalui `disabled_skills`.
- Tool groups dashboard dipetakan ke native `enabled_toolsets`:
  - `group:runtime` -> `terminal`
  - `group:fs` -> `file`
  - `group:web` -> `web`, `browser`
- Model/reasoning ditulis melalui native config/profile APIs dan diverifikasi lewat read-back.
- Unsupported/stale mutation ditolak, bukan dibalas sukses palsu.

### 4. Pengujian yang sebelumnya lulus

Sebelum pekerjaan katalog model terakhir:

- 185 related tests lulus pada 12 test files.
- 66/66 targeted file-persistence tests lulus.
- ESLint pada file terkait bersih.
- `git diff --check` bersih.
- Live settings round-trip melalui dashboard berhasil memakai profil sementara, lalu profil uji dibersihkan.
- `hermes3d.service` dan `hermes3d-backend.service` aktif; health endpoint pernah mengembalikan HTTP 200.

Full suite lama menghasilkan 1.226 lulus dan 2 gagal. Kedua kegagalan juga ada pada baseline sebelum perubahan, jadi bukan regresi fitur ini.

## Pekerjaan terakhir: katalog model

### Root cause yang ditemukan

`models.list` di `server/hermes-agent/bridge.js` membaca `result.options`, padahal RPC native `model.options` mengembalikan katalog dalam `result.providers[]`, dengan model pada `provider.models[]`.

Karena hasil parsing selalu kosong, adapter menggantinya dengan fallback palsu:

```text
{ id: "hermes", name: "hermes" }
```

Itulah sebabnya dropdown dashboard hanya menampilkan satu pilihan `hermes`.

Probe native disimpan di:

```text
/root/.hermes/cache/scratch/probe-hermes3d-models.cjs
/root/.hermes/cache/scratch/hermes3d-native-models.json
```

Probe mengonfirmasi backend mengembalikan beberapa provider dan model sebenarnya. Jangan menyalin credential/session token dari environment atau konfigurasi ke log/commit.

### Perubahan katalog model yang sudah diterapkan

`models.list` sekarang:

- membaca `result.providers`;
- mengubah setiap `provider.models` menjadi opsi `{id, name, provider}`;
- membuang `unavailable_models`;
- deduplicate berdasarkan pasangan provider + model;
- meneruskan capability `reasoning` bila tersedia;
- tidak lagi menciptakan fallback `hermes`;
- mengembalikan error jujur bila katalog kosong/tidak tersedia.

Tests baru sudah membuktikan:

- provider/model catalog berhasil dipetakan;
- model unavailable dibuang;
- kegagalan API tidak lagi menghasilkan model palsu.

### Status test terakhir

Command:

```bash
npm run test -- --run --maxWorkers=1 --no-file-parallelism \
  tests/unit/hermesAgentBridge.test.ts \
  -t 'native model catalogue' --testTimeout=15000
```

Hasil:

- 2 test lulus;
- 1 test gagal;
- kegagalan yang tersisa adalah session hydration masih mengembalikan `modelProvider: "hermes"` dan tidak mengembalikan model profil sebenarnya.

Expected:

```text
modelProvider: openai-codex
model: gpt-6-luna
```

Actual:

```text
modelProvider: hermes
model: undefined
```

## Langkah berikutnya yang harus dikerjakan

### A. Selesaikan session model/provider hydration

Lokasi utama:

- `server/hermes-agent/bridge.js:189-214`, fungsi `toHermes3dAgents`;
- `server/hermes-agent/bridge.js:734-753`, handler `sessions.list`.

Perubahan yang diperlukan:

1. Pada `toHermes3dAgents`, simpan provider dari profile roster:

```js
provider: asString(p.provider),
```

`model: asString(p.model)` sudah ada.

2. Pada main session di `sessions.list`, ganti hard-code:

```js
modelProvider: "hermes"
```

menjadi data agent:

```js
model: agent.model || undefined,
modelProvider: agent.provider || undefined,
```

3. Pada stored sessions, gunakan data session jika native API mengembalikannya, lalu fallback ke profile:

```js
model: asString(s.model) || agent.model || undefined,
modelProvider: asString(s.provider) || agent.provider || undefined,
```

4. Pastikan roster yang dipakai `sessions.list` sudah terbaru setelah penggantian model. Periksa apakah perlu memanggil `loadAgentRoster()` sebelum membentuk hasil session, atau refresh roster setelah `sessions.patch` berhasil.

Patch sebelumnya gagal diterapkan hanya karena dua baris `modelProvider: "hermes"` ambigu. Terapkan dengan konteks lebih lengkap, bukan replace global buta.

### B. Verifikasi provider selection saat save

Periksa jalur frontend berikut:

- `src/features/agents/operations/useGatewayConfigSyncController.ts`
- `src/features/office/screens/OfficeScreen.tsx`
- `src/features/agents/operations/useAgentSettingsMutationController.ts`
- `src/lib/gateway/models.ts`

Pastikan model dropdown mempertahankan pasangan provider + model. Dua provider dapat memiliki model ID yang sama. Jangan hanya menyimpan `model.id` bila provider dipilih secara terpisah.

Native shape penting:

- `profiles.describe().model`: `{ provider, default }`
- profile model REST: body `{ provider, model }`
- RPC configure model memerlukan string provider dan model.

### C. Jalankan tests

Mulai dari targeted:

```bash
npm run test -- --run --maxWorkers=1 --no-file-parallelism \
  tests/unit/hermesAgentBridge.test.ts --testTimeout=15000
```

Lalu related suite:

```bash
npm run test -- --run --maxWorkers=1 --no-file-parallelism \
  tests/unit/hermesAgentBridge.test.ts \
  tests/unit/hermesAgentSettingsPersistence.test.ts \
  tests/unit/hermesAgentFilesPersistence.test.ts \
  tests/unit/gatewayAgentSkillsAllowlist.test.ts \
  tests/unit/gatewayAgentOverrides.test.ts \
  tests/unit/agentPermissionsOperation.test.ts \
  tests/unit/agentPermissionsRoleHelpers.test.ts \
  tests/unit/gatewayAgentFiles.test.ts \
  tests/unit/skillsGatewayClient.test.ts \
  tests/unit/gatewayConfigPatch.test.ts \
  tests/unit/deleteAgentOperation.test.ts \
  tests/unit/gatewayProxy.test.ts \
  --testTimeout=15000
```

VM hanya sekitar 3.3 GB RAM. Tetap gunakan `--maxWorkers=1 --no-file-parallelism`.

Lanjutkan dengan:

```bash
npx eslint server/hermes-agent/bridge.js \
  server/hermes-agent/api.js \
  server/hermes-agent/agent-files.js \
  server/hermes-agent/profiles.js \
  server/hermes-agent/settings.js \
  tests/unit/hermesAgentBridge.test.ts \
  tests/unit/hermesAgentFilesPersistence.test.ts \
  tests/unit/hermesAgentSettingsPersistence.test.ts

git diff --check
```

### D. Verifikasi live

Setelah tests lulus:

```bash
systemctl restart hermes3d.service
```

Jangan menganggap status `active` berarti Next.js siap. Tunggu sampai:

```text
http://127.0.0.1:3000/api/health
```

mengembalikan HTTP 200.

Kemudian verifikasi dari dashboard:

1. buka profil `luna`;
2. pastikan dropdown menampilkan semua model available dari provider yang login;
3. pilih model yang bukan `hermes`;
4. simpan;
5. reload halaman/reconnect;
6. pastikan provider dan model tetap sama;
7. jalankan chat pendek untuk memastikan runtime memakai model tersebut.

Gunakan profil sementara untuk test write otomatis. Jangan mengubah atau menghapus profil pengguna untuk pengujian.

## Batasan keselamatan

- Jangan modifikasi upstream `/root/.hermes/hermes-agent`; perbaikan harus berada di adapter `/opt/hermes3d`.
- Jangan mencetak credential, token, atau environment service.
- Jangan menormalisasi identifier untuk delete.
- Jangan menghapus profil `default`.
- Semua mutasi harus diverifikasi dengan read-back.
- Gunakan backend profile path dari roster Hermes; jangan resolve path tersebut pada filesystem host Studio.
- Jangan melaporkan sukses bila backend gagal atau hasil write tidak dapat diverifikasi.

## File utama

- `server/hermes-agent/bridge.js`
- `server/hermes-agent/api.js`
- `server/hermes-agent/agent-files.js`
- `server/hermes-agent/profiles.js`
- `server/hermes-agent/settings.js`
- `tests/unit/hermesAgentBridge.test.ts`
- `tests/unit/hermesAgentFilesPersistence.test.ts`
- `tests/unit/hermesAgentSettingsPersistence.test.ts`
- `src/lib/gateway/models.ts`
- `src/features/agents/operations/useGatewayConfigSyncController.ts`
- `src/features/agents/operations/useAgentSettingsMutationController.ts`
- `src/features/office/screens/OfficeScreen.tsx`

## Ringkasan singkat untuk agent berikutnya

Create, delete, profile files, SOUL, nama, skills, tools, approvals, dan settings persistence sudah diimplementasikan serta pernah lolos related tests/live verification. Bug dropdown satu model sudah ditemukan dan sebagian diperbaiki: adapter sebelumnya membaca field katalog yang salah dan menciptakan fallback palsu `hermes`. Selesaikan propagation provider/model pada `sessions.list`, pastikan frontend menyimpan pasangan provider + model, jalankan seluruh related tests, restart dengan health readiness, lalu verifikasi pilihan model profil `luna` bertahan setelah reload.
