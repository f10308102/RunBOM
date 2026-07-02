# RunBOM Demo

這份文件是一個可以直接照著跑的 demo 腳本，用來展示 RunBOM 如何產生 Runtime BOM、如何看輸出，以及如何把 BOM 丟給 Grype 做弱點掃描。

Demo 重點：

- RunBOM 觀測的是「實際執行過」與「實際載入過」的 ELF 檔案。
- 它輸出的是 CycloneDX 1.7 Runtime BOM。
- 它本身不做 CVE 掃描，弱點比對交給 Grype、Trivy 或 Dependency-Track。
- Linux 使用 eBPF，FreeBSD 使用 DTrace。

## Demo 準備

你需要一台支援的 Linux 或 FreeBSD 主機，以及對應的 release binary。

Linux 檔案：

```bash
runtime-sca
config.example.toml
```

FreeBSD 檔案：

```sh
runtime-sca-freebsd
config.example.toml
```

先加上執行權限：

```bash
chmod +x runtime-sca
chmod +x runtime-sca-freebsd
```

Linux demo 建議使用 Ubuntu 24.04 LTS。其它 Linux 發行版請先用 `doctor` 確認。

## Demo 1: Linux 快速觀測

### 1. 檢查主機能力

```bash
sudo ./runtime-sca doctor --json
```

重點看：

```json
{
  "supported": true,
  "architecture": "x86_64",
  "btf_available": true,
  "privileged": true,
  "package_manager": "dpkg"
}
```

如果 `supported` 是 `false`，先看 `errors` 欄位。常見原因是沒有 root 權限、核心太舊、缺少 BTF、缺少 tracefs，或找不到 `dpkg-query` / `rpm`。

### 2. Terminal A: 開始觀測

```bash
sudo ./runtime-sca observe \
  --duration 30 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl
```

這個指令會觀測 30 秒。觀測期間不要關掉它。

### 3. Terminal B: 觸發一些 workload

在另一個 terminal 執行：

```bash
for i in $(seq 1 5); do
  /bin/ls /usr/bin >/dev/null
  /usr/bin/env >/dev/null
  command -v openssl >/dev/null 2>&1 && openssl version >/dev/null 2>&1
  command -v curl >/dev/null 2>&1 && curl --version >/dev/null 2>&1
  command -v ssh >/dev/null 2>&1 && ssh -V >/dev/null 2>&1
  sleep 1
done
```

這些命令只是 demo 用，用來製造一些實際執行的 executable 和 runtime library mapping。

### 4. 確認輸出檔案

觀測結束後應該看到：

```bash
ls -lh runtime-bom.cdx.json evidence.jsonl
```

如果有 `jq`，可以快速看 BOM 摘要：

```bash
jq '.bomFormat, .specVersion, (.components | length)' runtime-bom.cdx.json
```

預期會看到：

```text
"CycloneDX"
"1.7"
<component count>
```

看前幾個 component：

```bash
jq '.components[0:5] | map({type, name, version, purl, cpe})' runtime-bom.cdx.json
```

看 RunBOM 寫入的觀測 metadata：

```bash
jq '.metadata.properties[] | select(.name | startswith("runtime-sca:"))' runtime-bom.cdx.json
```

## Demo 2: 針對特定 PID 觀測

這個 demo 用來展示 `--target-pid`。你可以換成 nginx、java、python、node 或任何你想分析的服務。

### 1. 找到目標 PID

以 nginx 為例：

```bash
PID="$(pgrep -o nginx)"
echo "$PID"
```

如果沒有 nginx，可以用你自己的程式 PID。若只是想測流程，也可以先啟動一個背景程序：

```bash
sleep 120 &
PID="$!"
echo "$PID"
```

### 2. 只觀測該 PID 與其子程序

```bash
sudo ./runtime-sca observe \
  --duration 30 \
  --target-pid "$PID" \
  --output-bom targeted-runtime-bom.cdx.json \
  --evidence-output targeted-evidence.jsonl
```

如果目標服務在觀測期間有 fork 或啟動子程序，RunBOM 會一起追蹤那些子程序。

## Demo 3: 一次輸出多種視圖

這個 demo 展示 RunBOM 的四種常見輸出。

```bash
sudo ./runtime-sca observe \
  --duration 60 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl \
  --os-lib-bom os-libs.cdx.json \
  --process-lib-report process-libs.json \
  --linked-bom linked-runtime-bom.cdx.json
```

