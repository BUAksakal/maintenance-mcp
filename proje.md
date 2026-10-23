# Proje: Endüstriyel Bakım Asistanı

> **Çalışma adı:** `maintenance-mcp` (istediğin zaman değiştirebilirsin)
> **Tek cümle:** Herhangi bir AI agent'ını (Claude, Hermes, kendi agent'ın) MCP üzerinden bağlanarak, sanal bir fabrikadaki makine arızalarını gerçek sensör verisiyle tespit eden, bakım kılavuzundan kaynak göstererek teşhis eden ve iş emrini insan onayına bağlayan bir bakım asistanına dönüştüren sistem.

---

## İçindekiler

1. [Problem ve motivasyon](#1-problem-ve-motivasyon)
2. [Sistem ne yapacak? (Demo senaryosu)](#2-sistem-ne-yapacak-demo-senaryosu)
3. [Hedefler ve kapsam dışı](#3-hedefler-ve-kapsam-dışı)
4. [Mimari](#4-mimari)
5. [Veri setleri](#5-veri-setleri)
6. [Teknoloji yığını](#6-teknoloji-yığını)
7. [Repo yapısı](#7-repo-yapısı)
8. [Yol haritası (fazlar)](#8-yol-haritası-fazlar)
9. [Çalışma şekli (Git akışı)](#9-çalışma-şekli-git-akışı)
10. [CI/CD planı](#10-cicd-planı)
11. [Değerlendirme (evaluation) planı](#11-değerlendirme-evaluation-planı)
12. [Güvenlik ve etik](#12-güvenlik-ve-etik)
13. [Riskler ve çözümler](#13-riskler-ve-çözümler)
14. [Yardım alacağımız kaynaklar](#14-yardım-alacağımız-kaynaklar)
15. [Başarı kriterleri ve kariyer çıktıları](#15-başarı-kriterleri-ve-kariyer-çıktıları)
16. [Sözlük](#16-sözlük)
17. [İlk hafta görev listesi](#17-ilk-hafta-görev-listesi)

---

## 1. Problem ve motivasyon

**Problem:** Fabrikalarda motorlar, pompalar ve tezgahlar 7/24 çalışır. Bir makine arızalandığında üretim durur. Teknisyen bir sorunla karşılaştığında:

- sensör grafiklerine bakar,
- kalın bakım kılavuzlarını karıştırır,
- "Bu arızayı daha önce yaşamış mıydık?" diye eski kayıtları arar.

Bu süreç yavaştır ve bilgi dağınıktır.

**Çözüm:** Sensör verisini okuyabilen, kılavuzda arama yapabilen ve iş emri taslağı hazırlayabilen bir **MCP sunucusu**. Herhangi bir AI agent'ı bu sunucuya bağlandığında bakım uzmanı gibi çalışır. Son karar her zaman insandadır.

**Neden bu proje?**

- Sensör/aktüatör geçmişini AI agent'larla birleştiriyor; kalabalıktan ayırıyor.
- Bavyera'daki üretim şirketlerinin (otomotiv, otomasyon) gerçek bir problemine dokunuyor.
- Agent'lar, MCP, RAG, monitoring, evaluation, Docker, CI/CD ve bulut becerilerini tek, çalışan bir sistemde zorunlu kılıyor.

---

## 2. Sistem ne yapacak? (Demo senaryosu)

1. Sanal fabrikada 3 makine çalışıyor; hepsi normal.
2. Terminalden arıza enjekte ediliyor:
   ```bash
   factory inject --machine motor-1 --fault inner_race
   ```
3. Kullanıcı Claude'a soruyor: *"Fabrikada sorunlu bir makine var mı?"*
4. Claude, MCP sunucusu üzerinden:
   - makineleri listeliyor ve sensörleri okuyor,
   - Motor 1'in titreşiminin normal seviyenin çok üstünde olduğunu tespit ediyor,
   - bakım kılavuzunda ilgili arıza modunu buluyor ve **bölüm numarasıyla** kaynak gösteriyor,
   - bir iş emri taslağı hazırlıyor ve **onay istiyor**.
5. Kullanıcı onaylıyor; iş emri kaydediliyor.

> **Şeffaflık notu:** Makineler, gerçek ve açık veri setlerinin yeniden oynatılmasıyla (replay) çalışır. README'de bu açıkça belirtilir.

---

## 3. Hedefler ve kapsam dışı

### Öğrenme hedefleri
- AI agent'larını derinlemesine anlamak (tool tasarımı, RAG, orkestrasyon)
- MCP sunucusu yazmak ve yayınlamak
- Agent'ları izlemek (monitoring/observability) ve ölçmek (evaluation)
- Yazılım mühendisliği pratikleri: Git akışı, test, CI/CD, paketleme, Docker, bulut

### Kariyer hedefleri
- GitHub'da gösterilebilir, uçtan uca çalışan bir proje
- CV'ye ölçülmüş sonuçlarla yazılabilecek bir satır
- Her sürümde bir LinkedIn paylaşımı
- Olası Masterarbeit konusu için temel

### Kapsam dışı (en azından v1'de)
- Gerçek bir fabrikaya bağlantı (v6'da opsiyonel)
- Agent'ın kendi başına aksiyon alması (her yazma işlemi insan onaylı)
- Makine öğrenmesi modeli eğitmek (anomali tespiti başta kural tabanlı)
- Ticari ürün olarak satmak

---

## 4. Mimari

```
   Claude Desktop    Hermes    Kendi LangGraph agent'ın (v3)
          \             |             /
           └──────── MCP (stdio) ────┘
                        │
        ┌───────── MCP Sunucusu ─────────┐
        │ Tools:                          │
        │  list_machines, read_sensors,   │
        │  get_history, detect_anomaly,   │
        │  search_manual,                 │
        │  draft_work_order (onaylı)      │
        │ Resources: kılavuzlar, FMEA     │
        └───────────────┬─────────────────┘
                        │
     ┌──────────────────┼───────────────────┐
     │                  │                   │
 Sanal Fabrika     Bilgi Katmanı        İş Emirleri
 (replay +         (LlamaIndex:         (dosya → ileride
  arıza            kılavuz + FMEA)       veritabanı)
  enjeksiyonu)
     │
 DataSource arayüzü
     ├─ ReplaySource (v1: AI4I, CWRU)
     └─ OPC UA / MQTT Source (v6, opsiyonel)
```

### Bileşenler

| Bileşen | Görevi | Sürüm |
|---|---|---|
| **Veri katmanı** | Veri setlerini indirme, okuma, özellik çıkarma (RMS, kurtosis...) | v1 |
| **Sanal fabrika** | Kayıtları 1 Hz ile oynatma, arıza enjeksiyonu, CLI | v1 |
| **DataSource arayüzü** | Veri kaynağını değiştirilebilir yapma (replay → gerçek makine) | v1 |
| **MCP sunucusu** | Agent'ların kullanacağı tool'lar ve kaynaklar | v1 |
| **Bilgi katmanı** | LlamaIndex ile kılavuz ve FMEA'da kaynaklı arama | v1 |
| **Evaluation** | Etiketli senaryolarla otomatik doğruluk ölçümü | v2 |
| **Orkestrasyon** | LangGraph ile teşhis akışı ve insan onayı | v3 |
| **Monitoring** | Token, maliyet, gecikme izleme ve panel | v3 |
| **Paketleme ve deploy** | Docker, docker compose, AWS, PyPI | v4 |
| **Benchmark** | IBM AssetOpsBench ile ölçüm ve katkı | v5 |

### En önemli tasarım ilkesi
**Veri kaynağı değiştirilebilir olmalı.** Tool'lar ve agent, verinin replay'den mi yoksa gerçek bir makineden mi geldiğini bilmemeli. Böylece ileride gerçek bir sisteme geçişte sadece yeni bir `DataSource` yazılır; geri kalan her şey aynı kalır.

---

## 5. Veri setleri

| Veri seti | İçerik | Nasıl alınır | Lisans / Not | Kullanım |
|---|---|---|---|---|
| **AI4I 2020** | 10.000 satır: hava/işlem sıcaklığı, devir, tork, takım aşınması, 5 arıza türü (arıza oranı yaklaşık %3,4) | `pip install ucimlrepo` → `fetch_ucirepo(id=601)` veya UCI sayfasından CSV | CC BY 4.0. Kendisi de endüstriyi yansıtan sentetik veri. | **Tezgah 1** |
| **CWRU Bearing** | Gerçek motor test düzeneğinden rulman titreşimi; sağlam ve farklı arıza türleri (iç bilezik, dış bilezik, bilye); 12 kHz ve 48 kHz | Resmi Bearing Data Center sitesinden `.mat` dosyaları veya `cwru` Python paketi | Kullanım koşullarını sitede kontrol et | **Motor 1** |
| **NASA IMS Bearing** | Rulmanların arızaya kadar haftalarca kaydedilmiş titreşimi (yavaş bozulma) | data.nasa.gov → "IMS Bearings" → `IMS.zip` | ABD kamu çalışması. Büyük dosya; alt küme kullan. | **Pompa 1** (v2) |
| **AssetOpsBench** | Endüstriyel agent senaryoları, arıza modları, sensör verisi | Hugging Face: `ibm-research/AssetOpsBench`, tam ortam GitHub: `IBM/AssetOpsBench` | Apache-2.0 | **Evaluation** (v5) |

### İndirme linkleri ve adımlar

> İndirilen tüm dosyalar `data/raw/` altına konur. Faz 1'de yazacağımız `download` komutu bu adımları otomatik yapacak; ilk seferde elle indirip veriyi tanıman öğretici olur.

#### AI4I 2020 → Tezgah 1

| | |
|---|---|
| **Sayfa** | https://archive.ics.uci.edu/dataset/601/ai4i |
| **Elle indirme** | Sayfadaki **DOWNLOAD** butonu → zip içinde `ai4i2020.csv` (yaklaşık 510 KB) |
| **Python ile** | `pip install ucimlrepo` → `from ucimlrepo import fetch_ucirepo` → `fetch_ucirepo(id=601)` |
| **Atıf (DOI)** | https://doi.org/10.24432/C5HS5C |

#### CWRU Bearing → Motor 1

| | |
|---|---|
| **Ana sayfa** | https://engineering.case.edu/bearingdatacenter |
| **Veri dosyaları sayfası** | https://engineering.case.edu/bearingdatacenter/download-data-file |
| **Sağlam rulman** | https://engineering.case.edu/bearingdatacenter/normal-baseline-data |
| **12k sürücü tarafı arızalar** (başlangıç için yeterli) | https://engineering.case.edu/bearingdatacenter/12k-drive-end-bearing-fault-data |
| **48k sürücü tarafı arızalar** | https://engineering.case.edu/bearingdatacenter/48k-drive-end-bearing-fault-data |
| **Fan tarafı arızalar** | https://engineering.case.edu/bearingdatacenter/12k-fan-end-bearing-fault-data |
| **Elle indirme** | Sayfalardaki tablolarda her hücre bir kayıttır; dosya adına tıklayınca `.mat` dosyası iner |
| **Biçim** | MATLAB `.mat`. Değişkenler: `DE_time` (sürücü tarafı ivmeölçer), `FE_time` (fan tarafı), `BA_time` (taban), `RPM`. Python'da `scipy.io.loadmat` ile açılır. |
| **Python paketi** | https://github.com/Litchiware/cwru (dosyaları otomatik indirir) |
| **Başlangıç seçimi** | 1 sağlam kayıt + iç bilezik, dış bilezik ve bilye arızalarından birer kayıt (aynı yük koşulunda). Kesin seçimi Faz 1'de birlikte yapacağız. |

#### NASA IMS Bearings → Pompa 1 (v2)

| | |
|---|---|
| **Sayfa** | https://data.nasa.gov/dataset/ims-bearings |
| **Doğrudan indirme** | https://data.nasa.gov/docs/legacy/IMS.zip |
| **İçerik** | Rulmanların arızaya kadar çalıştırıldığı 3 deney; yaklaşık 10 dakikada bir alınmış kısa titreşim kayıtları, düz metin dosyaları |
| **Not** | Büyük bir arşiv. Sadece bir deneyin bir kısmını kullanacağız. |

#### AssetOpsBench → Evaluation (v5)

| | |
|---|---|
| **Veri (senaryolar)** | https://huggingface.co/datasets/ibm-research/AssetOpsBench |
| **Kod ve tam ortam (Docker)** | https://github.com/IBM/AssetOpsBench |
| **Python ile** | `pip install datasets` → `load_dataset("ibm-research/AssetOpsBench", ...)` (alt küme adları dataset sayfasında yazıyor) |

#### Yedek ve ek kaynaklar

| Kaynak | Ne işe yarar? |
|---|---|
| **Paderborn Üniversitesi Bearing DataCenter:** https://mb.uni-paderborn.de/kat/forschung/datacenter/bearing-datacenter/ | Almanya'dan; titreşim ve motor akımı birlikte ölçülmüş. İleride "akım sensörü" eklemek istersek. |
| **Rulman veri setleri listesi:** https://github.com/VictorBauler/awesome-bearing-dataset | Tüm açık rulman veri setlerinin karşılaştırmalı listesi |
| **CWRU zip kopyası (Figshare):** https://figshare.com/articles/dataset/CWRU_zip/28606778 | Üçüncü taraf yedek (yaklaşık 10 MB). Resmi kaynak değil; sadece resmi site erişilemezse. |

### Veri kuralları
- Veri **git'e girmez** (`data/` klasörü `.gitignore`'da). Bunun yerine tek komutla indiren bir script olur.
- Testler için `tests/fixtures/` altında **çok küçük** örnek veri tutulur; CI internete bağımlı olmaz.
- Ham titreşim (binlerce örnek/saniye) agent'a verilmez; her saniye için **özet özellikler** hesaplanır.

### Hesaplanacak özellikler (Faz 1)
| Özellik | Ne anlatır? |
|---|---|
| **RMS** | Titreşimin genel şiddeti |
| **Peak** | En yüksek anlık değer |
| **Kurtosis** | Sivri darbeler var mı? (rulman hasarının tipik işareti) |
| **Crest factor** | Peak / RMS; darbeli titreşim göstergesi |

---

## 6. Teknoloji yığını

| Kategori | Araç | Neden? |
|---|---|---|
| **Dil** | Python 3.12 | Bazı kütüphaneler en yeni Python sürümlerini geç destekleyebilir; 3.12 güvenli seçim |
| **Paket/ortam yönetimi** | `uv` | Hızlı, modern; venv, bağımlılık ve Python sürümünü tek araçta yönetir |
| **Proje tanımı** | `pyproject.toml` | Python projelerinin standart yapılandırma dosyası |
| **Versiyon kontrol** | Git + GitHub | Kod geçmişi, branch, PR, Issues, Actions |
| **Kod stili** | `ruff` | Lint + format; tek araçta hızlı |
| **Test** | `pytest` | Python'da standart test aracı |
| **Commit öncesi kontrol** | `pre-commit` | Hatalı kodun commit edilmesini engeller |
| **Tip kontrolü** | `mypy` (opsiyonel) | Tip hatalarını erken yakalar |
| **CI/CD** | GitHub Actions | Her push'ta test; sürümde paket ve imaj üretimi |
| **Veri** | `pandas`, `numpy`, `scipy` | Tablo verisi, sayısal hesaplama, `.mat` okuma |
| **CLI** | `typer` | Sanal fabrika komutları (`run`, `inject`, `status`) |
| **MCP** | MCP Python SDK (FastMCP) | MCP sunucusu yazmanın resmi yolu |
| **MCP test** | MCP Inspector | Tool'ları LLM olmadan elle denemek |
| **RAG** | LlamaIndex | Kılavuz ve FMEA üzerinde kaynaklı arama |
| **Orkestrasyon** | LangGraph (v3) | Kontrollü teşhis akışı, insan onayı |
| **LLM** | Gemini (ücretsiz katman), Ollama (lokal), Claude Desktop (MCP istemcisi olarak) | Kota ve maliyet esnekliği |
| **Monitoring** | OpenTelemetry + Langfuse veya Arize Phoenix (v3'te seçilecek) | LLM çağrılarını, token ve maliyeti izlemek |
| **Panel** | Streamlit veya Gradio | Fabrika panosu ve canlı demo |
| **Container** | Docker, docker compose (v4) | Tek komutla çalışan sistem |
| **Registry** | GitHub Container Registry (GHCR) | Docker imajlarını saklamak |
| **Paket yayını** | PyPI (Trusted Publishing ile) | `pip install` ile kurulabilir MCP sunucusu |
| **Bulut** | AWS (başlangıçta tek bir EC2 + docker compose) | Canlı demo |

---

## 7. Repo yapısı

```
maintenance-mcp/
├── .github/
│   └── workflows/
│       └── ci.yml               # lint + test (Faz 0)
├── src/
│   └── maintenance_mcp/
│       ├── __init__.py
│       ├── data/                # Faz 1: indirme, okuma, özellik çıkarma
│       │   ├── download.py
│       │   ├── ai4i.py
│       │   ├── cwru.py
│       │   └── features.py
│       ├── factory/             # Faz 2: sanal fabrika
│       │   ├── sources.py       # DataSource arayüzü + ReplaySource
│       │   ├── machine.py
│       │   ├── factory.py
│       │   └── cli.py
│       ├── server/              # Faz 3: MCP sunucusu
│       │   └── server.py
│       ├── knowledge/           # Faz 4: LlamaIndex
│       │   └── index.py
│       └── agent/               # v3: LangGraph
├── docs/
│   ├── manual/                  # Faz 4: bakım kılavuzları (markdown)
│   ├── fmea.csv                 # arıza modu → belirti → sensör → aksiyon
│   └── architecture.md
├── tests/
│   ├── fixtures/                # küçük örnek veri
│   ├── test_features.py
│   ├── test_factory.py
│   └── test_server.py
├── eval/                        # v2: senaryolar ve ölçüm scriptleri
├── data/                        # indirilen veri (git'e girmez)
├── .gitignore
├── .pre-commit-config.yaml
├── pyproject.toml
├── CHANGELOG.md
├── DEVLOG.md                    # öğrenme günlüğü
└── README.md
```

---

## 8. Yol haritası (fazlar)

> **Varsayım:** Haftada 8–10 saat. Süreler tahmindir.
> **Not:** Faz 0, 1 ve 2 için LlamaIndex/LangGraph bilgisi gerekmez; kursa paralel ilerlenebilir.

### Faz 0: Mühendislik temeli (yaklaşık 1 hafta)

**Öğrenilecekler:** Git (commit, branch, merge, PR), Python paketleme (`pyproject.toml`, `src/` yapısı), `uv`, lint (`ruff`), test (`pytest`), `pre-commit`, GitHub Actions ile CI.

- [ ] GitHub'da repo aç; README'ye tek cümlelik hedefi yaz
- [ ] `uv` ile projeyi başlat, `src/` yapısını kur
- [ ] `.gitignore` (`data/`, `.venv/`, `.env`, `__pycache__/`)
- [ ] `ruff` ve `pre-commit` ayarla
- [ ] Basit bir fonksiyon ve onun `pytest` testi
- [ ] `.github/workflows/ci.yml`: her push ve PR'da `ruff` + `pytest`
- [ ] README'ye CI rozeti
- [ ] Issue → branch → PR → merge akışını bir kez uçtan uca dene
- [ ] `DEVLOG.md`'ye ilk kayıt

**Bitti sayılır:** PR açıldığında CI otomatik çalışıyor ve yeşil.

### Faz 1: Veri (yaklaşık 1 hafta)

**Öğrenilecekler:** pandas ile keşifsel veri analizi (EDA), sinyal işleme temelleri, tekrarlanabilir veri hattı, test fixture'ları.

- [ ] AI4I'yi indir; notebook'ta incele (sütunlar, dağılımlar, arıza türleri)
- [ ] CWRU'dan sağlam + 2–3 arızalı kayıt indir; bir kaydı çiz ve incele
- [ ] `features.py`: pencere bazlı RMS, peak, kurtosis, crest factor
- [ ] Testler: bilinen sinyallerle doğrulama (ör. sinüsün RMS'i = genlik / √2)
- [ ] `download` komutu: veriyi tek komutla `data/` altına indirir
- [ ] `tests/fixtures/` altına küçük örnekler

**Bitti sayılır:** Tek komutla veri iniyor, özellikler hesaplanıyor, testler CI'da geçiyor.

### Faz 2: Sanal fabrika (yaklaşık 1 hafta)

**Öğrenilecekler:** Nesne yönelimli tasarım, arayüz (abstract class), bağımlılığı tersine çevirme (dependency inversion), zaman serisi akışı, CLI tasarımı.

- [ ] `DataSource` arayüzü (abstract class)
- [ ] `ReplaySource`: bir kaydı 1 Hz ile oynatır
- [ ] `Machine`: durum (normal/arızalı); arıza enjeksiyonu = oynatılan kaydı değiştirmek
- [ ] `Factory`: Motor 1 (CWRU) ve Tezgah 1 (AI4I)
- [ ] CLI: `factory run`, `factory status`, `factory inject`, `factory reset`
- [ ] Testler: oynatma sırası, arıza geçişi, sıfırlama

**Bitti sayılır:** Fabrika çalışıyor; arıza enjekte edildiğinde Motor 1'in özellikleri değişiyor.

### Faz 3: MCP sunucusu (yaklaşık 1 hafta)

**Öğrenilecekler:** MCP protokolü (tools, resources, prompts, stdio), JSON şeması, iyi tool tasarımı, LLM'siz test.

- [ ] MCP kavramlarını öğren; basit bir "hello" tool'u yaz
- [ ] Tool'lar: `list_machines`, `read_sensors`, `get_history`, `detect_anomaly` (önce kural tabanlı: normal seviyeye göre eşik)
- [ ] Her tool'un açıklamasını özenle yaz (LLM sadece bunu görür)
- [ ] MCP Inspector ile tool'ları elle dene
- [ ] Testler: tool'ları LLM olmadan çağır
- [ ] Claude Desktop'a bağla; "Fabrikada sorun var mı?" diye sor

**Bitti sayılır:** Claude, arıza enjekte edilen makineyi doğru buluyor.

### Faz 4: Bilgi katmanı ve teşhis (yaklaşık 1 hafta)

**Öğrenilecekler:** RAG, chunking, kaynak gösterme, insan onayı (human-in-the-loop).

- [ ] `docs/manual/`: her makine için kısa bakım kılavuzu (markdown, numaralı bölümler)
- [ ] `docs/fmea.csv`: arıza modu → belirti → ilgili sensör → önerilen aksiyon
- [ ] LlamaIndex ile index; `search_manual` tool'u (bölüm numarasıyla döner)
- [ ] `draft_work_order`: taslağı `pending_approval` durumunda kaydeder; onay ayrı bir adım
- [ ] Uçtan uca demo

**Bitti sayılır:** Demo senaryosu (bölüm 2) baştan sona çalışıyor.

### Yayın: v1.0.0

- [ ] README: problem, demo videosu/GIF, mimari, kurulum, sınırlamalar
- [ ] `CHANGELOG.md`
- [ ] `v1.0.0` etiketi ve GitHub Release
- [ ] LinkedIn paylaşımı

### v1 sonrası

| Sürüm | İçerik | Öğrenilecekler |
|---|---|---|
| **v2: Evaluation** | Etiketli senaryolar; doğruluk, yanlış alarm, kaynak doğruluğu; NASA IMS ile yavaş bozulma | Agent değerlendirme, metrik tasarımı |
| **v3: Orkestrasyon + Monitoring** | `diagnose` tool'u içinde LangGraph akışı; token, maliyet, gecikme izleme ve panel | LangGraph, OpenTelemetry, observability |
| **v4: Docker + CD** | Dockerfile, docker compose; GitHub Actions ile imaj → GHCR; sürümde PyPI'ye otomatik yayın; AWS'de canlı demo | Container, CD, bulut deploy |
| **v5: AssetOpsBench** | Benchmark'ta ölçüm; "LLM tek başına vs LLM + bu sunucu"; senaryo katkısı (PR) | Benchmark, açık kaynak katkı |
| **v6 (opsiyonel)** | OPC UA veya MQTT `DataSource` | Endüstriyel protokoller |

---

## 9. Çalışma şekli (Git akışı)

Tek başına çalışsan bile şirketlerdeki akışı kullan:

1. **Issue aç:** Her görev bir GitHub Issue. (İstersen GitHub Projects ile Kanban panosu.)
2. **Branch aç:** `feat/replay-source`, `fix/rms-window`, `docs/readme` gibi.
3. **Küçük commit'ler:** Conventional Commits biçiminde:
   - `feat:` yeni özellik
   - `fix:` hata düzeltme
   - `test:` test ekleme
   - `docs:` dokümantasyon
   - `chore:` bakım işleri
4. **Pull Request aç:** Açıklamaya ne yaptığını ve neden yaptığını yaz.
5. **CI yeşil olmadan merge yok.**
6. **DEVLOG.md:** Her adımın sonunda 2–3 cümle: ne öğrendim, nerede takıldım, nasıl çözdüm.

---

## 10. CI/CD planı

| Aşama | Tetikleyici | Ne çalışır? | Sürüm |
|---|---|---|---|
| **CI: kalite** | Her push ve PR | `ruff check`, `ruff format --check`, `pytest` | v1 |
| **CI: tip kontrolü** | Her push ve PR | `mypy` (opsiyonel) | v1 |
| **Evaluation** | Elle veya haftada bir | Senaryo ölçümleri (LLM maliyeti yüzünden her push'ta değil) | v2 |
| **CD: Docker imajı** | `main` branch'e merge | İmajı build edip GHCR'a gönder | v4 |
| **CD: PyPI** | `v*` etiketi | Paketi build edip PyPI'ye yayınla (Trusted Publishing) | v4 |
| **CD: Deploy** | Yeni sürüm | AWS'deki demoyu güncelle | v4 |

**Gizli bilgiler:** API anahtarları asla repoya girmez. Lokal'de `.env` (gitignore'da), GitHub'da **Secrets**.

---

## 11. Değerlendirme (evaluation) planı

Arızayı biz enjekte ettiğimiz ve veri setleri etiketli olduğu için **doğru cevap her zaman bilinir.**

### Senaryo biçimi (örnek)
```json
{
  "id": "S001",
  "inject": {"machine": "motor-1", "fault": "inner_race"},
  "question": "Fabrikada sorunlu bir makine var mı?",
  "expected": {"machine": "motor-1", "fault_type": "inner_race", "manual_section": "4.2"}
}
```

### Metrikler
| Metrik | Ne ölçer? |
|---|---|
| **Tespit doğruluğu** | Doğru makineyi buldu mu? |
| **Teşhis doğruluğu** | Arıza türü doğru mu? |
| **Kaynak doğruluğu** | Doğru kılavuz bölümünü gösterdi mi? |
| **Yanlış alarm oranı** | Arıza yokken "arıza var" dedi mi? |
| **Gecikme** | Cevap ne kadar sürdü? |
| **Token ve maliyet** | Teşhis başına kaç token, kaç cent? |

### Karşılaştırmalar
- LLM tek başına vs LLM + bu MCP sunucusu
- Büyük bulut modeli vs küçük lokal model
- Kural tabanlı anomali tespiti vs istatistiksel yöntem

---

## 12. Güvenlik ve etik

- **Varsayılan olarak sadece okuma.** Yazma işlemleri (iş emri) açık onay gerektirir.
- **Agent hiçbir zaman kendi başına aksiyon almaz.**
- **Şeffaflık:** Verinin açık veri setlerinden yeniden oynatıldığı README'de açıkça yazılır.
- **Lisanslar:** Her veri setinin lisansına uyulur ve kaynak gösterilir.
- **Gizli bilgiler:** API anahtarları `.env` ve GitHub Secrets'ta; repoda asla.
- **Canlı demo:** Kullanım limiti ve maliyet tavanı.

---

## 13. Riskler ve çözümler

| Risk | Çözüm |
|---|---|
| **API kotası** (Gemini ücretsiz katmanında günlük istek limiti düşük) | Geliştirmede sık sık LLM çağırma; tool'ları LLM'siz test et; gerekirse lokal model (Ollama) |
| **8 GB RAM** (lokal modeller zayıf kalabilir) | Lokal modeli sadece basit testlerde kullan; asıl demolarda bulut model |
| **Kapsamın büyümesi** | v1 kapsamına sadık kal; yeni fikirleri Issue olarak not edip sonraki sürüme bırak |
| **Zaman / motivasyon** | Küçük adımlar; her hafta çalışan bir şey; DEVLOG ile ilerlemeyi gör |
| **Python sürüm uyumsuzluğu** | Projeyi `uv` ile Python 3.12'ye sabitle |
| **Büyük veri setleri** (NASA IMS) | Sadece alt küme kullan; tamamını indirme |
| **CWRU'nun yüksek örnekleme hızı** | Ham veri yerine pencere bazlı özellikler |

---

## 14. Yardım alacağımız kaynaklar

### Kurs
- **Hugging Face Agents Course:** LlamaIndex (2.2), LangGraph (2.3), Unit 3 (Agentic RAG), Unit 4 (GAIA), **Bonus Unit 2 (Observability & Evaluation)**: https://huggingface.co/learn/agents-course

### Resmi dokümantasyonlar
- **MCP:** https://modelcontextprotocol.io
- **uv:** https://docs.astral.sh/uv/
- **ruff:** https://docs.astral.sh/ruff/
- **pytest:** https://docs.pytest.org
- **GitHub Actions:** https://docs.github.com/actions
- **LlamaIndex:** https://docs.llamaindex.ai
- **LangGraph:** https://langchain-ai.github.io/langgraph/
- **Langfuse:** https://langfuse.com
- **Arize Phoenix:** https://phoenix.arize.com

### Veri setleri
- **AI4I 2020:** https://archive.ics.uci.edu/dataset/601/ai4i
- **CWRU Bearing Data Center:** https://engineering.case.edu/bearingdatacenter
- **`cwru` Python paketi:** https://github.com/Litchiware/cwru
- **NASA IMS Bearings:** https://data.nasa.gov/dataset/ims-bearings
- **AssetOpsBench (veri):** https://huggingface.co/datasets/ibm-research/AssetOpsBench
- **AssetOpsBench (kod):** https://github.com/IBM/AssetOpsBench

### Claude (öğretmen olarak)
Her adımda aynı yöntem:
1. **Kavram:** Bu adım neden var, gerçek hayatta nasıl kullanılıyor?
2. **İskelet:** Boşluklu kod ve ipuçları
3. **Sen yazarsın** ve çalıştırırsın
4. **İnceleme:** Kodu ve çıktıyı birlikte okuruz
5. **Not:** Öğrendiğini DEVLOG'a yazarsın

---

## 15. Başarı kriterleri ve kariyer çıktıları

### v1 için başarı kriterleri
- [ ] CI yeşil, testler geçiyor
- [ ] Tek komutla veri iniyor, fabrika çalışıyor
- [ ] Claude, enjekte edilen arızayı doğru makinede buluyor
- [ ] Teşhis, kılavuz bölümüyle kaynak gösteriyor
- [ ] İş emri sadece onayla kaydediliyor
- [ ] README ve demo videosu hazır

### CV satırı (v5 sonrası hedef)
> *Built an open-source MCP server that turns any AI agent into an industrial maintenance assistant using real bearing vibration data (LangGraph, LlamaIndex, Docker, AWS, CI/CD); improved diagnosis accuracy on IBM's AssetOpsBench from X% to Y% with full cost and latency monitoring.*

### LinkedIn paylaşım planı
| Sürüm | Paylaşım konusu |
|---|---|
| v1 | "Claude'u bir bakım uzmanına çeviren MCP sunucusu yazdım" |
| v2 | "Agent'ımın doğruluğunu nasıl ölçüyorum" |
| v3 | "Agent'ımı nasıl izliyorum: token, maliyet, gecikme" |
| v4 | "Projemi Docker ve GitHub Actions ile AWS'ye nasıl taşıdım" |
| v5 | "IBM'in endüstriyel benchmark'ındaki sonuçlarım" |

---

## 16. Sözlük

| Terim | Anlamı |
|---|---|
| **Agent** | Bir LLM'in tool'ları kullanarak adım adım görev çözdüğü sistem |
| **MCP** | Model Context Protocol; tool'ları herhangi bir AI agent'ına sunmak için standart "fiş" |
| **Tool** | LLM'in çağırabildiği fonksiyon; LLM sadece adını ve açıklamasını görür |
| **RAG** | Retrieval-Augmented Generation; önce ilgili bilgiyi bul, sonra LLM onunla cevap versin |
| **Replay** | Kaydedilmiş gerçek veriyi sanki canlıymış gibi tekrar oynatmak |
| **Arıza enjeksiyonu** | Test için bir makineyi bilerek "bozmak" |
| **RMS** | Titreşimin genel şiddeti |
| **Kurtosis** | Sinyalde sivri darbelerin olup olmadığını gösteren ölçü |
| **FMEA** | Arıza modu ve etkileri analizi; hangi arıza hangi belirtiyi verir |
| **İş emri (work order)** | Bakım ekibine verilen resmi görev |
| **Human-in-the-loop** | Kritik adımlarda son kararı insanın vermesi |
| **CI** | Continuous Integration; her değişiklikte testlerin otomatik çalışması |
| **CD** | Continuous Delivery/Deployment; onaylanan kodun otomatik paketlenip yayına alınması |
| **Observability / Monitoring** | Sistemin içinde ne olduğunu izlemek (token, gecikme, hata) |
| **Evaluation** | Sistemin ne kadar doğru çalıştığını etiketli senaryolarla ölçmek |
| **Semantic versioning** | Sürüm numaralandırma: `MAJOR.MINOR.PATCH` (ör. `1.2.0`) |

---

## 17. İlk hafta görev listesi

- [ ] Proje adını kesinleştir
- [ ] GitHub'da repo aç
- [ ] `uv` kur ve projeyi Python 3.12 ile başlat
- [ ] `src/maintenance_mcp/` yapısını oluştur
- [ ] `.gitignore`, `README.md`, `DEVLOG.md` ekle
- [ ] `ruff` ve `pre-commit` ayarla
- [ ] İlk fonksiyonu ve testini yaz
- [ ] GitHub Actions CI'ını kur
- [ ] İlk Issue → branch → PR → merge döngüsünü tamamla
- [ ] DEVLOG'a ilk haftanın notlarını yaz

> **Bir sonraki adım:** Repo açıldığında Faz 0'a başlıyoruz. İlk konu: Git ve proje yapısı.
