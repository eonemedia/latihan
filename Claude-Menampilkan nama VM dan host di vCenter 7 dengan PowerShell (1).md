# Menampilkan nama VM dan host di vCenter 7 dengan PowerShell

**Created:** 5/22/2026 11:32:39  
**Updated:** 5/25/2026 15:53:08  
**Exported:** 5/29/2026 8:38:14  
**Link:** [https://claude.ai/chat/7c6e3351-06bc-4c9c-9157-4f7826ab51cf](https://claude.ai/chat/7c6e3351-06bc-4c9c-9157-4f7826ab51cf)  

## User:

> 5/22/2026 11:32:39

pada vcenter 7 bagaimana cara menampilkan nama vm dan host tempat vm itu berada menggunakan powershell

## Assistant:

> 5/22/2026 11:32:54

Berikut beberapa cara menggunakan PowerCLI (PowerShell untuk vCenter):

## Koneksi ke vCenter dulu

```powershell
Connect-VIServer -Server "vcenter-hostname" -User "administrator@vsphere.local" -Password "password"
```

---

## 1. Cara Paling Sederhana

```powershell
Get-VM | Select-Object Name, @{N="Host";E={$_.VMHost.Name}}
```

---

## 2. Dengan Informasi Lebih Lengkap

```powershell
Get-VM | Select-Object Name, 
    @{N="Host";E={$_.VMHost.Name}}, 
    @{N="Cluster";E={$_.VMHost.Parent.Name}},
    PowerState
```

---

## 3. Filter VM Tertentu

```powershell
# Berdasarkan nama VM
Get-VM -Name "nama-vm*" | Select-Object Name, @{N="Host";E={$_.VMHost.Name}}

# Hanya VM yang sedang menyala
Get-VM | Where-Object {$_.PowerState -eq "PoweredOn"} | 
    Select-Object Name, @{N="Host";E={$_.VMHost.Name}}
```

---

## 4. Export ke CSV

```powershell
Get-VM | Select-Object Name, @{N="Host";E={$_.VMHost.Name}} | 
    Export-Csv -Path "C:\vm-host-list.csv" -NoTypeInformation
```

---

## 5. Tampilkan per Host

```powershell
Get-VMHost | ForEach-Object {
    $host = $_
    Get-VM -Location $host | Select-Object Name, 
        @{N="Host";E={$host.Name}}
}
```

---

## Instalasi PowerCLI (jika belum ada)

```powershell
Install-Module -Name VMware.PowerCLI -Scope CurrentUser
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false
```

Perintah paling umum digunakan adalah **nomor 1** — simpel dan langsung menampilkan nama VM beserta host-nya.

## User:

> 5/25/2026 9:54:27

sebutkan perintah yang paling umum digunakan

## Assistant:

> 5/25/2026 9:54:35

## Perintah PowerCLI Paling Umum di vCenter 7

### 🔌 Koneksi
```powershell
Connect-VIServer -Server "vcenter-ip"          # Konek ke vCenter
Disconnect-VIServer -Confirm:$false            # Putus koneksi
```

---

### 🖥️ VM Management
```powershell
Get-VM                                         # Tampilkan semua VM
Get-VM -Name "nama-vm"                         # Cari VM tertentu
Start-VM -VM "nama-vm"                         # Nyalakan VM
Stop-VM -VM "nama-vm" -Confirm:$false          # Matikan VM (hard)
Shutdown-VMGuest -VM "nama-vm" -Confirm:$false # Matikan VM (graceful)
Restart-VM -VM "nama-vm" -Confirm:$false       # Restart VM
Suspend-VM -VM "nama-vm"                       # Suspend VM
```

---

### 🏠 Host Management
```powershell
Get-VMHost                                     # Tampilkan semua host
Get-VMHost -Name "nama-host"                   # Cari host tertentu
Get-VMHost | Select Name, PowerState, Version  # Info dasar host
```

---

### 📊 Informasi & Monitoring
```powershell
# VM + Host nya
Get-VM | Select Name, @{N="Host";E={$_.VMHost.Name}}, PowerState

# Resource usage
Get-VM | Select Name, NumCpu, MemoryGB, PowerState

# Snapshot
Get-VM | Get-Snapshot                          # Lihat semua snapshot
```

---

### 📁 Datastore
```powershell
Get-Datastore                                  # Lihat semua datastore
Get-Datastore | Select Name, FreeSpaceGB, CapacityGB
```

---

### 🔍 Filter & Export
```powershell
# Filter VM yang menyala
Get-VM | Where-Object {$_.PowerState -eq "PoweredOn"}

# Export ke CSV
Get-VM | Select Name, @{N="Host";E={$_.VMHost.Name}} | Export-Csv "vm-list.csv" -NoTypeInformation
```

Perintah **`Get-VM`**, **`Get-VMHost`**, dan **`Get-Datastore`** adalah yang paling sering dipakai sehari-hari.

## User:

> 5/25/2026 9:57:20

buat untuk menampilkan data vm berikut :

* VM_Name
* VM_Folder
* Cluster
* Host
* Power_State
* Guest_OS
* vCPU
* Memory_GB
* IP_Address
* Tools_Status
* Datastore
* Used_GB
* Provisioned_GB

## Assistant:

> 5/25/2026 9:57:34

```powershell
Connect-VIServer -Server "vcenter-ip" -User "administrator@vsphere.local" -Password "password"

Get-VM | Select-Object `
    @{N="VM_Name";        E={$_.Name}},
    @{N="VM_Folder";      E={$_.Folder.Name}},
    @{N="Cluster";        E={
        $parent = $_.VMHost.Parent
        if ($parent -is [VMware.VimAutomation.ViCore.Types.V1.Inventory.Cluster]) {
            $parent.Name
        } else {
            "Standalone"
        }
    }},
    @{N="Host";           E={$_.VMHost.Name}},
    @{N="Power_State";    E={$_.PowerState}},
    @{N="Guest_OS";       E={$_.ExtensionData.Config.GuestFullName}},
    @{N="vCPU";           E={$_.NumCpu}},
    @{N="Memory_GB";      E={[math]::Round($_.MemoryGB, 2)}},
    @{N="IP_Address";     E={
        ($_.Guest.IPAddress | Where-Object {$_ -match '^\d+\.\d+\.\d+\.\d+$'}) -join ", "
    }},
    @{N="Tools_Status";   E={$_.ExtensionData.Guest.ToolsStatus}},
    @{N="Datastore";      E={
        ($_ | Get-Datastore | Select-Object -ExpandProperty Name) -join ", "
    }},
    @{N="Used_GB";        E={[math]::Round($_.UsedSpaceGB, 2)}},
    @{N="Provisioned_GB"; E={[math]::Round($_.ProvisionedSpaceGB, 2)}} `
| Sort-Object Cluster, Host, VM_Name `
| Export-Csv -Path "C:\vm-inventory.csv" -NoTypeInformation -Encoding UTF8

Write-Host "Done! File saved to C:\vm-inventory.csv" -ForegroundColor Green
```

---

### Jika ingin tampil di layar (tanpa export):
```powershell
Get-VM | Select-Object `
    @{N="VM_Name";        E={$_.Name}},
    @{N="VM_Folder";      E={$_.Folder.Name}},
    @{N="Cluster";        E={
        $parent = $_.VMHost.Parent
        if ($parent -is [VMware.VimAutomation.ViCore.Types.V1.Inventory.Cluster]) {
            $parent.Name
        } else { "Standalone" }
    }},
    @{N="Host";           E={$_.VMHost.Name}},
    @{N="Power_State";    E={$_.PowerState}},
    @{N="Guest_OS";       E={$_.ExtensionData.Config.GuestFullName}},
    @{N="vCPU";           E={$_.NumCpu}},
    @{N="Memory_GB";      E={[math]::Round($_.MemoryGB, 2)}},
    @{N="IP_Address";     E={
        ($_.Guest.IPAddress | Where-Object {$_ -match '^\d+\.\d+\.\d+\.\d+$'}) -join ", "
    }},
    @{N="Tools_Status";   E={$_.ExtensionData.Guest.ToolsStatus}},
    @{N="Datastore";      E={($_ | Get-Datastore | Select-Object -ExpandProperty Name) -join ", "}},
    @{N="Used_GB";        E={[math]::Round($_.UsedSpaceGB, 2)}},
    @{N="Provisioned_GB"; E={[math]::Round($_.ProvisionedSpaceGB, 2)}} `
| Sort-Object Cluster, Host, VM_Name `
| Format-Table -AutoSize
```

---

### Catatan penting:
| Field | Keterangan |
|---|---|
| `IP_Address` | Hanya muncul jika **VMware Tools** terinstall & running |
| `Guest_OS` | Diambil dari config, bukan deteksi langsung |
| `Tools_Status` | Nilai: `toolsOk`, `toolsOld`, `toolsNotInstalled`, `toolsNotRunning` |
| `Used_GB` vs `Provisioned_GB` | Used = aktual di disk, Provisioned = total dialokasikan (termasuk thin provision) |

## User:

> 5/25/2026 9:59:18

dari vcenter kita bisa mebuat folder VM dan template, apakah bisa menggunakan powercli ? berikan contohnya

## Assistant:

> 5/25/2026 9:59:38

Ya, bisa! PowerCLI mendukung pengelolaan folder VM dan Template. Berikut contohnya:

---

## 📁 Folder Management

### Melihat Folder yang Ada
```powershell
# Tampilkan semua folder
Get-Folder

# Hanya folder tipe VM
Get-Folder -Type VM

# Cari folder tertentu
Get-Folder -Name "Production"
```

### Membuat Folder
```powershell
# Buat folder di root (Datacenter)
New-Folder -Name "Production" -Location (Get-Datacenter "Datacenter-Name")

# Buat folder di dalam folder yang sudah ada
New-Folder -Name "Web-Servers" -Location (Get-Folder -Name "Production")

# Buat nested folder sekaligus
$parent = New-Folder -Name "Production" -Location (Get-Datacenter "Datacenter-Name")
$child  = New-Folder -Name "Web-Servers" -Location $parent
$grand  = New-Folder -Name "Frontend"    -Location $child
```

### Memindahkan & Menghapus Folder
```powershell
# Pindahkan VM ke folder
Move-VM -VM "nama-vm" -Destination (Get-Folder -Name "Production")

# Pindahkan folder ke folder lain
Move-Inventory -Item (Get-Folder "Web-Servers") -Destination (Get-Folder "Production")

# Hapus folder (harus kosong)
Remove-Folder -Folder (Get-Folder "nama-folder") -Confirm:$false
```

---

## 📦 Template Management

### Melihat Template
```powershell
# Tampilkan semua template
Get-Template

# Cari template tertentu
Get-Template -Name "Windows-2022-Template"

# Info detail template
Get-Template | Select-Object Name, 
    @{N="Guest_OS"; E={$_.ExtensionData.Config.GuestFullName}},
    @{N="vCPU";     E={$_.NumCpu}},
    @{N="Memory_GB";E={$_.MemoryGB}}
```

### Konversi VM ↔ Template
```powershell
# Konversi VM menjadi Template
Get-VM -Name "Windows-2022-Base" | Set-VM -ToTemplate -Confirm:$false

# Konversi Template kembali menjadi VM
Set-Template -Template "Windows-2022-Template" -ToVM -Confirm:$false
```

### Deploy VM dari Template
```powershell
# Deploy VM dari template (basic)
New-VM -Name "VM-Baru" `
       -Template (Get-Template "Windows-2022-Template") `
       -VMHost (Get-VMHost "nama-host") `
       -Datastore (Get-Datastore "nama-datastore")

# Deploy dengan folder tujuan
New-VM -Name "VM-Baru" `
       -Template (Get-Template "Windows-2022-Template") `
       -VMHost (Get-VMHost "nama-host") `
       -Datastore (Get-Datastore "nama-datastore") `
       -Location (Get-Folder "Production")

# Deploy dengan customization spec (hostname, IP, dll)
New-VM -Name "VM-Baru" `
       -Template (Get-Template "Windows-2022-Template") `
       -VMHost (Get-VMHost "nama-host") `
       -Datastore (Get-Datastore "nama-datastore") `
       -Location (Get-Folder "Production") `
       -OSCustomizationSpec (Get-OSCustomizationSpec "Windows-Spec")
```

### Memindahkan Template ke Folder
```powershell
# Pindahkan template ke folder tertentu
Move-Template -Template (Get-Template "Windows-2022-Template") `
              -Destination (Get-Folder "Templates")
```

### Kloning Template
```powershell
# Clone template menjadi VM baru
New-VM -Name "VM-Clone" `
       -Template (Get-Template "Windows-2022-Template") `
       -VMHost (Get-VMHost "nama-host") `
       -Datastore (Get-Datastore "nama-datastore") `
       -LinkedClone  # opsional: pakai linked clone
```

---

## 🔄 Contoh Workflow Lengkap

```powershell
# 1. Buat struktur folder
$dc       = Get-Datacenter "Datacenter-Name"
$folderVM = New-Folder -Name "Production"  -Location $dc
$folderTp = New-Folder -Name "Templates"   -Location $dc
New-Folder -Name "Web-Servers" -Location $folderVM
New-Folder -Name "DB-Servers"  -Location $folderVM

# 2. Pindahkan template ke folder Templates
Move-Template -Template (Get-Template "Windows-2022-Template") `
              -Destination $folderTp

# 3. Deploy VM dari template ke folder Production
New-VM -Name "Web-01" `
       -Template (Get-Template "Windows-2022-Template") `
       -VMHost (Get-VMHost "nama-host") `
       -Datastore (Get-Datastore "nama-datastore") `
       -Location (Get-Folder "Web-Servers")

# 4. Nyalakan VM
Start-VM -VM "Web-01"
```

---

### Ringkasan Perintah

| Aksi | Perintah |
|---|---|
| Lihat folder | `Get-Folder` |
| Buat folder | `New-Folder` |
| Hapus folder | `Remove-Folder` |
| Lihat template | `Get-Template` |
| VM → Template | `Set-VM -ToTemplate` |
| Template → VM | `Set-Template -ToVM` |
| Deploy dari template | `New-VM -Template` |
| Pindah template | `Move-Template` |

## User:

> 5/25/2026 10:01:17

buat script powercli untuk create banyak folder inputnya dari file csv/txt.

## Assistant:

> 5/25/2026 10:01:48

## Script PowerCLI - Create Folder dari File CSV

### 📄 Format File CSV (`folders.csv`)

```csv
FolderName,ParentFolder,Datacenter,FolderType
Production,,Datacenter-01,VM
Development,,Datacenter-01,VM
Staging,,Datacenter-01,VM
Web-Servers,Production,Datacenter-01,VM
DB-Servers,Production,Datacenter-01,VM
App-Servers,Production,Datacenter-01,VM
Web-Dev,Development,Datacenter-01,VM
DB-Dev,Development,Datacenter-01,VM
Templates,,Datacenter-01,VM
Win-Templates,Templates,Datacenter-01,VM
Linux-Templates,Templates,Datacenter-01,VM
```

> **Catatan:** Kolom `ParentFolder` dikosongkan jika folder dibuat di root Datacenter.

---

### 📝 Script PowerCLI (`Create-Folders.ps1`)

```powershell
# ============================================================
# Script  : Create-Folders.ps1
# Fungsi  : Membuat folder vCenter dari file CSV
# vCenter : 7.x
# ============================================================

param(
    [string]$vCenterServer = "vcenter-ip",
    [string]$Username      = "administrator@vsphere.local",
    [string]$Password      = "password",
    [string]$CsvFile       = ".\folders.csv"
)

# ─── Fungsi Logging ─────────────────────────────────────────
function Write-Log {
    param([string]$Message, [string]$Status = "INFO")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $color = switch ($Status) {
        "OK"      { "Green" }
        "SKIP"    { "Yellow" }
        "ERROR"   { "Red" }
        default   { "Cyan" }
    }
    Write-Host "[$timestamp] [$Status] $Message" -ForegroundColor $color
}

# ─── Fungsi Get or Create Folder ────────────────────────────
function Get-OrCreate-Folder {
    param(
        [string]$FolderName,
        $ParentLocation
    )

    # Cek apakah folder sudah ada di dalam parent
    $existing = Get-Folder -Name $FolderName -Location $ParentLocation -ErrorAction SilentlyContinue |
                Where-Object { $_.Parent.Name -eq $ParentLocation.Name -or 
                               $_.ParentId -eq $ParentLocation.Id }

    if ($existing) {
        Write-Log "Folder '$FolderName' sudah ada di '$($ParentLocation.Name)' - dilewati" "SKIP"
        return $existing
    }

    try {
        $newFolder = New-Folder -Name $FolderName -Location $ParentLocation -ErrorAction Stop
        Write-Log "Folder '$FolderName' berhasil dibuat di '$($ParentLocation.Name)'" "OK"
        return $newFolder
    }
    catch {
        Write-Log "Gagal membuat folder '$FolderName' : $_" "ERROR"
        return $null
    }
}

# ─── Main Script ────────────────────────────────────────────

# Validasi file CSV
if (-not (Test-Path $CsvFile)) {
    Write-Log "File CSV tidak ditemukan: $CsvFile" "ERROR"
    exit 1
}

# Koneksi ke vCenter
Write-Log "Menghubungkan ke vCenter: $vCenterServer" "INFO"
try {
    Connect-VIServer -Server $vCenterServer `
                     -User $Username `
                     -Password $Password `
                     -ErrorAction Stop | Out-Null
    Write-Log "Berhasil terhubung ke $vCenterServer" "OK"
}
catch {
    Write-Log "Gagal koneksi ke vCenter: $_" "ERROR"
    exit 1
}