輸出說明：

| 檔案 | 說明 |
|------|------|
| `runtime-bom.cdx.json` | 完整 Runtime BOM，CycloneDX 1.7 |
| `evidence.jsonl` | 每個元件一行 JSONL 觀測證據 |
| `os-libs.cdx.json` | 只包含 OS 套件提供且 runtime 有載入的 library |
| `process-libs.json` | executable 到 loaded libraries 的 JSON 關係圖 |
| `linked-runtime-bom.cdx.json` | 完整 BOM 加上 CycloneDX dependencies graph |

查看某個 executable 載入哪些 library：

```bash
jq '.executables[0]' process-libs.json
```

查看 linked BOM 的 dependencies：

```bash
jq '.dependencies[0:10]' linked-runtime-bom.cdx.json
```

## Demo 4: 用 Grype 掃描 Runtime BOM

RunBOM 不做弱點比對。要掃 CVE，可以把 `runtime-bom.cdx.json` 交給 Grype。

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

同時輸出表格和 JSON：

```bash
grype sbom:runtime-bom.cdx.json \
  --distro ubuntu:24.04 \
  -o table \
  -o json=grype-report.json
```

注意：Linux OS 套件掃描建議明確加上 `--distro`。Runtime BOM 的 purl 會標示 distro family，但不一定包含完整 OS 版本；不指定版本可能造成掃描工具漏報或誤判。

## Demo 5: FreeBSD 快速觀測

FreeBSD 版使用 DTrace。以下假設你已經是 root。

### 1. 載入 DTrace

```sh
kldload dtraceall
```

### 2. 檢查主機能力

```sh
./runtime-sca-freebsd doctor --json
```

重點看：

```json
{
  "supported": true,
  "os_id": "freebsd",
  "dtrace_available": true,
  "package_manager": "pkg"
}
```

### 3. 開始觀測

```sh
./runtime-sca-freebsd observe \
  --duration 30 \
  --output-bom runtime-bom.cdx.json \
  --evidence-output evidence.jsonl
```

另一個 terminal 觸發一些命令：

```sh
for i in 1 2 3 4 5; do
  /bin/ls /usr/bin >/dev/null
  /usr/bin/env >/dev/null
  command -v openssl >/dev/null 2>&1 && openssl version >/dev/null 2>&1
  sleep 1
done
```

### 4. FreeBSD 弱點檢查

FreeBSD 建議使用原生 VuXML 資料源：

```sh
pkg audit -F
```

Grype / Trivy 通常沒有完整 FreeBSD pkg vulnerability data source；Runtime BOM 仍然可以保存 runtime 證據與 CycloneDX component inventory。

## Demo 解說重點

如果你要錄影片或做簡報，可以用下面這段話解釋：

> RunBOM does not list every package installed on the machine. It observes what actually executes and what libraries are actually mapped during a time window. The result is a Runtime BOM in CycloneDX format, which can be used for audit, runtime dependency review, and downstream vulnerability scanning.

中文說法：

> RunBOM 不是列出整台機器安裝了什麼，而是觀測某段時間內真正被執行、真正被載入的元件。最後輸出 CycloneDX Runtime BOM，讓我們可以做稽核、runtime dependency review，或交給外部工具做弱點掃描。

## 常見 Demo 問題

### BOM 裡東西很少

這通常是正常的。Runtime BOM 只包含觀測期間真的發生的元件。請在 `observe` 執行期間操作你的服務，例如打 API、開網頁、跑批次工作或觸發 plugin/module loading。

### `doctor` 顯示 unsupported

先看 `errors`。最常見原因是：

- 沒有 root 權限。
- Linux kernel 太舊。
- 缺少 BTF。
- tracefs 沒掛載。
- 缺少 `dpkg-query` / `rpm` / `pkg`。
- FreeBSD 沒有載入 DTrace module。

### Grype 顯示沒有弱點

不一定代表真的沒有弱點。Linux OS 套件掃描請確認有加 `--distro`，例如：

```bash
grype sbom:runtime-bom.cdx.json --distro ubuntu:24.04
```

### 可以在 production 跑嗎？

RunBOM 是觀測型工具，不會修改套件或程式。正式環境仍建議先用短時間觀測，例如 `--duration 30` 或 `--duration 60`，確認 host capability 與輸出內容符合預期後再延長觀測時間。
