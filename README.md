# SOC-LAB-WAZUH-DASHBOARD

Projekt przedstawia autorski dashboard stworzony w środowisku Wazuh/Kibana, mający na celu scentralizowane monitorowanie bezpieczeństwa stacji roboczej z systemem Windows 10. Celem prac było wyeliminowanie szumu informacyjnego i stworzenie przejrzystego panelu operacyjnego dla analityka bezpieczeństwa.

## Hardware & Host OS 

* **System operacyjny hosta:** Windows 10 Pro (64-bit)
* **Procesor (CPU):** Intel(R) Core(TM) i7-2600 CPU @ 3.40GHz
* **Pamięć RAM:** 32,0 GB
* **Dysk:** SSD Patriot P210 512GB

## Software Stack
* **Hiperwzorzec:** Oracle VirtualBox (wersja 7.2.10) – posłużył do stworzenia izolowanej sieci, w której maszyny komunikują się w obrębie podsieci `192.168.0.0/24`, zapewniając stabilne środowisko laboratoryjne.
* **SIEM / XDR:** Wazuh Manager (Virtual Appliance oparty na Ubuntu) – `192.168.0.115` (Centralny punkt zbierania i analizy logów).
* **Endpoint (Ofiara):** Windows 10 Pro – `192.168.0.110` (Zainstalowany agent Wazuh oraz sensor Microsoft Sysmon z konfiguracją SwiftOnSecurity).
* **System Atakującego:** Kali Linux – adresacja w tej samej podsieci `192.168.0.109` (Platforma do generowania ruchu i symulacji ataków).

## 1. Główne Funkcjonalności Dashboardu
Zaprojektowany przeze mnie dashboard zawiera moduły kluczowe dla szybkiej oceny stanu bezpieczeństwa hosta:

### 1. Panel Procedur Reagowania (Playbook)
Wykorzystałem moduł `Markdown`, aby umieścić na pulpicie podręczną instrukcję postępowania (tzw. SOC Playbook). Dzięki temu, w przypadku wykrycia alertu o poziomie krytycznym (Level 12+), analityk ma przed oczami konkretne kroki, co skraca czas reakcji (MTTR - Mean Time To Respond).

### 2. Monitoring Stanu i Zagrożeń
**Alert Level Gauge:** Wskaźnik typu `Gauge`, który wizualizuje skalę zagrożenia na podstawie zliczonych alertów. Pozwala na błyskawiczne określenie, czy system znajduje się w stanie "bezpiecznym", czy wymaga interwencji.
**Metric Counter:** Licznik prezentujący zagregowaną liczbę alertów w czasie rzeczywistym, służący jako główne "powiadomienie" o incydentach.

### 3. Analiza Trendów
**Line Chart (Activity):** Wykres liniowy prezentujący natężenie zdarzeń w czasie. Pozwala zidentyfikować anomalie, takie jak nagłe skoki aktywności, które mogą wskazywać na skanowanie portów lub próby przejęcia uprawnień.
**Pie Chart (Severity Distribution):** Wizualizacja rozkładu ważności zdarzeń, która pomaga priorytetyzować pracę analityka.

### 4. Szczegółowy Rejestr Zdarzeń
Dolna sekcja dashboardu to konfigurowalna tabela, która filtruje logi według:
* Nazwy reguły (`rule.description`).
* Poziomu ważności (`rule.level`).
* Nazwy agenta (`agent.name`).

## 2. Wdrożenie techniczne

Aby Dashboard działał poprawnie na wybranym agencie, zastosowałem globalny filtr w środowisku Kibana:
`agent.ip: "192.168.0.110"`
Pozwoliło to na odizolowanie logów ofiary od logów systemowych serwera, co drastycznie podniosło czytelność danych. Wizualizacje zostały oparte na domyślnych indeksach `wazuh-alerts-*` z wykorzystaniem agregacji `Terms` oraz `Date Histogram` dla wykresów czasowych.

## 3. Analiza Operacyjna: Podgląd Systemu (24h)

<img width="1867" height="785" alt="1" src="https://github.com/user-attachments/assets/0744307a-2713-4d66-8d81-07486b992a2f" />

Powyższy zrzut ekranu przedstawia realny widok dashboardu w środowisku produkcyjnym/laboratoryjnym, ilustrujący aktywność na hoście `DESKTOP-UR3OJVV` w cyklu dobowym. Dashboard został skonfigurowany w sposób pozwalający na natychmiastową kategoryzację zdarzeń.

### Szczegółowy podział komponentów Dashboardu:

#### 1. Moduł Procedur Reagowania (SOC Playbook)
Umieszczony w lewym górnym rogu panel Markdown pełni rolę "ściągi" dla analityka. Skrócenie procesu decyzyjnego poprzez wypunktowanie priorytetowych działań (izolacja hosta, analiza PID, eskalacja do administratora) jest kluczowe w sytuacjach podbramkowych (MTTR).

#### 2. Moduł Wizualizacji Stanu (Metrics & Gauge)
* **Alert Level Gauge:** Wskazuje obecnie 483 zdarzenia, co przy przyjętej skali (0-1000) pozycjonuje system w strefie średniego/wysokiego zagrożenia (żółto-pomarańczowy segment).
* **Metric Counter:** Liczba "26" odnosi się do ostatniego interwału czasowego, pozwalając na szybką detekcję nagłego wzrostu aktywności w porównaniu do średniej.

#### 3. Wykres Trendów (Line Chart - Activity)
Wykres liniowy wizualizuje natężenie zdarzeń w rozbiciu na 30-minutowe interwały. Widoczny "płaski" odcinek przechodzący w wyraźne skoki aktywności pozwala na korelację działań atakującego w czasie – od fazy rekonesansu, przez modyfikacje systemowe, aż po aktywne działania post-exploitation.

#### 4. Szczegółowy Rejestr (Alerts-Windows Table)
Tabela pozwala na zaawansowany "Threat Hunting". Analizując przykładowe dane, możemy zidentyfikować następujące techniki ataku (zgodnie z metodologią MITRE ATT&CK):

* **Modyfikacje Rejestru (Level 5):** Liczne zdarzenia typu `Registry Value Integrity Checksum Changed` (75) oraz `Registry Key Integrity Checksum Changed` (59) sugerują próby utrzymania trwałości (Persistence) lub modyfikacje konfiguracji systemowej przez szkodliwe oprogramowanie.
* **Wykonywanie skryptów (Level 9 & 4):** Wykrycie `C:\Windows\SysWOW64\WindowsPowerShell 1.0\powershell.exe` tworzącego plik skryptu wskazuje na aktywność typu "Fileless" lub użycie PowerShell do pobrania dodatkowych modułów ataku (Downloader/Stager).
* **Detekcja Malware (Level 15):** Zdarzenie `Executable file dropped in folder commonly used by malware` z najwyższym poziomem ważności (15) jest bezpośrednim sygnałem krytycznym, wymagającym natychmiastowej aktywacji procedury izolacji hosta, zgodnie z instrukcją w panelu Markdown.
* **Monitoring Plików:** `Executable dropped in Windows root folder` (73) świadczy o próbie eskalacji uprawnień lub instalacji binariów w chronionych lokalizacjach systemowych.

## 4. Przyszły rozwój
* **Integracja z powiadomieniami:** Konfiguracja alertów wysyłanych przez e-mail/Slack dla zdarzeń o poziomie >= 12.
* **Threat Hunting:** Wdrożenie dodatkowych reguł wykrywających techniki z macierzy MITRE ATT&CK.
* **Automatyzacja:** Skryptowanie izolacji hosta przez API Wazuh po wykryciu specyficznego ataku.