# Baca CSV
$folders = Import-Csv -Path $CsvFile
Write-Log "Total folder yang akan diproses: $($folders.Count)" "INFO"
Write-Log ("─" * 60) "INFO"

# Counter
$created  = 0
$skipped  = 0
$failed   = 0

# Proses setiap baris CSV
foreach ($row in $folders) {

    $folderName  = $row.FolderName.Trim()
    $parentName  = $row.ParentFolder.Trim()
    $dcName      = $row.Datacenter.Trim()

    # Ambil Datacenter
    $dc = Get-Datacenter -Name $dcName -ErrorAction SilentlyContinue
    if (-not $dc) {
        Write-Log "Datacenter '$dcName' tidak ditemukan untuk folder '$folderName'" "ERROR"
        $failed++
        continue
    }

    # Tentukan lokasi parent
    if ([string]::IsNullOrEmpty($parentName)) {
        # Parent = root Datacenter
        $parentLocation = $dc
    }
    else {
        # Cari parent folder
        $parentLocation = Get-Folder -Name $parentName -Location $dc -ErrorAction SilentlyContinue |
                          Select-Object -First 1

        if (-not $parentLocation) {
            Write-Log "Parent folder '$parentName' tidak ditemukan untuk '$folderName'" "ERROR"
            $failed++
            continue
        }
    }

    # Buat folder
    $result = Get-OrCreate-Folder -FolderName $folderName -ParentLocation $parentLocation

    if ($result) {
        # Cek apakah baru dibuat atau sudah ada
        if ($result.Name -eq $folderName) {
            $created++
        }
    }
    else {
        $failed++
    }
}

