param (
    [Parameter()]
    [int]$TaskType)

# Global Variable
 
# ChangeMe (Server)
$PoolType = "MANUAL"
$vcaddress = "192.168.4.10"
$caddress = "192.168.4.12"

# changeMe (account)
$vcuser = "administrator@vsphere.local"
$vcpass = "SolVMware1!"
$cuser = "lk\administrator"
$cpass = "SolVMware1!"

# functions
function WriteLog($message)
{
Write-Host " ... $message" -ForegroundColor DarkGray
}

function ShutdownDesktopPool($reportFile)
{
$results = @()
$count = 0
$startTime = Get-Date

$pools = Get-hvPool -PoolType $PoolType | Sort-Object -Property id

Foreach ($pool In $pools)
{
$isNeedToWait = $false

Write-Host
" [{0}] {1}" -f $pool.base.Name, $pool.base.DisplayName

WriteLog "Get Desktop In Pool"



WriteLog "Get VM In Pool"
$desktopvms =Get-HVMachineSummary -PoolName $pool.base.Name | Sort-Object -Property Name

WriteLog "Get Current Session"
$sessions = Get-HVMachineSummary -PoolName $pool.base.name -State CONNECTED -ErrorAction SilentlyContinue | Sort-Object -Property HostName

# Shutdown VM
WriteLog "Shutdown Starting"

if ($desktopVMs)
{
Foreach ($desktopVM In $desktopVMs)
{
$isConnectedVM = $false
$item = "" | Select-Object PoolName, DesktopName, Task, TaskResult

if ($sessions)
{
Foreach ($session In $sessions)
{
if ($desktopVM.base.name.ToLower() -eq $session.base.name.ToLower())
{
$isConnectedVM = $true
break
}
}
}

if ($isConnectedVM)
{
$item.PoolName = $pool.base.Name
$item.DesktopName = $desktopVM.base.Name
$item.Task = "CONNECTED"
$item.TaskResult = ""

$results += $item
}
else
{
$vm = $vms | Where-Object { $_.Name -eq $desktopVM.base.Name }
# 체크해봐야하는 구문
if (-not ($vm))
{
$vm = Get-VM -Name $desktopVM.base.Name
}

if ($vm.PowerState -eq "PoweredOff")
{

$item.PoolName = $pool.base.name
$item.DesktopName = $desktopVM.base.name
$item.Task = "ALREADY POWEROFF"
$item.TaskResult = ""

$results += $item
}
else
{
if ($vm.ExtensionData.Guest.ToolsRunningStatus -eq "guestToolsRunning")
{
Shutdown-VMGuest -VM $vm -Confirm:$false -ErrorAction SilentlyContinue | Out-Null

$item.PoolName = $pool.base.name
$item.DesktopName = $desktopVM.base.name
$item.Task = "SHUTDOWN"
$item.TaskResult = ""

$results += $item
$count++
$isNeedToWait = $true
}
else
{
Stop-VM -VM $vm -RunAsync -Confirm:$false -ErrorAction SilentlyContinue | Out-Null

$item.PoolName = $pool.base.name
$item.DesktopName = $desktopVM.base.name
$item.Task = "POWEROFF"
$item.TaskResult = ""

$results += $item
$count++
$isNeedToWait = $true
}
}
}
}
}

if ($isNeedToWait)
{
# 이후 전원 정책 정상적용 여부를 확인하기 위해 300초 대기함
WriteLog "Wait for 300 seconds"

Write-Host " " -NoNewLine

for($i=0; $i -lt 30; $i++)
{
if (($i % 30) -eq 0) { Write-Host "." -NoNewLine -ForegroundColor Gray }
Start-Sleep -Seconds 1
}

Write-Host ""
# Wait 300 Second
}


Foreach ($item In ($results | Where-Object { $_.PoolName -eq $pool.base.Name }))
{
if (($item.Task -eq "SHUTDOWN") -or ($item.Task -eq "POWEROFF"))
{
$vm = Get-VM -Name $item.DesktopName

if ($vm.PowerState -eq "PoweredOff")
{
($results | Where-Object { $_.DesktopName -eq $item.DesktopName -and $_.PoolName -eq $pool.base.Name }).TaskResult = "SUCCESS"
}
else
{
Stop-VM -VM $vm -RunAsync -Confirm:$false -ErrorAction SilentlyContinue | Out-Null
($results | Where-Object { $_.DesktopName -eq $item.DesktopName -and $_.PoolName-eq $pool.base.Name }).TaskResult = "FAILED -> POWEROFF"
}
}
}

# Shutdown VM
WriteLog "Shutdown End"
}

$item = "" | Select-Object PoolName, DesktopName, Task, TaskResult
$item.PoolName = $startTime
$results += $item

$item = "" | Select-Object PoolName, DesktopName, Task, TaskResult
$item.PoolName = (Get-Date)
$results += $item

$date = Get-Date -Format "yy-mm-dd"
WriteLog ("Export Result " + $reportFile)
$results | Export-Csv -Path "$reportFile" -Encoding Unicode
}

