# How-bring-back-ReadyBoost-after-its-removal-in-Windows-11
Open PowerShell(Admin)
And run the script given the readyboost cache file will there 
first plug pendrive then run the script and then unplug the pendrive for 10 seconds
and after that plug it in and there you go

# Title: ReadyBoost Manager (Interactive)
# Description: Enables SysMain, forces compatibility, and allows "Dedicate" or "Custom" sizing.
# Author: Your Programming Partner
# Version: 3.0

# --- START OF SCRIPT ---

# 1. Verify Administrator Privileges
if (-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Write-Warning "Access Denied. Run as Administrator."
    Start-Sleep -Seconds 5
    exit
}

Clear-Host
Write-Host "--- Advanced ReadyBoost Manager ---" -ForegroundColor Cyan
Write-Host "This tool fixes missing tabs and lets you configure the cache size."
Write-Host ""

# 2. Fix SysMain Service (Required for ReadyBoost)
$sysMain = Get-Service -Name "SysMain" -ErrorAction SilentlyContinue
if ($sysMain.Status -ne 'Running') {
    Write-Host "Starting SysMain service..." -ForegroundColor Yellow
    try {
        Set-Service -Name "SysMain" -StartupType Automatic
        Start-Service -Name "SysMain"
        Write-Host "SysMain started." -ForegroundColor Green
    }
    catch {
        Write-Host "Could not start SysMain. ReadyBoost requires this service." -ForegroundColor Red
        Start-Sleep -Seconds 5
        exit
    }
}

Write-Host ""

# 3. Select Drive
$removableDrives = Get-Volume | Where-Object { $_.DriveType -eq 'Removable' -and $_.FileSystem -in ('NTFS', 'FAT32', 'exFAT') }

if (-not $removableDrives) {
    Write-Host "No removable USB drives found." -ForegroundColor Red
    Start-Sleep -Seconds 5
    exit
}

Write-Host "Available Drives:" -ForegroundColor Cyan
$i = 1
foreach ($d in $removableDrives) {
    $sizeGB = [math]::Round($d.Size / 1GB, 2)
    $freeGB = [math]::Round($d.SizeRemaining / 1GB, 2)
    Write-Host "[$i] Drive $($d.DriveLetter): ($($d.FileSystemLabel)) - $freeGB GB Free of $sizeGB GB [$($d.FileSystem)]"
    $i++
}

$selection = Read-Host "`nEnter the number of the drive to configure (1-$($removableDrives.Count))"
try {
    $selectedVolume = $removableDrives[$selection - 1]
    if (-not $selectedVolume) { throw "Invalid" }
}
catch {
    Write-Host "Invalid selection." -ForegroundColor Red
    exit
}

# 4. Calculate Limits
# FAT32 cannot handle files larger than 4GB (4096MB). 
# NTFS/exFAT ReadyBoost limit is 32GB (32768MB).
$freeSpaceMB = [math]::Round($selectedVolume.SizeRemaining / 1MB)
$systemLimitMB = if ($selectedVolume.FileSystem -eq "FAT32") { 4094 } else { 32768 }

# The actual max is the smaller of: Free Space OR System Limit
$maxPossibleMB = [math]::Min($freeSpaceMB - 100, $systemLimitMB) 

if ($maxPossibleMB -lt 250) {
    Write-Host "Error: Not enough free space. Minimum 250MB required." -ForegroundColor Red
    exit
}

# 5. Choose Mode (Dedicate vs Custom)
Write-Host ""
Write-Host "--- Configuration for Drive $($selectedVolume.DriveLetter): ---" -ForegroundColor White
Write-Host "[1] Dedicate this device to ReadyBoost" -ForegroundColor Green
Write-Host "    (Uses maximum available space: $maxPossibleMB MB)"
Write-Host "[2] Use this device (Custom size)" -ForegroundColor Yellow
Write-Host "[3] Do not use this device (Disable)" -ForegroundColor Gray

$mode = Read-Host "Select an option [1-3]"
$finalSizeMB = 0
$deviceStatus = 2 # Default to Enabled

