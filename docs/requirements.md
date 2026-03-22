# Traffic Monitoring System

## Requirement Engineering

Fase ida tuir mai ne'e halo tuir saida mak pro hanonin ona:

### 1. **Feasibility Study** - Traffic monitoring system 
    - note: ida ne'e optional kuando ita hakarak lee

study primeiro hodi determina katak sistema ne'e merese duni atu kria.

- **Technical Feasibility**: video ne'ebe fo sai trafiku iha tiha ona, no dataset mos fahe tiha ona ba training no testing, no tools ne'ebe uza tuir nesesidade mak **python** no nia library/package sira no detection model vehicle ne'ebe iha no gratuitamente/open source.

- **Operational Feasibility**: polisia tranzitu sira presiza maneira ne'ebe efisiente liu hodi monitora engarrafamento(kemacetan). Sistema ida ne'e ajuda polisia sira hodi haree lokasi ne'ebe mak trafiku(macet) liu laho tenke ba kedas fatin hodi hein trafiku(macet) ne'e akontese. Polisia sei simu deit notifikasi fatin ida ne'ebe mak trafiku(macet) liu, atu nune'e polisia bele ba diretamente fatin refere hodi controla trafiku(macet) la presiza buka idak idak.

- **Time Feasibility**: Dados sira preparado tiha ona hodi nune'e ekonomiza tempo. Model sei halo integra deit ona ba sistema no Interface komesa kria deit ona. 

- **Feasibility Report**: 
    | Aspetu | Status | esplikasaun |
    |---------|------|------------|
    | Technical  | viavel  | tools no dados preparado |
    | Operational  | Viavel  | usefull ba polisia tranzitu sira |
    | Time  | Viavel  | dados preparad, hein deit developmet |

### 2. Requirement Elicitation and Analysis Elicitation

- **Elicitation**(prosesu no maneira atu recolha ka investiga nesesidade(requirement) husi user ka stakeholder(ema ne'ebe utiliza sistema ida ne'e))

Rekolha requirement system ho maneira halo diskusaun iha team, ho nia rezultado mak hanesan tuir mai. User ba sistema mak Polisia Tranzitu sira Presiza:
- Bele haree video trafiku iha lokasi oioin.
- Bele hatene fatin/lokasi ida ne'ebe mak macet liu.
- Bele hetan notifikasi se kuandu iha engarrafamento(kemacetan) ne'ebe grave atu nune'e bele koloka polisia tranzitu sira lalais hodi atende engarrafamento(kemacetan) ne'e refere.
- Bele haree dados rekordu engarrafamento(kemacetan)

- **Analiza**(prosesu analiza, ordena no klarifika requirements ne'ebe rekolha ona durante elicitation)

Husi requirements ne'ebe rekolha ona, sei agrupa sai rua ne'e mak:

- **Functional Requirements**:
    - Sistema sei fo sai video trafiku ne'ebe prosesu ona husi model.
    - Sistema sei fo sai grafiku ida hodi fo sai fatin ida ne'ebe mak trafiku(macet) liu.
    - Sistema sei fo hatene notifikasaun wainhira akontese trafiku(kemacet).
    - Sistema Rai dados rekordu trafiku nian.

- **Non-Functional Requirements**:
    - UI/UX tenke fasil hodi komprende husi polisia 
    - Sistema la'o 24 horas
    - model detekta vehicle mos tenke akurat

### 3. User and System Requirements (ne'e importante ba System Development)

- **User Requirements**
requirement husi parte user(polisia tranzitu) hakerek iha lingua ne'ebe fasil atu komprende:
- Polisia sira bele haree video trafiku husi lokasi/fatin ne'ebe prosesu ona ka pasan ona kamera ba.
- polisia bele hatene lokasi ida ne'ebe mak trafiku(macet) liu.
- polisi hetan notifikasi automatikamente wainhira engarrafamento(kemacetan) ne'ebe grave.
- Polisia bele haree dados rekordu Trafiku nian bazeia ba fatin no oras.

- **System Requirements**
Nesesidade husi parte sistema nian hakerek ho forma tekniku liu:
- Functional:
    - Sistema halo prosesu video Trafiku utiliza model machine learning ne'ebe train ona.
    - Sistema detekta no sura total vehicle husi kada frame video.
    - Sistema halo klasifikasaun kondisaun trafiku bazeia ba total vehicle ne'ebe detekta ona.
    - Sistem fo sai lista lokasi/fatin halo ordena husi ida ne'ebe mak trafiku(macet) liu.
    - sistema hatama notifiksasaun automatikamente wainhira total vehicle liu limitasaun.
    - sistema rai data rekordu engarrafamento(kemacetan) kada lokasaun no oras.

- Non-functional
    - Sistema Fasil hodi utiliza husi Polisia 
    - Sistema la'o 24 jam ho full
    - Model detekta vehicle iha akurasi ne'ebe minimal 80%
    - Dados sira tama iha systema ne'e tenke kada 30 segundo

  | Status | Total vehicle | esplikasaun |
    |---------|------|------------|
    | 🟢 smooth  | < 10 vehicle  | Normal |
    | 🟡 crowded  | 10 to'o 20 vehicle  | Presiza monitora|
    | 🔴 congested  | > 20 vehicle  | Notifikasaun haruka |


### 4. Requirements Documents

- **Systema Rezumu**
Traffic monitoring system mak sistema ida ne'ebe halo prosesu video trafiku utiliza model YOLOv8 hodi detekta no sura total vehicle. Sistema ida ne'e sei utiliza husi polisia tranzitu hodi monitora kondisaun engarrafamento(kemacetan) iha fatin ne'ebe mensiona ona iha leten laho tenke ba kampu refere.

- **Autor**
User ida deit ne mak Polisia tranzitu.

- **Lista Functional Requirements**

  | ID | Requirements |
    |---------|------|
    | FR-01  | Sistema fo sai video trafic ne'ebe prosesu ona husi model  |
    | FR-02   | sistema fo sai fatin/lokasi ida ne'ebe mak trafiku(macet) liu, liu husi grafiku  |
    | FR-03 | sistema haruka notifikasi automatikamente wainhira engarrafamentu(kemacetan) akontese |
    | FR-04 | sistema rai dados rekordu trafiku nian  |

- **Lista Non-Functional Requirement**

    | ID | Requirements |
    |---------|------|
    | NFR-01  | UI/UX fasil hodi komprende husi polisia tranzitu sira  |
    | NFR-02   | Sistema la'o 24 oras |
    | NFR-03 | model YOLOv8 nia akurasi minimal 70% |