function ChangeDesktopPoolPowerPolicyRemainOn()
{
$pools = Get-Pool -PoolType $PoolType | Sort-Object -Property pool_id

Foreach ($pool In $pools){

Write-Host
" [{0}] {1}" -f $pool.pool_id, $pool.DisplayName

Write-Host "Change PowerPolicy To Remain On"
Update-ManualPool -Pool_id $pool.pool_id -PowerPolicy RemainOn
}
}

function ChangeDesktopPoolPowerPolicyAlwaysOn()
{
$pools = Get-Pool -PoolType $PoolType | Sort-Object -Property pool_id

Foreach ($pool In $pools){

Write-Host
" [{0}] {1}" -f $pool.pool_id, $pool.DisplayName

Write-Host "Change PowerPolicy To Always On"
Update-ManualPool -Pool_id $pool.pool_id -PowerPolicy AlwaysOn
}
}

Clear-Host

$starttime = Get-Date

$reportFolder = "C:"+"\Reports"
if (-not (Test-Path -Path $reportFolder)) { New-Item -ItemType directory -Path $reportFolder | Out-Null }

Write-Host "Step-1 Prepare Exection Environment" -ForegroundColor Yellow

WriteLog "Loading VMware.View.Broker"
If (-not (Get-PSSnapIn -Name VMware.View.Broker -ErrorAction SilentlyContinue))
{
Add-PSSnapin -Name VMware.View.Broker -ErrorAction SilentlyContinue | Out-Null
}

WriteLog "Loading VMware.VimAutomation.Core"
If (-not (Get-PSSnapIn -Name VMware.VimAutomation.Core -ErrorAction SilentlyContinue))
{
Add-PSSnapin VMware.VimAutomation.Core -ErrorAction SilentlyContinue | Out-Null
}

Write-Host ""

$vcuser = "administrator@vsphere.local"
$vcpass = "SolVMware1!"

Write-Host "Step-2 Connect To vCenter Server [$vcaddress]" -ForegroundColor Yellow

if (Connect-VIServer -Server $vcaddress -User $vcuser -Password $vcpass -ErrorAction SilentlyContinue )
{
WriteLog "success"
}
else
{
WriteLog "failed to vCenter Server[$vcaddress]"
return
}

Write-Host ""

Write-Host "Step-3 Connect To Horizon Server [$caddress]" -ForegroundColor Yellow

if (Connect-hvServer -Server $caddress -User $cuser -Password $cpass -ErrorAction SilentlyContinue )
{
WriteLog "success"
}
else
{
WriteLog "failed to Horizon Server[$caddress]"
return
}

Write-Host ""

switch ($TaskType)
{
# Change DesktopPool Power Policy - RemainOn
"1"
{
Write-Host "Step-4 Shutdown Desktop Pool" -ForegroundColor Yellow
ShutdownDesktopPool ($reportFolder + "\ShutdownPool-Report_" + (Get-Date -Format "yyyy-MM-dd_HH-MM-ss") + ".csv")
}

"2"
{
Write-Host "Change DesktopPool Power Policy - Remain On" -ForegroundColor Yellow

$getpools = Get-HVPool -PoolType MANUAL

foreach($getpool in $getpools){

    Set-HVPool -PoolName $getpool.base.name -Key 'desktopSettings.logoffSettings.powerPolicy' -Value 'TAKE_NO_POWER_ACTION'
    
}
}

"3"
{
Write-Host "Change DesktopPool Power Policy - Always On" -ForegroundColor Yellow

$getpools = Get-HVPool -PoolType MANUAL

foreach( $getpool in $getpools){

    Set-HVPool -PoolName $getpool.base.name -Key 'desktopSettings.logoffSettings.powerPolicy' -Value 'ALWAYS_POWERED_ON'
    
}

}
}

Write-Host ""

Write-Host "Step-Final Disconnect vCenter Server [$vcaddress]" -ForegroundColor Yellow

Disconnect-VIServer -Server * -Confirm:$false | Out-Null
WriteLog "success"
Write-Host ""
Write-Host ""
Write-Host ""

Write-Host "..and Disconnect Horizon Server [$caddress]" -ForegroundColor Yellow

Disconnect-HVServer -Server * -Confirm:$false | Out-Null
WriteLog "success"
Write-Host ""
Write-Host ""
Write-Host ""

$endtime = Get-Date
$elapsedtime = ($endtime - $starttime)

"시작 시간: {0}" -f $starttime
"완료 시간: {0}" -f $endtime
"소요 시간: {0:N2} 초" -f $elapsedtime.TotalSeconds
Write-Host
