# Traffic Monitoring System

## System Modelling

Fase ida tuir mai ne'e halo tuir saida mak requirement engineering :

### 1. **Use Case Diagram** - Traffic Monitoring System

Diagrama ne'e hatudu interasaun entre user (Polisia Tranzitu) ho sistema, no saida mak sira bele halo iha sistema.

- **Aktor Primáriu**: Polisia Tranzitu
- **Aktor Sekundáriu**: Sistema Otomátiku

**Lista Use Case:**

| ID | Use Case | Aktor | Deskrisaun |
|---------|------|------------|------------|
| UC-01 | Haree video trafiku | Polisia Tranzitu | Polisia bele haree video trafiku ne'ebe prosesu ona husi model |
| UC-02 | Haree ranking lokasi macet | Polisia Tranzitu | Polisia bele haree lokasi ida ne'ebe mak macet liu |
| UC-03 | Simu notifikasaun | Polisia Tranzitu | Polisia simu notifikasaun otomátiku wainhira akontese kemacetan |
| UC-04 | Haree rekord trafiku | Polisia Tranzitu | Polisia bele haree históriku dadus trafiku bazeia ba fatin no oras |
| UC-05 | Detekta vehicle | Sistema Otomátiku | Sistema detekta no sura vehicle otomátiku husi video |
| UC-06 | Haruka notifikasaun otomátiku | Sistema Otomátiku | Sistema haruka notifikasaun wainhira vehicle liu limitasaun |

**Relasaun Use Case:**
- UC-01 **«include»** UC-05 : haree video presiza deteksaun vehicle
- UC-03 **«include»** UC-06 : notifikasaun ba polisia presiza sistema haruka otomátiku

---

### 2. **System Architecture** - Traffic Monitoring System

Sistema ne'e iha layer tolu ne'ebe halo servisu hamutuk:

**Presentation Layer** (Interface ne'ebe polisia haree):
- Dashboard UI — haree video no ranking lokasi macet
- Notifikasaun UI — haree alerta kemacetan
- Rekord UI — haree históriku trafiku

**Processing Layer** (Parte ne'ebe sistema prosesu dadus):
- YOLOv8 Model — detekta no sura vehicle, akurasi ≥ 70%
- Traffic Classifier — klasifika kondisaun trafiku (smooth/crowded/congested)
- Alert Engine — detekta kemacetan no haruka notifikasaun

**Data Layer** (Parte ne'ebe sistema rai dadus):
- Video Input — CCTV ka rekording
- Traffic Records DB — rai rekord trafiku kada 30 segundo
- Notification Log — rai históriku notifikasaun

**Fluxu:**
```
[Presentation Layer]
        ↕
[Processing Layer]
        ↕
[Data Layer]
```

---

### 3. **Data Flow Diagram (DFD)** - Traffic Monitoring System

DFD hatudu oinsá dadus halao husi fonte ida ba seluk iha sistema.

**External Entity:**
- CCTV Kamera — fonte video trafiku
- Polisia Tranzitu — user ne'ebe simu output sistema

**Prosesu:**

| ID | Prosesu | Deskrisaun |
|---------|------|------------|
| P1 | Prosesu video | Simu video stream husi CCTV no prepara frame |
| P2 | Detekta no sura vehicle | YOLOv8 detekta vehicle iha kada frame |
| P3 | Klasifika kondisaun | Klasifika trafiku bazeia ba total vehicle |
| P4 | Jerál notifikasaun | Haruka alerta ba polisia wainhira macet |
| P5 | Hatudu dashboard | Hatudu rezultadu ba polisia liu husi UI |

**Data Store:**

| ID | Data Store | Deskrisaun |
|---------|------|------------|
| DS1 | Video | Rai video trafiku ne'ebe prosesu ona |
| DS2 | Rekord Trafiku | Rai dadus kondisaun trafiku kada lokasaun no oras |
| DS3 | Notifikasaun Log | Rai históriku notifikasaun ne'ebe haruka ona |

**Fluxu Dadus:**
- CCTV → P1 : video stream
- P1 → P2 : frame
- P2 → DS1 : rai video
- P2 → P3 : total vehicle
- P3 → DS2 : status trafiku
- P3 → P4 : alerta kemacetan
- P4 → DS3 : log notifikasaun
- P4 → Polisia : notifikasaun kemacetan
- P3 → P5 : dadus kondisaun
- P5 → Polisia : dashboard view

---

### 4. **Activity Diagram** - Traffic Monitoring System

Diagrama ne'e hatudu fluxu atividade sistema no polisia husi hahu to'o remata.

**Swimlane Sistema:**
- Simu video stream husi CCTV
- Prosesu frame ho YOLOv8
- Sura total vehicle
- Klasifika kondisaun trafiku
- Rai rekord trafiku ba database
- Atualiza dashboard kada 30 segundo
- Repete prosesu hotu otomátiku

**Swimlane Polisia Tranzitu:**
- Simu notifikasaun wainhira vehicle liu 20
- Haree dashboard trafiku
- Halo desizaun: bá kontrola fatin ka lae
- Bá diretamente fatin macet (se desizaun ya)

**Fluxu Desizaun:**

| Kondisaun | Rezultadu |
|---------|------|
| Vehicle > 20 | Sistema haruka notifikasaun → Polisia simu alerta |
| Vehicle ≤ 20 | Sistema rai rekord deit, la haruka notifikasaun |
| Polisia desidi bá | Polisia bá diretamente fatin macet |
| Polisia desidi la bá | Prosesu remata |

**Non-Functional ne'ebe aplika:**
- Sistema repete atividade hotu kada **30 segundo** (NFR-04)
- Sistema la'o **24 oras** kontinuu (NFR-02)
- Akurasi deteksaun **≥ 70%** (NFR-03)

---

### 5. **Rezumu Diagrama**

| Diagrama | Objetivu |
|---------|------|
| Use Case Diagram | Hatudu interasaun entre user no sistema |
| System Architecture | Hatudu estrutura layer husi sistema |
| Data Flow Diagram | Hatudu fluxu dadus iha sistema |
| Activity Diagram | Hatudu fluxu atividade sistema no user |