switch ($mode) {
    "1" { 
        # Dedicate
        $finalSizeMB = $maxPossibleMB
        Write-Host "Selected: Dedicate Device ($finalSizeMB MB)" -ForegroundColor Green
    }
    "2" {
        # Custom
        do {
            try {
                $inputSize = Read-Host "Enter size in MB (250 - $maxPossibleMB)"
                $finalSizeMB = [int]$inputSize
            } catch { $finalSizeMB = 0 }
            
            if ($finalSizeMB -lt 250 -or $finalSizeMB -gt $maxPossibleMB) {
                Write-Host "Invalid size. Must be between 250 and $maxPossibleMB." -ForegroundColor Red
            }
        } until ($finalSizeMB -ge 250 -and $finalSizeMB -le $maxPossibleMB)
        Write-Host "Selected: Custom Size ($finalSizeMB MB)" -ForegroundColor Yellow
    }
    "3" {
        # Disable
        $deviceStatus = 1 # 1 usually means recognized but disabled for RB
        Write-Host "Selected: Disable ReadyBoost" -ForegroundColor Gray
    }
    default {
        Write-Host "Invalid selection. Exiting."
        exit
    }
}

# 6. Apply to Registry
Write-Host "Searching Registry..." -ForegroundColor White

# Get Volume Serial to find correct key
$volInfo = Get-CimInstance -ClassName Win32_LogicalDisk | Where-Object { $_.DeviceID -eq "$($selectedVolume.DriveLetter):" }
$volSerial = $volInfo.VolumeSerialNumber

$baseRegPath = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\EMDMgmt"
$foundKeys = Get-ChildItem -Path $baseRegPath -ErrorAction SilentlyContinue
$targetKey = $null

foreach ($key in $foundKeys) {
    if ($key.Name -match $selectedVolume.FileSystemLabel -or $key.Name -match $volSerial) {
        $targetKey = $key.PSPath
        break
    }
}

if (-not $targetKey) {
    # If key doesn't exist, we can try to find a generic USB key or warn user
    Write-Host "Registry key not found. Windows must scan the drive first." -ForegroundColor Red
    Write-Host "Please open the drive properties in Explorer once, then run this script again."
    Start-Sleep -Seconds 5
    exit
}

Write-Host "Updating Registry Key: $targetKey" -ForegroundColor Cyan

try {
    # Set Status
    Set-ItemProperty -Path $targetKey -Name "DeviceStatus" -Value $deviceStatus -Type DWord -Force
    
    # Force Speed Checks (Spoofing) to ensure tab is available
    Set-ItemProperty -Path $targetKey -Name "ReadSpeedKBs" -Value 32000 -Type DWord -Force
    Set-ItemProperty -Path $targetKey -Name "WriteSpeedKBs" -Value 32000 -Type DWord -Force
    Set-ItemProperty -Path $targetKey -Name "DoNotRemind" -Value 0 -Type DWord -Force

    # Set Size if enabled
    if ($deviceStatus -eq 2) {
        Set-ItemProperty -Path $targetKey -Name "CacheSizeInMB" -Value $finalSizeMB -Type DWord -Force
        # CacheStatus 1 means "Healthy/Active"
        Set-ItemProperty -Path $targetKey -Name "CacheStatus" -Value 1 -Type DWord -Force
        
        # PhysicalDeviceSizeMB helps Windows know the limit
        Set-ItemProperty -Path $targetKey -Name "PhysicalDeviceSizeMB" -Value ([math]::Round($selectedVolume.Size/1MB)) -Type DWord -Force
    }

    Write-Host "Success! Configuration updated." -ForegroundColor Green
}
catch {
    Write-Host "Registry Error: $($_.Exception.Message)" -ForegroundColor Red
}

# 7. Final Instructions
Write-Host ""
Write-Host "--- DONE ---" -ForegroundColor Cyan
Write-Host "1. Unplug your USB drive."
Write-Host "2. Wait 5 seconds."
Write-Host "3. Plug it back in."
Write-Host "4. If asked, choose 'Speed up my system' or check Properties -> ReadyBoost."
Write-Host "   (The settings you chose should now be active/default)."
Write-Host ""
Write-Host "Press any key to exit..."
$Host.UI.RawUI.ReadKey("NoEcho,IncludeKeyUp") | Out-Null
