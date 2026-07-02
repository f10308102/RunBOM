# Runtime SCA

Languages: [中文](#中文) | [English](#english)

## 中文

Runtime SCA 是一個只觀測、只讀取的 Runtime SBOM 探針。它會在一段觀測時間內捕捉主機上「實際被執行的 ELF 執行檔」與「實際被映射成可執行的共享函式庫」，再把這些檔案解析成作業系統套件或 standalone ELF，最後輸出 CycloneDX 1.7 JSON。

簡單說，它不是看「系統安裝了什麼」，而是看「程式執行時真的用到了什麼」。

Linux 版使用 eBPF，FreeBSD 版使用 DTrace。兩個版本的輸出格式一致，後續都可以交給 Grype、Trivy、Dependency-Track 或其他支援 CycloneDX 的工具處理。

## 可以做什麼

- 產生執行期 Runtime BOM / Runtime SBOM。
- 觀測已存在程序與觀測期間新啟動的程序。
- 捕捉執行檔啟動與共享函式庫載入。
- 將檔案對應回 Debian / Ubuntu 的 `dpkg` 套件。
- 將檔案對應回 RHEL / CentOS / Fedora / SUSE 系的 `rpm` 套件。
- 將 FreeBSD 檔案對應回 `pkg` 套件。
- 無法歸屬到套件時，退回 ELF build-id 與 SHA-256 的 standalone 元件。
- 輸出 CycloneDX 1.7 BOM。
- 選配輸出 evidence JSONL、OS library BOM、process-to-library JSON、linked CycloneDX dependencies graph。
- 針對指定 PID 與其子程序做 targeted observation。

Runtime SCA 本身不做 CVE 比對、不掃描原始碼、不掃描 container image，也不會輸出完整安裝套件清單。Runtime BOM 只代表觀測時間窗內實際看見的執行檔與函式庫。

## Release 檔案

| 檔案 | 用途 |
|------|------|
| `runtime-sca` | Linux x86_64 執行檔，建議使用 musl 靜態版本 |
| `runtime-sca-freebsd` | FreeBSD 執行檔，請依目標架構提供 amd64 或 arm64 對應版本 |
| `config.example.toml` | 範例設定檔，可選 |
| `README.md` | 使用說明 |
| `SHA256SUMS` | 檔案雜湊，建議提供 |

下載後如果沒有執行權限，先執行：

```bash
chmod +x runtime-sca
chmod +x runtime-sca-freebsd
```

## 下載驗證與 VirusTotal

每個 Release 建議提供 `SHA256SUMS`。下載後可先比對雜湊，確認檔案就是 Release 宣告的版本：

```bash
sha256sum runtime-sca
sha256sum runtime-sca-freebsd
```

目前已掃描的 Release binaries 如下。這些結果只適用於各自 SHA-256 對應的檔案；只要重新編譯，雜湊就會改變，需要重新掃描。

| 項目 | 內容 |
|------|------|
| 檔案 | `runtime-sca` |
| VirusTotal 偵測結果 | `0 / 48` security vendors flagged this file as malicious |
| VirusTotal 報告 | <https://www.virustotal.com/gui/file/7747e6e337560629ffd860806178fc745de20bb8a28a7db256a8cdad1533640d> |
| SHA-256 | `7747e6e337560629ffd860806178fc745de20bb8a28a7db256a8cdad1533640d` |
| SHA-1 | `c753c4a8065e8d1025aaad4497c11bb6f010fce1` |
| MD5 | `12733d93eedcf453a8a0563b492ddaea` |
| 檔案大小 | `3.66 MB (3,834,744 bytes)` |
| 檔案類型 | `ELF 64-bit x86-64, static-pie linked, stripped` |
| First Submission | `2026-07-02 12:27:08 UTC` |
| Last Analysis | `2026-07-02 12:27:08 UTC` |

| 項目 | 內容 |
|------|------|
| 檔案 | `runtime-sca-freebsd` |
| VirusTotal 偵測結果 | `0 / 62` security vendors flagged this file as malicious |
| VirusTotal 報告 | <https://www.virustotal.com/gui/file/a1deb89f1a39063497c189e5c73acef2f6ae4f0d7433a77df6a67bd5791b94f7> |
| SHA-256 | `a1deb89f1a39063497c189e5c73acef2f6ae4f0d7433a77df6a67bd5791b94f7` |
| SHA-1 | `3faf37155fdd1f15479d4e254028d8d6f22d2123` |
| MD5 | `da51e0386731ee79c1961d09bbca65e7` |
| 檔案大小 | `3.52 MB (3,685,952 bytes)` |
| 檔案類型 | `ELF 64-bit x86-64, FreeBSD, statically linked, stripped` |
| First Submission | `2026-07-02 12:37:24 UTC` |
| Last Analysis | `2026-07-02 12:37:24 UTC` |

VirusTotal 結果僅供交叉檢查，不代表絕對安全。Runtime SCA 需要 root 權限並使用 eBPF / DTrace 觀測執行期行為，部分安全產品可能因行為特徵產生誤判；請搭配來源、雜湊、Release 簽章與實際環境驗證一起判斷。

## 支援版本

實際是否可用請以 `doctor` 指令為準。Runtime SCA 會檢查核心能力、BTF、tracepoint、套件管理工具與權限。

### Linux

| 系統 | 核心 / 條件 | 套件管理 | 狀態 |
|------|-------------|----------|------|
| Ubuntu 24.04 LTS | kernel 6.8 | dpkg | 已實測通過 |
| Ubuntu 22.04 LTS | kernel 5.15 | dpkg | 符合需求，建議先跑 `doctor` |
| Debian 12 | kernel 6.1 | dpkg | 符合需求，建議先跑 `doctor` |
| RHEL / Rocky / AlmaLinux 9 | kernel 5.14 | rpm | 符合需求，建議先跑 `doctor` |
| CentOS Stream / RHEL 8.4+ | kernel 4.18 with BPF/BTF backport | rpm | 可能支援，以 `doctor` 實測能力為準 |
| Ubuntu 20.04 HWE | kernel 5.15 | dpkg | 可能支援，以 `doctor` 為準 |
| Ubuntu 20.04 GA | kernel 5.4 | dpkg | 通常不支援 |
| CentOS / RHEL 7 | kernel 3.10 | rpm | 不支援 |

Linux 硬性需求：

- x86_64。
- root 權限；實務上請直接用 `sudo` 執行 `doctor` 與 `observe`。
- kernel 5.8+，或發行版已 backport BPF ring buffer 與 BTF。
- `/sys/kernel/btf/vmlinux` 存在。
- `tracefs` 與必要 syscall / sched tracepoints 可用。
- `dpkg-query` 或 `rpm` 可用。

### FreeBSD

| 系統 | 條件 | 套件管理 | 狀態 |
|------|------|----------|------|
| FreeBSD 13.x | DTrace 可用 | pkg | 已實作，請以 `doctor` 驗證 |
| FreeBSD 14.x | DTrace 可用 | pkg | 已實作，請以 `doctor` 驗證 |
| FreeBSD 15.x | DTrace 可用 | pkg | 已實作，請以 `doctor` 驗證 |

FreeBSD 硬性需求：

- amd64 或 arm64，且下載對應架構的 binary。
- root 權限。
- DTrace kernel module 已載入。
- `pkg` 可用。

FreeBSD 可先載入 DTrace：

```sh
kldload dtraceall
```

## 快速開始：Linux

把 `runtime-sca` 放到目標主機後，先檢查環境：

```bash
sudo ./runtime-sca doctor --json
```

如果 JSON 裡的 `supported` 是 `true`，就可以開始觀測：

```bash
sudo ./runtime-sca observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl
```

觀測期間請啟動或操作你想分析的服務。Runtime SCA 只會記錄觀測期間真的被執行或映射的元件。

使用設定檔：

```bash
./runtime-sca validate --config config.example.toml

sudo ./runtime-sca observe \
  --config config.example.toml \
  --duration 60 \
  --output-bom runtime-bom.cdx.json
```

## 快速開始：FreeBSD

FreeBSD 最小安裝環境通常沒有 `sudo`，以下假設你已經是 root。

```sh
chmod +x runtime-sca-freebsd
kldload dtraceall

./runtime-sca-freebsd doctor --json

./runtime-sca-freebsd observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl
```

不要用 `sh runtime-sca-freebsd` 執行；它是二進位執行檔，不是 shell script。

## Command Line

```text
runtime-sca <COMMAND> [OPTIONS]
```

| 指令 | 說明 | 是否需要 root |
|------|------|---------------|
| `doctor` | 檢查主機是否支援，不留下 eBPF / DTrace 掛載 | 是 |
| `validate` | 驗證設定檔與輸出路徑，不載入探針 | 否 |
| `observe` | 開始觀測並輸出 BOM | 是 |
| `version` | 顯示版本、Event ABI、toolchain 與目標架構 | 否 |

完整參數說明：

```bash
./runtime-sca /help
./runtime-sca doctor --help
./runtime-sca observe --help
```

### `doctor`

```bash
sudo ./runtime-sca doctor
sudo ./runtime-sca doctor --json
sudo ./runtime-sca doctor --config config.example.toml --json
```

`doctor` 用來確認目標主機是否真的能跑。遇到不支援時，請先看 `errors` 欄位。

### `validate`

```bash
./runtime-sca validate --config config.example.toml
```

只檢查 TOML 設定與輸出路徑，不載入 eBPF 或 DTrace。

### `observe`

```bash
sudo ./runtime-sca observe [OPTIONS]
```

常用參數：

| 參數 | 預設 | 說明 |
|------|------|------|
| `--config <FILE>` | 內建預設 | 讀取 TOML 設定檔 |
| `--duration <SECS>` | `0` | 觀測秒數；`0` 表示直到 Ctrl+C 或 SIGTERM |
| `--target-pid <PID>` | 全主機 | 只觀測指定 PID 與其子程序，可重複指定 |
| `--output-bom <FILE>` | `runtime-bom.cdx.json` | 主要 CycloneDX Runtime BOM |
| `--evidence-output <FILE>` | 無 | 每個元件一行 JSONL 證據 |
| `--os-lib-bom <FILE>` | 無 | 只包含 OS 套件函式庫的 CycloneDX |
| `--process-lib-report <FILE>` | 無 | 每個 executable 載入哪些 library 的 JSON |
| `--linked-bom <FILE>` | 無 | 完整 inventory 加上 CycloneDX dependencies graph |
| `--no-proc-baseline` | baseline 開啟 | 關閉啟動時對既有程序的 baseline 掃描 |

觀測 60 秒：

```bash
sudo ./runtime-sca observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json
```

一直觀測直到手動停止：

```bash
sudo ./runtime-sca observe \
  --duration 0 \
  --output-bom runtime-bom.cdx.json
```

只觀測某個服務與其子程序：

```bash
PID="$(pgrep -o nginx)"

sudo ./runtime-sca observe \
  --duration 60 \
  --target-pid "$PID" \
  --output-bom nginx-runtime.cdx.json \
  --evidence-output nginx-evidence.jsonl
```

同一次觀測輸出多種視圖：

```bash
sudo ./runtime-sca observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl \
  --os-lib-bom os-libs.cdx.json \
  --process-lib-report process-libs.json \
  --linked-bom linked-runtime-bom.cdx.json
```

### `version`

```bash
./runtime-sca version
./runtime-sca version --json
```

目前專案版本為 `0.1.0`，Event ABI 版本為 `1`。

## 設定檔範例

`config.example.toml`：

```toml
# Linux procfs 根目錄；一般主機維持 /proc 即可。FreeBSD 後端不使用。
proc_root = "/proc"

# Linux sysfs 根目錄；用來檢查 BTF 與 tracefs。FreeBSD 後端不使用。
sys_root = "/sys"

# 預設觀測秒數；0 表示一直跑到 Ctrl+C 或 SIGTERM。
duration_seconds = 60

# 主要輸出檔：CycloneDX 1.7 Runtime BOM。
output_bom = "runtime-bom.cdx.json"

# 選配輸出：每個觀測元件一行 JSONL 證據；省略或設為 null 表示不輸出。
evidence_output = "evidence.jsonl"

# 啟動時是否先掃描目前已存在的程序與 executable mappings。
proc_baseline = true

# 最多記錄多少個被觀測到的檔案，避免長時間觀測造成記憶體無界成長。
max_observed_files = 8192

# 所有觀測路徑字串的總位元組上限。
max_observed_path_bytes = 8388608

# 查詢套件歸屬的逾時秒數；Linux 使用 dpkg-query / rpm，FreeBSD 使用 pkg。
package_query_timeout_seconds = 5

# 空陣列表示觀測整台主機；填 PID 則只觀測那些 PID 與其子程序。
target_pids = []
```

可選輸出也可以寫進設定檔：

```toml
# 選配輸出：只包含 OS 套件提供且 runtime 有載入的 library。
os_lib_bom = "os-libs.cdx.json"

# 選配輸出：JSON 格式的 executable -> loaded libraries 關係圖。
process_lib_report = "process-libs.json"

# 選配輸出：完整 Runtime BOM，並加入 CycloneDX dependencies graph。
linked_bom = "linked-runtime-bom.cdx.json"
```

CLI 參數會覆寫設定檔中的常用觀測欄位，例如 `--duration`、`--output-bom`、`--target-pid`。

## 輸出檔案

### Runtime BOM

主要輸出是 CycloneDX 1.7 JSON：

```bash
runtime-bom.cdx.json
```

每個 component 會盡量包含：

- `name`
- `version`
- `purl`
- `cpe`
- `SHA-256`
- ELF build-id
- 實際出現位置 `evidence.occurrences`
- Runtime SCA 自訂 metadata properties

套件 purl 範例：

```text
pkg:deb/ubuntu/openssl@3.0.13-0ubuntu3.5?arch=amd64
pkg:rpm/rocky/openssl-libs@3.0.7-27.el9?arch=x86_64
pkg:freebsd/nginx@1.24.0_1
```

無法對應到 OS 套件的 ELF 會使用 standalone identity：

```text
runtime-sca:elf:sha256:<sha256>
```

### Evidence JSONL

`--evidence-output evidence.jsonl` 會輸出每個元件的觀測證據，適合稽核或後續資料處理。

### OS Library BOM

`--os-lib-bom os-libs.cdx.json` 只輸出「由 OS 套件提供、且在 runtime 被載入的 library」。這份檔案適合拿去做 OS library 風險檢查。

### Process Library Report

`--process-lib-report process-libs.json` 會輸出每個 executable 載入哪些 library。範例結構：

```json
{
  "schema_version": "1.0",
  "observation": {
    "start": "...",
    "end": "...",
    "scope": "targeted"
  },
  "executables": [
    {
      "path": "/usr/sbin/nginx",
      "name": "nginx",
      "version": "1.24.0-2ubuntu7.12",
      "pids": [1234],
      "libraries": [
        {
          "path": "/usr/lib/x86_64-linux-gnu/libssl.so.3",
          "name": "openssl",
          "version": "3.0.13-0ubuntu3.5"
        }
      ]
    }
  ]
}
```

### Linked BOM

`--linked-bom linked-runtime-bom.cdx.json` 會輸出完整 component 清單，並額外加入 CycloneDX `dependencies`，用來表示 executable package 在 runtime 觀測到的 library dependency。

## 弱點掃描範例

Runtime SCA 本身只產生 SBOM，不做弱點比對。要掃 CVE，可把 `runtime-bom.cdx.json` 交給外部工具。

Ubuntu 24.04：

```bash
grype sbom:runtime-bom.cdx.json --distro ubuntu:24.04
```

Ubuntu 22.04：

```bash
grype sbom:runtime-bom.cdx.json --distro ubuntu:22.04
```

Debian 12：

```bash
grype sbom:runtime-bom.cdx.json --distro debian:12
```

RHEL / Rocky / AlmaLinux 9：

```bash
grype sbom:runtime-bom.cdx.json --distro redhat:9
```

輸出 JSON 報告：

```bash
grype sbom:runtime-bom.cdx.json \
  --distro ubuntu:24.04 \
  -o table \
  -o json=grype-report.json
```

注意：掃 Linux OS 套件時建議明確指定 `--distro`。Runtime BOM 的 package URL 會標出 distro family，但不一定包含完整 OS 版本；不指定版本可能造成掃描器誤判或漏報。

FreeBSD 建議使用原生 VuXML 資料源：

```sh
pkg audit -F
```

## Exit Codes

| Code | 意義 |
|------|------|
| `0` | 成功 |
| `2` | 設定錯誤或輸出路徑錯誤 |
| `10` | 平台、核心或主機能力不支援 |
| `12` | eBPF / DTrace 載入或 attach 失敗 |
| `13` | 輸出檔案寫入失敗 |
| `23` | 內部錯誤 |

## 常見問題

### 為什麼 BOM 裡元件比預期少？

Runtime SCA 只記錄觀測期間真的發生的執行檔與函式庫。如果某條程式路徑沒有被觸發，或 library 沒有被載入，就不會出現在 Runtime BOM。

### 這可以取代完整 SBOM 嗎？

不建議。Runtime BOM 適合回答「實際執行時用到了什麼」。完整 build-time SBOM、image SBOM、source SBOM 仍然有自己的用途。最好的做法通常是兩者並用。

### 這會修改系統嗎？

Runtime SCA 的目標是觀測。Linux 版會載入 eBPF probe，FreeBSD 版會啟動 DTrace；觀測結束後會先卸載 probe，再解析套件與輸出檔案。它不會修改被觀測程式或系統套件。

### `doctor` 失敗怎麼辦？

先看 JSON 裡的 `errors`。常見原因是：

- 沒有 root 權限。
- Linux kernel 太舊。
- 缺少 BTF。
- tracefs 沒掛載或必要 tracepoint 不存在。
- 沒有 `dpkg-query` / `rpm` / `pkg`。
- FreeBSD 沒有載入 DTrace module。

### 可以在 container 裡跑嗎？

通常需要在 host 上跑，因為 eBPF / DTrace 需要主機層級能力。若在 container 裡執行，必須確認容器有足夠權限與主機檔案系統視角；實務上建議直接在目標主機執行。

## English

Runtime SCA is a read-only Runtime SBOM probe. During an observation window, it captures the ELF executables that actually ran and the shared libraries that were actually mapped as executable, resolves those files to OS packages or standalone ELF identities, and emits a CycloneDX 1.7 JSON Runtime BOM.

In short, it does not answer "what is installed on this host?" It answers "what did this running workload actually use?"

The Linux build uses eBPF. The FreeBSD build uses DTrace. Both builds produce the same output format, so the generated BOM can be consumed by tools such as Grype, Trivy, Dependency-Track, or any other CycloneDX-compatible system.

### What It Does

- Generates a runtime Runtime BOM / Runtime SBOM.
- Observes already-running processes and processes started during the observation window.
- Captures executable starts and shared-library loads.
- Resolves files to Debian / Ubuntu packages through `dpkg`.
- Resolves files to RHEL / CentOS / Fedora / SUSE-family packages through `rpm`.
- Resolves FreeBSD files to `pkg` packages.
- Falls back to standalone ELF identity using ELF build-id and SHA-256 when no package owner is found.
- Emits CycloneDX 1.7 BOMs.
- Optionally emits evidence JSONL, OS library BOM, process-to-library JSON, and linked CycloneDX dependencies graph.
- Supports targeted observation for specific PIDs and their descendants.

Runtime SCA does not perform CVE matching, source-code scanning, container-image scanning, or full installed-package inventory. A Runtime BOM only represents executables and libraries observed within the selected time window.

### Release Files

| File | Purpose |
|------|---------|
| `runtime-sca` | Linux x86_64 binary, preferably the static musl build |
| `runtime-sca-freebsd` | FreeBSD binary; publish the matching amd64 or arm64 build |
| `config.example.toml` | Optional example configuration |
| `README.md` | Usage guide |
| `SHA256SUMS` | Recommended release checksums |

If the downloaded files are not executable yet:

```bash
chmod +x runtime-sca
chmod +x runtime-sca-freebsd
```

### Download Verification And VirusTotal

Each Release should provide `SHA256SUMS`. After downloading, compare hashes to confirm that the file matches the release artifact:

```bash
sha256sum runtime-sca
sha256sum runtime-sca-freebsd
```

The currently scanned release binaries are listed below. Each result only applies to the exact file with the listed SHA-256. Rebuilding the binary changes the hash and requires a new scan.

| Field | Value |
|-------|-------|
| File | `runtime-sca` |
| VirusTotal detection | `0 / 48` security vendors flagged this file as malicious |
| VirusTotal report | <https://www.virustotal.com/gui/file/7747e6e337560629ffd860806178fc745de20bb8a28a7db256a8cdad1533640d> |
| SHA-256 | `7747e6e337560629ffd860806178fc745de20bb8a28a7db256a8cdad1533640d` |
| SHA-1 | `c753c4a8065e8d1025aaad4497c11bb6f010fce1` |
| MD5 | `12733d93eedcf453a8a0563b492ddaea` |
| File size | `3.66 MB (3,834,744 bytes)` |
| File type | `ELF 64-bit x86-64, static-pie linked, stripped` |
| First Submission | `2026-07-02 12:27:08 UTC` |
| Last Analysis | `2026-07-02 12:27:08 UTC` |

| Field | Value |
|-------|-------|
| File | `runtime-sca-freebsd` |
| VirusTotal detection | `0 / 62` security vendors flagged this file as malicious |
| VirusTotal report | <https://www.virustotal.com/gui/file/a1deb89f1a39063497c189e5c73acef2f6ae4f0d7433a77df6a67bd5791b94f7> |
| SHA-256 | `a1deb89f1a39063497c189e5c73acef2f6ae4f0d7433a77df6a67bd5791b94f7` |
| SHA-1 | `3faf37155fdd1f15479d4e254028d8d6f22d2123` |
| MD5 | `da51e0386731ee79c1961d09bbca65e7` |
| File size | `3.52 MB (3,685,952 bytes)` |
| File type | `ELF 64-bit x86-64, FreeBSD, statically linked, stripped` |
| First Submission | `2026-07-02 12:37:24 UTC` |
| Last Analysis | `2026-07-02 12:37:24 UTC` |

VirusTotal results are only a cross-check, not a security guarantee. Runtime SCA needs root privileges and uses eBPF / DTrace to observe runtime behavior, so some security products may flag similar behavior heuristically. Evaluate the release source, checksum, release signature, VirusTotal result, and your own environment together.

### Supported Versions

Always use `doctor` as the final source of truth. Runtime SCA checks kernel capability, BTF, tracepoints, package-manager availability, and privileges on the target host.

#### Linux

| System | Kernel / condition | Package manager | Status |
|--------|--------------------|-----------------|--------|
| Ubuntu 24.04 LTS | kernel 6.8 | dpkg | Verified |
| Ubuntu 22.04 LTS | kernel 5.15 | dpkg | Meets requirements; run `doctor` first |
| Debian 12 | kernel 6.1 | dpkg | Meets requirements; run `doctor` first |
| RHEL / Rocky / AlmaLinux 9 | kernel 5.14 | rpm | Meets requirements; run `doctor` first |
| CentOS Stream / RHEL 8.4+ | kernel 4.18 with BPF/BTF backport | rpm | May be supported; rely on `doctor` |
| Ubuntu 20.04 HWE | kernel 5.15 | dpkg | May be supported; rely on `doctor` |
| Ubuntu 20.04 GA | kernel 5.4 | dpkg | Usually unsupported |
| CentOS / RHEL 7 | kernel 3.10 | rpm | Unsupported |

Linux requirements:

- x86_64.
- Root privileges; run `doctor` and `observe` with `sudo`.
- Linux kernel 5.8+, or a distribution kernel with backported BPF ring buffer and BTF support.
- `/sys/kernel/btf/vmlinux` exists.
- `tracefs` and the required syscall / sched tracepoints are available.
- `dpkg-query` or `rpm` is available.

#### FreeBSD

| System | Condition | Package manager | Status |
|--------|-----------|-----------------|--------|
| FreeBSD 13.x | DTrace available | pkg | Implemented; validate with `doctor` |
| FreeBSD 14.x | DTrace available | pkg | Implemented; validate with `doctor` |
| FreeBSD 15.x | DTrace available | pkg | Implemented; validate with `doctor` |

FreeBSD requirements:

- amd64 or arm64, using the matching binary.
- Root privileges.
- DTrace kernel modules loaded.
- `pkg` is available.

Load DTrace first when needed:

```sh
kldload dtraceall
```

### Quick Start: Linux

After placing `runtime-sca` on the target host, check host capability first:

```bash
sudo ./runtime-sca doctor --json
```

If the JSON output contains `"supported": true`, start observation:

```bash
sudo ./runtime-sca observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl
```

Start or exercise the workload during the observation window. Runtime SCA only records files that were actually executed or mapped during that time.

Using a configuration file:

```bash
./runtime-sca validate --config config.example.toml

sudo ./runtime-sca observe \
  --config config.example.toml \
  --duration 60 \
  --output-bom runtime-bom.cdx.json
```

### Quick Start: FreeBSD

Minimal FreeBSD installations often do not include `sudo`; the example assumes you are already root.

```sh
chmod +x runtime-sca-freebsd
kldload dtraceall

./runtime-sca-freebsd doctor --json

./runtime-sca-freebsd observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl
```

Do not run `sh runtime-sca-freebsd`; it is a binary executable, not a shell script.

### Command Line

```text
runtime-sca <COMMAND> [OPTIONS]
```

| Command | Description | Needs root |
|---------|-------------|------------|
| `doctor` | Check whether the host is supported; leaves no eBPF / DTrace attachment behind | Yes |
| `validate` | Validate configuration and output paths without loading probes | No |
| `observe` | Observe runtime activity and emit BOMs | Yes |
| `version` | Print version, Event ABI, toolchain, and target architecture | No |

Full help:

```bash
./runtime-sca /help
./runtime-sca doctor --help
./runtime-sca observe --help
```

#### `doctor`

```bash
sudo ./runtime-sca doctor
sudo ./runtime-sca doctor --json
sudo ./runtime-sca doctor --config config.example.toml --json
```

Use `doctor` to confirm whether the target host can run Runtime SCA. If it reports unsupported, inspect the `errors` field.

#### `validate`

```bash
./runtime-sca validate --config config.example.toml
```

Validates TOML and output paths only. It does not load eBPF or DTrace.

#### `observe`

```bash
sudo ./runtime-sca observe [OPTIONS]
```

Common options:

| Option | Default | Description |
|--------|---------|-------------|
| `--config <FILE>` | built-in defaults | Read a TOML config file |
| `--duration <SECS>` | `0` | Observation seconds; `0` means until Ctrl+C or SIGTERM |
| `--target-pid <PID>` | host-wide | Observe only this PID and descendants; repeatable |
| `--output-bom <FILE>` | `runtime-bom.cdx.json` | Main CycloneDX Runtime BOM |
| `--evidence-output <FILE>` | none | JSONL evidence, one component per line |
| `--os-lib-bom <FILE>` | none | CycloneDX containing OS-package-owned runtime libraries only |
| `--process-lib-report <FILE>` | none | JSON executable-to-loaded-libraries report |
| `--linked-bom <FILE>` | none | Full inventory plus CycloneDX dependencies graph |
| `--no-proc-baseline` | baseline enabled | Disable startup baseline scan of existing processes |

Observe for 60 seconds:

```bash
sudo ./runtime-sca observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json
```

Observe until manually stopped:

```bash
sudo ./runtime-sca observe \
  --duration 0 \
  --output-bom runtime-bom.cdx.json
```

Observe one service and its child processes:

```bash
PID="$(pgrep -o nginx)"

sudo ./runtime-sca observe \
  --duration 60 \
  --target-pid "$PID" \
  --output-bom nginx-runtime.cdx.json \
  --evidence-output nginx-evidence.jsonl
```

Emit multiple output views from one observation:

```bash
sudo ./runtime-sca observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl \
  --os-lib-bom os-libs.cdx.json \
  --process-lib-report process-libs.json \
  --linked-bom linked-runtime-bom.cdx.json
```

#### `version`

```bash
./runtime-sca version
./runtime-sca version --json
```

Current project version: `0.1.0`. Event ABI version: `1`.

### Configuration Example

`config.example.toml`:

```toml
# Linux procfs root. Keep "/proc" on a normal Linux host. Ignored by FreeBSD.
proc_root = "/proc"

# Linux sysfs root, used for BTF and tracefs checks. Ignored by FreeBSD.
sys_root = "/sys"

# Observation duration in seconds. 0 means run until Ctrl+C or SIGTERM.
duration_seconds = 60

# Main output: CycloneDX 1.7 Runtime BOM.
output_bom = "runtime-bom.cdx.json"

# Optional output: one JSONL evidence record per observed component.
evidence_output = "evidence.jsonl"

# Capture already-running processes and executable mappings at startup.
proc_baseline = true

# Maximum number of observed files kept in memory.
max_observed_files = 8192

# Total byte budget for observed path strings.
max_observed_path_bytes = 8388608

# Timeout in seconds for native package ownership queries.
# Linux uses dpkg-query / rpm; FreeBSD uses pkg.
package_query_timeout_seconds = 5

# Empty means host-wide observation. Add PIDs to observe only those PIDs and descendants.
target_pids = []
```

Optional outputs can also be configured:

```toml
# Optional output: OS-package-owned runtime libraries only.
os_lib_bom = "os-libs.cdx.json"

# Optional output: executable -> loaded libraries JSON report.
process_lib_report = "process-libs.json"

# Optional output: full Runtime BOM plus CycloneDX dependencies graph.
linked_bom = "linked-runtime-bom.cdx.json"
```

CLI flags override common observation settings from the config file, such as `--duration`, `--output-bom`, and `--target-pid`.

### Output Files

#### Runtime BOM

The main output is CycloneDX 1.7 JSON:

```bash
runtime-bom.cdx.json
```

Each component includes as much identity and evidence as possible:

- `name`
- `version`
- `purl`
- `cpe`
- `SHA-256`
- ELF build-id
- Actual observed paths in `evidence.occurrences`
- Runtime SCA metadata properties

Package URL examples:

```text
pkg:deb/ubuntu/openssl@3.0.13-0ubuntu3.5?arch=amd64
pkg:rpm/rocky/openssl-libs@3.0.7-27.el9?arch=x86_64
pkg:freebsd/nginx@1.24.0_1
```

Standalone ELF fallback identity:

```text
runtime-sca:elf:sha256:<sha256>
```

#### Evidence JSONL

`--evidence-output evidence.jsonl` writes one JSONL evidence record per observed component. This is useful for audit trails and downstream processing.

#### OS Library BOM

`--os-lib-bom os-libs.cdx.json` emits only runtime-loaded libraries owned by OS packages.

#### Process Library Report

`--process-lib-report process-libs.json` emits the libraries loaded by each executable. Example shape:

```json
{
  "schema_version": "1.0",
  "observation": {
    "start": "...",
    "end": "...",
    "scope": "targeted"
  },
  "executables": [
    {
      "path": "/usr/sbin/nginx",
      "name": "nginx",
      "version": "1.24.0-2ubuntu7.12",
      "pids": [1234],
      "libraries": [
        {
          "path": "/usr/lib/x86_64-linux-gnu/libssl.so.3",
          "name": "openssl",
          "version": "3.0.13-0ubuntu3.5"
        }
      ]
    }
  ]
}
```

#### Linked BOM

`--linked-bom linked-runtime-bom.cdx.json` emits the full component inventory and adds CycloneDX `dependencies` to represent the executable packages and library packages observed at runtime.

### Vulnerability Scanning Example

Runtime SCA only produces SBOMs; it does not perform vulnerability matching. Feed `runtime-bom.cdx.json` to an external scanner.

Ubuntu 24.04:

```bash
grype sbom:runtime-bom.cdx.json --distro ubuntu:24.04
```

Ubuntu 22.04:

```bash
grype sbom:runtime-bom.cdx.json --distro ubuntu:22.04
```

Debian 12:

```bash
grype sbom:runtime-bom.cdx.json --distro debian:12
```

RHEL / Rocky / AlmaLinux 9:

```bash
grype sbom:runtime-bom.cdx.json --distro redhat:9
```

Write a JSON report:

```bash
grype sbom:runtime-bom.cdx.json \
  --distro ubuntu:24.04 \
  -o table \
  -o json=grype-report.json
```

When scanning Linux OS packages, explicitly pass `--distro`. Runtime BOM package URLs include the distro family, but not always the full OS version; omitting it can cause false negatives or incorrect matching.

For FreeBSD, use the native VuXML data source:

```sh
pkg audit -F
```

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `2` | Configuration error or output path problem |
| `10` | Unsupported platform, kernel, or host capability |
| `12` | eBPF / DTrace load or attach failure |
| `13` | Output write failure |
| `23` | Internal error |

### FAQ

#### Why does the BOM contain fewer components than expected?

Runtime SCA only records executables and libraries that actually appeared during the observation window. If a code path was not triggered, or a library was not loaded, it will not appear in the Runtime BOM.

#### Can this replace a full SBOM?

Not by itself. Runtime BOMs answer "what was actually used at runtime?" Build-time SBOMs, image SBOMs, and source SBOMs are still useful. In practice, they complement each other.

#### Does Runtime SCA modify the system?

Runtime SCA is designed for observation. The Linux build loads eBPF probes; the FreeBSD build runs DTrace. After observation stops, probes are detached before package and ELF resolution happens. It does not modify observed programs or system packages.

#### What should I do if `doctor` fails?

Check the `errors` field in the JSON output. Common causes include:

- Missing root privileges.
- Linux kernel too old.
- Missing BTF.
- tracefs not mounted or required tracepoints unavailable.
- Missing `dpkg-query` / `rpm` / `pkg`.
- FreeBSD DTrace modules not loaded.

#### Can this run inside a container?

Usually it should run on the host, because eBPF / DTrace require host-level capabilities. Running inside a container is possible only when the container has enough privileges and the correct host filesystem view; running directly on the target host is recommended.

## License

`MIT OR Apache-2.0`