# ─── Summary ────────────────────────────────────────────────
Write-Log ("─" * 60) "INFO"
Write-Log "SELESAI - Summary:" "INFO"
Write-Log "  Total diproses : $($folders.Count)" "INFO"
Write-Log "  Berhasil dibuat: $created" "OK"
Write-Log "  Sudah ada (skip): $skipped" "SKIP"
Write-Log "  Gagal          : $failed" "ERROR"

# Disconnect
Disconnect-VIServer -Server $vCenterServer -Confirm:$false
Write-Log "Koneksi ke vCenter ditutup" "INFO"
```

---

### ▶️ Cara Menjalankan

```powershell
# Cara 1 - Jalankan langsung dengan parameter default
.\Create-Folders.ps1

# Cara 2 - Dengan parameter custom
.\Create-Folders.ps1 -vCenterServer "192.168.1.10" `
                     -Username "administrator@vsphere.local" `
                     -Password "P@ssword123" `
                     -CsvFile ".\folders.csv"
```

---

### 📋 Contoh Output

```
[2024-01-15 09:00:01] [INFO] Menghubungkan ke vCenter: 192.168.1.10
[2024-01-15 09:00:03] [OK]   Berhasil terhubung ke 192.168.1.10
[2024-01-15 09:00:03] [INFO] Total folder yang akan diproses: 11
[2024-01-15 09:00:03] [INFO] ────────────────────────────────────────────────────────────
[2024-01-15 09:00:04] [OK]   Folder 'Production' berhasil dibuat di 'Datacenter-01'
[2024-01-15 09:00:04] [OK]   Folder 'Development' berhasil dibuat di 'Datacenter-01'
[2024-01-15 09:00:05] [OK]   Folder 'Staging' berhasil dibuat di 'Datacenter-01'
[2024-01-15 09:00:05] [OK]   Folder 'Web-Servers' berhasil dibuat di 'Production'
[2024-01-15 09:00:05] [SKIP] Folder 'DB-Servers' sudah ada di 'Production' - dilewati
[2024-01-15 09:00:06] [OK]   Folder 'App-Servers' berhasil dibuat di 'Production'
[2024-01-15 09:00:06] [INFO] ────────────────────────────────────────────────────────────
[2024-01-15 09:00:06] [INFO] SELESAI - Summary:
[2024-01-15 09:00:06] [INFO]   Total diproses  : 11
[2024-01-15 09:00:06] [OK]    Berhasil dibuat  : 10
[2024-01-15 09:00:06] [SKIP]  Sudah ada (skip) : 1
[2024-01-15 09:00:06] [ERROR] Gagal            : 0
```

---

### 💡 Tips Tambahan

| Kebutuhan | Solusi |
|---|---|
| Nested folder lebih dari 2 level | Urutkan CSV dari parent ke child |
| Simpan log ke file | Tambahkan `\| Tee-Object -FilePath "log.txt"` |
| Dry run (tanpa eksekusi) | Tambahkan parameter `-WhatIf` di `New-Folder` |
| Banyak Datacenter berbeda | Isi kolom `Datacenter` sesuai masing-masing baris |

## User:

> 5/25/2026 10:04:39

buat 2 panduan , pertama kusus folder management dan yang kedua selain dari folder management. gunakan memory saya

## Assistant:

> 5/25/2026 10:06:31

Saya akan buat 2 file panduan sesuai preferensi kamu. Mari saya cek memory dulu.

> Viewed file, used tool
>
> - Managing memory...
> - Cek skill markdown
> - **Done**

Baik, langsung saya buat kedua file `.md` dengan metadata YAML sesuai preferensi kamu.

> Created 2 files, presented file
>
> - Panduan PowerCLI - Folder Management vCenter 7
> - Panduan PowerCLI - VM, Template, Host, Datastore & Monitoring vCenter 7
> - Presenting file(s)...
> - **Done**

Kedua panduan sudah siap dengan metadata YAML lengkap:

**📄 File 1 — `2026-05-25-powercli-folder-management.md`**
Khusus Folder Management, mencakup: melihat folder, membuat folder (tunggal & nested), memindahkan item, menghapus folder, dan bulk create dari CSV.

**📄 File 2 — `2026-05-25-powercli-vm-template-host-datastore.md`**
Mencakup selain folder management: VM Management (power, snapshot, modifikasi), Template Management (konversi, deploy, bulk deploy), Host Management, Datastore Management, Monitoring, dan Export Inventory VM lengkap.

> File: [/mnt/user-data/outputs/2026-05-25-powercli-folder-management.md](https://claude.ai/api/organizations/8228179f-910d-454e-9967-59412805a8ef/conversations/7c6e3351-06bc-4c9c-9157-4f7826ab51cf/wiggle/download-file?path=%2Fmnt%2Fuser-data%2Foutputs%2F2026-05-25-powercli-folder-management.md)

> File: [/mnt/user-data/outputs/2026-05-25-powercli-vm-template-host-datastore.md](https://claude.ai/api/organizations/8228179f-910d-454e-9967-59412805a8ef/conversations/7c6e3351-06bc-4c9c-9157-4f7826ab51cf/wiggle/download-file?path=%2Fmnt%2Fuser-data%2Foutputs%2F2026-05-25-powercli-vm-template-host-datastore.md)

## User:

> 5/25/2026 15:52:35

# ============================================================
# Script  : Create-Folders.ps1
# Fungsi  : Membuat folder vCenter dari file CSV
# vCenter : 7.x
# ============================================================
param(
    [string]$vCenterServer = "vcenter-ip",
    [string]$Username      = "administrator@vsphere.local",
    [string]$Password      = "password",
    [string]$CsvFile       = ".\folders.csv"
)
function Write-Log {
    param([string]$Message, [string]$Status = "INFO")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $color = switch ($Status) {
        "OK"    { "Green" }
        "SKIP"  { "Yellow" }
        "ERROR" { "Red" }
        default { "Cyan" }
    }
    Write-Host "[$timestamp] [$Status] $Message" -ForegroundColor $color
}
function Get-OrCreate-Folder {
    param([string]$FolderName, $ParentLocation)
    $existing = Get-Folder -Name $FolderName -Location $ParentLocation `
                           -ErrorAction SilentlyContinue | Select-Object -First 1
    if ($existing) {
        Write-Log "Folder '$FolderName' sudah ada di '$($ParentLocation.Name)' - dilewati" "SKIP"
        return $existing
    }
    try {
        $new = New-Folder -Name $FolderName -Location $ParentLocation -ErrorAction Stop
        Write-Log "Folder '$FolderName' berhasil dibuat di '$($ParentLocation.Name)'" "OK"
        return $new
    }
    catch {
        Write-Log "Gagal membuat folder '$FolderName': $_" "ERROR"
        return $null
    }
}
# Validasi file CSV
if (-not (Test-Path $CsvFile)) {
    Write-Log "File CSV tidak ditemukan: $CsvFile" "ERROR"; exit 1
}
# Koneksi vCenter
try {
    Connect-VIServer -Server $vCenterServer -User $Username `
                     -Password $Password -ErrorAction Stop | Out-Null
    Write-Log "Berhasil terhubung ke $vCenterServer" "OK"
}
catch {
    Write-Log "Gagal koneksi: $_" "ERROR"; exit 1
}
$folders = Import-Csv -Path $CsvFile
$created = 0; $skipped = 0; $failed = 0
foreach ($row in $folders) {
    $folderName = $row.FolderName.Trim()
    $parentName = $row.ParentFolder.Trim()
    $dcName     = $row.Datacenter.Trim()
    $dc = Get-Datacenter -Name $dcName -ErrorAction SilentlyContinue
    if (-not $dc) {
        Write-Log "Datacenter '$dcName' tidak ditemukan" "ERROR"; $failed++; continue
    }
    $parentLocation = if ([string]::IsNullOrEmpty($parentName)) {
        $dc
    } else {
        Get-Folder -Name $parentName -Location $dc -ErrorAction SilentlyContinue |
        Select-Object -First 1
    }
    if (-not $parentLocation) {
        Write-Log "Parent '$parentName' tidak ditemukan untuk '$folderName'" "ERROR"
        $failed++; continue
    }
    $result = Get-OrCreate-Folder -FolderName $folderName -ParentLocation $parentLocation
    if ($result) { $created++ } else { $failed++ }
}
Write-Log ("─" * 50) "INFO"
Write-Log "Total: $($folders.Count) | Dibuat: $created | Gagal: $failed" "INFO"
Disconnect-VIServer -Server $vCenterServer -Confirm:$false
Write-Log "Koneksi ditutup" "INFO"

modifikasi scriptnya untuk tampilkan cara penggunaan dan tambahkan satu parameter untuk custom port

## Assistant:

> 5/25/2026 15:53:08

```powershell
# ============================================================
# Script  : Create-Folders.ps1
# Fungsi  : Membuat folder vCenter dari file CSV
# vCenter : 7.x
# ============================================================
param(
    [string]$vCenterServer = "",
    [string]$Username      = "administrator@vsphere.local",
    [string]$Password      = "password",
    [string]$CsvFile       = ".\folders.csv",
    [int]$Port             = 443,
    [switch]$Help
)

# ─── Fungsi Help / Cara Penggunaan ──────────────────────────
function Show-Usage {
    $script = $MyInvocation.ScriptName
    if (-not $script) { $script = "Create-Folders.ps1" }

    Write-Host ""
    Write-Host "═══════════════════════════════════════════════════════════" -ForegroundColor Cyan
    Write-Host "   Create-Folders.ps1 - Bulk Folder Creator untuk vCenter  " -ForegroundColor Cyan
    Write-Host "═══════════════════════════════════════════════════════════" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "PENGGUNAAN:" -ForegroundColor Yellow
    Write-Host "   .\Create-Folders.ps1 [parameter]" -ForegroundColor White
    Write-Host ""
    Write-Host "PARAMETER:" -ForegroundColor Yellow
    Write-Host "   -vCenterServer   " -NoNewline -ForegroundColor Green
    Write-Host "<string>   IP atau hostname vCenter                  [WAJIB]"
    Write-Host "   -Username        " -NoNewline -ForegroundColor Green
    Write-Host "<string>   Username login vCenter                    [default: administrator@vsphere.local]"
    Write-Host "   -Password        " -NoNewline -ForegroundColor Green
    Write-Host "<string>   Password login vCenter                    [default: password]"
    Write-Host "   -CsvFile         " -NoNewline -ForegroundColor Green
    Write-Host "<string>   Path ke file CSV daftar folder            [default: .\folders.csv]"
    Write-Host "   -Port            " -NoNewline -ForegroundColor Green
    Write-Host "<int>      Port koneksi ke vCenter                   [default: 443]"
    Write-Host "   -Help            " -NoNewline -ForegroundColor Green
    Write-Host "           Tampilkan panduan penggunaan ini"
    Write-Host ""
    Write-Host "CONTOH:" -ForegroundColor Yellow
    Write-Host "   # Penggunaan minimal (wajib)" -ForegroundColor DarkGray
    Write-Host "   .\Create-Folders.ps1 -vCenterServer 192.168.1.10" -ForegroundColor White
    Write-Host ""
    Write-Host "   # Dengan username dan password custom" -ForegroundColor DarkGray
    Write-Host "   .\Create-Folders.ps1 -vCenterServer 192.168.1.10 ``" -ForegroundColor White
    Write-Host "                        -Username admin@vsphere.local ``" -ForegroundColor White
    Write-Host "                        -Password P@ssword123" -ForegroundColor White
    Write-Host ""
    Write-Host "   # Dengan port custom (misal: non-standard port)" -ForegroundColor DarkGray
    Write-Host "   .\Create-Folders.ps1 -vCenterServer 192.168.1.10 ``" -ForegroundColor White
    Write-Host "                        -Port 8443 ``" -ForegroundColor White
    Write-Host "                        -CsvFile .\data\folders.csv" -ForegroundColor White
    Write-Host ""
    Write-Host "   # Semua parameter lengkap" -ForegroundColor DarkGray
    Write-Host "   .\Create-Folders.ps1 -vCenterServer 192.168.1.10 ``" -ForegroundColor White
    Write-Host "                        -Username admin@vsphere.local ``" -ForegroundColor White
    Write-Host "                        -Password P@ssword123 ``" -ForegroundColor White
    Write-Host "                        -Port 8443 ``" -ForegroundColor White
    Write-Host "                        -CsvFile .\data\folders.csv" -ForegroundColor White
    Write-Host ""
    Write-Host "FORMAT CSV:" -ForegroundColor Yellow
    Write-Host "   FolderName,ParentFolder,Datacenter,FolderType" -ForegroundColor White
    Write-Host "   Production,,Datacenter-01,VM" -ForegroundColor White
    Write-Host "   Web-Servers,Production,Datacenter-01,VM" -ForegroundColor White
    Write-Host "   DB-Servers,Production,Datacenter-01,VM" -ForegroundColor White
    Write-Host ""
    Write-Host "   Catatan: Kosongkan kolom ParentFolder jika folder" -ForegroundColor DarkGray
    Write-Host "            dibuat langsung di root Datacenter." -ForegroundColor DarkGray
    Write-Host "            Urutkan CSV dari parent ke child." -ForegroundColor DarkGray
    Write-Host ""
    Write-Host "═══════════════════════════════════════════════════════════" -ForegroundColor Cyan
    Write-Host ""
}

# Tampilkan help jika -Help atau tidak ada vCenterServer
if ($Help -or [string]::IsNullOrEmpty($vCenterServer)) {
    Show-Usage
    exit 0
}

# ─── Fungsi Logging ─────────────────────────────────────────
function Write-Log {
    param([string]$Message, [string]$Status = "INFO")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $color = switch ($Status) {
        "OK"    { "Green" }
        "SKIP"  { "Yellow" }
        "ERROR" { "Red" }
        default { "Cyan" }
    }
    Write-Host "[$timestamp] [$Status] $Message" -ForegroundColor $color
}

# ─── Fungsi Get or Create Folder ────────────────────────────
function Get-OrCreate-Folder {
    param([string]$FolderName, $ParentLocation)

    $existing = Get-Folder -Name $FolderName -Location $ParentLocation `
                           -ErrorAction SilentlyContinue | Select-Object -First 1
    if ($existing) {
        Write-Log "Folder '$FolderName' sudah ada di '$($ParentLocation.Name)' - dilewati" "SKIP"
        return $existing
    }
    try {
        $new = New-Folder -Name $FolderName -Location $ParentLocation -ErrorAction Stop
        Write-Log "Folder '$FolderName' berhasil dibuat di '$($ParentLocation.Name)'" "OK"
        return $new
    }
    catch {
        Write-Log "Gagal membuat folder '$FolderName': $_" "ERROR"
        return $null
    }
}

# ─── Validasi File CSV ───────────────────────────────────────
if (-not (Test-Path $CsvFile)) {
    Write-Log "File CSV tidak ditemukan: $CsvFile" "ERROR"
    Write-Log "Jalankan '.\Create-Folders.ps1 -Help' untuk panduan penggunaan" "INFO"
    exit 1
}

# ─── Koneksi vCenter ─────────────────────────────────────────
Write-Log "Menghubungkan ke $vCenterServer (Port: $Port)..." "INFO"
try {
    Connect-VIServer -Server $vCenterServer `
                     -Port $Port `
                     -User $Username `
                     -Password $Password `
                     -ErrorAction Stop | Out-Null
    Write-Log "Berhasil terhubung ke $vCenterServer`:$Port" "OK"
}
catch {
    Write-Log "Gagal koneksi ke $vCenterServer`:$Port - $_" "ERROR"
    exit 1
}

# ─── Proses CSV ──────────────────────────────────────────────
$folders = Import-Csv -Path $CsvFile
Write-Log "Total folder yang akan diproses: $($folders.Count)" "INFO"
Write-Log ("─" * 50) "INFO"

$created = 0; $skipped = 0; $failed = 0

foreach ($row in $folders) {
    $folderName = $row.FolderName.Trim()
    $parentName = $row.ParentFolder.Trim()
    $dcName     = $row.Datacenter.Trim()

    $dc = Get-Datacenter -Name $dcName -ErrorAction SilentlyContinue
    if (-not $dc) {
        Write-Log "Datacenter '$dcName' tidak ditemukan untuk folder '$folderName'" "ERROR"
        $failed++; continue
    }

    $parentLocation = if ([string]::IsNullOrEmpty($parentName)) {
        $dc
    } else {
        Get-Folder -Name $parentName -Location $dc -ErrorAction SilentlyContinue |
        Select-Object -First 1
    }

    if (-not $parentLocation) {
        Write-Log "Parent '$parentName' tidak ditemukan untuk '$folderName'" "ERROR"
        $failed++; continue
    }

    $result = Get-OrCreate-Folder -FolderName $folderName -ParentLocation $parentLocation
    if ($result) { $created++ } else { $failed++ }
}

# ─── Summary ─────────────────────────────────────────────────
Write-Log ("─" * 50) "INFO"
Write-Log "SELESAI - Summary:" "INFO"
Write-Log "  Total diproses : $($folders.Count)" "INFO"
Write-Log "  Berhasil dibuat: $created" "OK"
Write-Log "  Gagal          : $failed" "ERROR"

Disconnect-VIServer -Server $vCenterServer -Confirm:$false
Write-Log "Koneksi ditutup" "INFO"
```

---

### Perubahan yang ditambahkan:

| # | Perubahan | Detail |
|---|---|---|
| 1 | Parameter `-Port` | Default `443`, bisa diubah ke port custom misal `8443` |
| 2 | Parameter `-Help` | Switch untuk tampilkan panduan |
| 3 | `Show-Usage` function | Tampil otomatis jika `-Help` atau `-vCenterServer` tidak diisi |
| 4 | `-vCenterServer` default kosong | Agar script otomatis tampilkan help jika dijalankan tanpa parameter |
| 5 | Port di log koneksi | Pesan log kini menyertakan port yang digunakan |

Powered by Claude Exporter (https://www.ai-chat-exporter.net)
