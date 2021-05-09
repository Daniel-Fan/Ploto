# Ploto
A basic Windows PowerShell based Chia Plotting Manager. 
Cause I was tired of spawning them myself.
一个基本的基于Windows PowerShell的Chia绘图管理器。

Consists of a PowerShell Module that allows to spawn, manage and move plots.

由允许生成，管理和移动图的PowerShell模块组成。
### PlotoSpawn
* [Get-PlotoOutDrives](https://github.com/tydeno/Ploto/blob/main/README.md#get-plotooutdrives)
* [Get-PlotoTempDrives](https://github.com/tydeno/Ploto/blob/main/README.md#get-plototempdrives)
* [Invoke-PlotoJob](https://github.com/tydeno/Ploto/blob/main/README.md#invoke-plotojob)
* [Start-PlotoSpawn](https://github.com/tydeno/Ploto/blob/main/README.md#start-plotospawns)

### PlotoManage
* [Get-PlotoJobs](https://github.com/tydeno/Ploto/blob/main/README.md#get-plotojobs)
* [Stop-PlotoJob](https://github.com/tydeno/Ploto/blob/main/README.md#stop-plotojob)
* [Remove-AbortedJobs](https://github.com/tydeno/Ploto/blob/main/README.md#remove-abortedjobs)

### PlotoMove
* [Get-PlotoPlots](https://github.com/tydeno/Ploto/blob/main/README.md#get-plotoplots)
* [Move-PlotoPlots](https://github.com/tydeno/Ploto/blob/main/README.md#move-plotoplots)
* [Start-PlotoMove](https://github.com/tydeno/Ploto/blob/main/README.md#start-plotomove)

### PlotoFarmer
* [Get-PlotoFarmLog](https://github.com/tydeno/Ploto/blob/main/README.md#check-farm-logs)

# PlotoSpawn
# 生成绘制

只要需要，只要有OutDrive空间，它就会在每个TempDrive上绘制1x绘图（如果您有6x TempDrives = 6x并行绘图作业）。

Ploto会定期检查是否可以使用TempDrive和OutDrive进行绘图。
如果没有可用的TempDrive或OutDrive，Ploto将在300秒内再次检查。

如果有可用空间，Ploto会确定最佳的OutDrive（最大可用空间）并调用chia.exe来开始绘图。
Ploto遍历所有可用的TempDrives一次，并为每个TempDrive生成一个图（只要有足够的OutDrive空间）。
之后，Ploto将检查Spawned的数量是否等于输入的定义。如果没有，Ploto会一直坚持下去。

您可以指定几个重要的参数来控制何时以及在何处生成，绘制和存储图。
PlotoSpawner通过您指定的公分母来标识您的驱动器，以进行模板化和存储绘图。
（公分母例如：新加卷）可以把最终储存驱动器的名称更换为`Chiaout1`,`Chiaout2`, 把缓存驱动器的名称更换为`Chiaplot1`,`Chiaplot2`

重要信息：Ploto假定您在驱动器的根目录中进行绘图，并且这些驱动器专用于绘图。因此，请确保您也这样做。
当驱动器包含其他数据时，它可能会起作用，但是Ploto是为空的，仅用于绘图的驱动器而设计的。
编辑：我注意到我在Q：\驱动器中有一些数据的文件夹。这样看来行得通。

For reference heres my setup:
* CPU: i9-9900k
* RAM: 32 GB DDR4
* TempDrives:

| Name          | DriveLetter | Type   | Size      |
|---------------|----------|--------|--------------|
|ChiaPlot 1 | I:\ | SATA SSD | 465 GB
|ChiaPlot 2 | H:\ | SATA SSD | 465 GB
|ChiaPlot 3 | E:\ | SATA SSD | 465 GB
|ChiaPlot 4 | Q:\ | SATA SSD | 1810 GB
|ChiaPlot 5 | J:\ | NVME SSD PCI 16x | 465 GB

* OutDrives:

| Name          | DriveLetter | Type   | Size      |
|---------------|----------|--------|--------------|
|ChiaOut 1 | K:\ | SATA HDD | 465 GB
|ChiaOut 2 | D:\ | SATA HDD | 465 GB

因此，对于我的TempDrive，我的分母就是它的“绘图”，对于目的地，我的分母是它的“输出”。
如果我想将例如2x驱动器的jost用作TempDrives，则将其重命名并调整分母。例如“ plotThis”

默认情况下，Ploto在每个磁盘上并行仅生成1x绘图作业。因此，当我以默认生成量启动Ploto时：
```powershell
Start-PlotoSpawns -BufferSize 3390 -Thread 2 -InputAmountToSpawn 36 -OutDriveDenom "out" -TempDriveDenom "plot" -WaitTimeBetweenPlotOnSeparateDisks 15 EnableBitfield $true -MaxParallelJobsOnAllDisks 5
```

将会发生以下情况：
如果临时驱动器和输出驱动器上有足够的可用空间，则Ploto将在每个磁盘上产生1x的作业，并在作业之间指定等待时间。对于每个作业，它都会知道该磁盘上正在进行的打印作业，从而重新计算最合适的输出驱动器。

使用参数“ -MaxParallelJobsOnAllDisks”，可以定义总体应并行多少个图作业。因此，这将是您的硬性规定。如果有多达您定义为最大数量的作业，则PlotoSpawner不会产生更多的作业。这样可以防止您的系统过量使用。

### I need more parallelization
### 我需要更多并行
使用“ -MaxParallelJobsOnSameDisks”，您可以定义单个磁盘上应并行存在多少个PlotsJob。此参数影响可以承载1个以上Plot的所有磁盘。 Ploto会检查每个磁盘上的可用空间，并确定它可以作为tempDrive保留的打印数量。也知道正在进行的工作。在达到-MaxParallelJobsOnAllDisks或-MaxParallelJobsOnSameDisk的硬上限之前，它将通过磁盘产生尽可能多的作业。

If I launch PlotoSpawner with these params like this:
```powershell
Start-PlotoSpawns -BufferSize 3390 -Thread 2 -InputAmountToSpawn 36 -OutDriveDenom "out" -TempDriveDenom "plot" -WaitTimeBetweenPlotOnSeparateDisks 15 -WaitTimeBetweenPlotOnSameDisk 60 -MaxParallelJobsOnAllDisks 7 -MaxParallelJobsOnSameDisk 3 -EnableBitfield $true
```

PlotoSpawner will at max spawn 7 parallel jobs, and max 3 Jobs in parallel on the same disk. This means for my temp drive setup the following:
| Name          | DriveLetter | Type   | Size      | Total Plots in parallel |
|---------------|----------|--------|--------------|-------------------------|
|ChiaPlot 1 | I:\ | SATA SSD | 465 GB | 1
|ChiaPlot 2 | H:\ | SATA SSD | 465 GB | 1
|ChiaPlot 3 | E:\ | SATA SSD | 465 GB | 1
|ChiaPlot 4 | Q:\ | SATA SSD | 1810 GB | 3
|ChiaPlot 5 | J:\ | NVME SSD PCI 16x | 465 GB | 1

因此，在每个磁盘和同一磁盘上的作业之间，将有7x个并行执行的绘图作业与定义的等待时间（以分钟为单位）并行运行。
驱动器Q：\不会看到-MaxParallelJobsOnSameDisk 3定义的并行3个以上的图。

当作业完成并且临时驱动器再次可用时，PlotoSpawner将产生下一个作业，直到产生了您指定为-InputAmountToSpawn的数量或达到最大上限为止。

PlotoSpawner redirects the output of chia.exe to to the following path: 
* C:\Users\me\.chia\mainnet\plotter\

And creates two log files for each job with the following notation:
* PlotoSpawnerLog_30_4_0_49_49ab3c48-532b-4f17-855d-3c5b4981528b_Tmp-E_Out-K.txt (chia.exe output)
* PlotoSpawnerLog_30_4_0_49_49ab3c48-532b-4f17-855d-3c5b4981528b_Tmp-E_Out-K'@'Stat.txt (Additional Info from PLotoSpawner)
如果你之前使用官方工具进行绘制，请全部删除`C:\Users\me\.chia\mainnet\plotter\`中的`log`文件


# Prereqs
# 准备条件
The following prereqs need to be met in order for Ploto to function properly:
* chia.exe is installed (version is determined automatically) 
* You may need to change PowerShell Execution Policy to allow the script to be imported.
您可能需要更改PowerShell执行策略以允许导入脚本。

You can do it by using Set-ExecutionPolicy like this:
```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```
根据我自己的实验，需要使用`unrestricted`模式
```powershell
Set-ExecutionPolicy unrestricted -Scope CurrentUser
```

# How to
If you want to use Ploto follow along.

## Spawn Plots（生成绘制）
1. Make sure your Out and TempDrives are named accordingly（确保缓存驱动器和最终储存驱动器准备完成，查看下方链接，是否正确修改了名称，可以被程序识别
* [Get-PlotoOutDrives](https://github.com/tydeno/Ploto/blob/main/README.md#get-plotooutdrives)）
2. Download Ploto code
3. Import-Module "Ploto" 
```powershell
Import-Module "C:\Users\Me\Downloads\Ploto\Ploto.psm1"
```
4. Launch PlotoSpawner
```powershell
Start-PlotoSpawns -BufferSize 3390 -Thread 2 -InputAmountToSpawn 36 -OutDriveDenom "out" -TempDriveDenom "plot" -WaitTimeBetweenPlotOnSeparateDisks 15 -WaitTimeBetweenPlotOnSameDisk 60 -MaxParallelJobsOnAllDisks 7 -MaxParallelJobsOnSameDisk 3 -EnableBitfield $true
```
```
PlotoSpawner @ 4/30/2021 3:19:13 AM : Spawned the following plot Job:
JobId :           ad917660-9de9-4810-8977-6ace317d7ddb
ProcessID         : 13192
OutDrive          : K:
TempDrive         : Q:
ArgumentsList     : plots create -k 32 -t Q:\ -d K:\
ChiaVersionUsed   : 1.1.2
LogPath           : C:\Users\me\.chia\mainnet\plotter\PlotoSpawnerLog_30_4_3_19_ad917660-9de9-4810-8977-6ace317d7ddb_Tmp-Q_Out-K.txt
StartTime         : 4/30/2021 3:19:13 AM

PlotoManager @ 4/30/2021 3:49:13 AM : Amount of spawned Plots in this iteration: 6
PlotoManager @ 4/30/2021 3:49:13 AM : Overall spawned Plots since start of script: 6
```


5. Leave the PowerShell Session open (can be minimized)


## Get Jobs
1. Open another PowerShell session 
2. Import-Module "Ploto" 
```powershell
Import-Module "C:\Users\Me\Downloads\Ploto\Ploto.psm1"
```
4. Launch Get-PlotoJobs and format Output
```powershell
Get-PlotoJobs | ft
```

```
JobId                                PlotId                                                           PID   Status       TempDrive OutDrive LogPath
-----------------                    ------                                                           ---   ------------ --------- -------- -------
49ab3c48-532b-4f17-855d-3c5b4981528b xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 11856 3.6          E:        K:       C:\Users\me\.chia...
8a0cc01e-37e7-4507-ad6e-cad9401c1381 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 9184  3.6          F:        K:       C:\Users\me\.chia...
95c7cd61-bd88-45a3-a6a2-c243338de480 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 1604  3.5          H:        D:       C:\Users\me\.chia...
465355ef-7da6-4691-8137-3eeba98976d5 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 16280 3.4          I:        K:       C:\Users\me\.chia...
2120b771-2376-49f5-8d47-99a411865ec9 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 14228 3.3          J:        D:       C:\Users\ne\.chia...
ad917660-9de9-4810-8977-6ace317d7ddb xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 13192 2.2          Q:        K:       C:\Users\me\.chia...
2b8596cd-3369-4e8c-a04f-26c85acdfd82 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 9752  2.1          Q:        K:       C:\Users\me\.chia...
cfff29b8-fdee-4988-ae89-9db035d809bc xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 11176 1.6          Q:        K:       C:\Users\me\.chia...
```

Check Jobs With PerformanceCounters:
```powershell
Get-PlotoJobs -PerfCounter 
```

```
JobId                 : 10e6deb5-6a13-4a0d-9c77-8c65d717bf6b
PlotId                : 332f93247b707d3bcf977889edff9bcbc9f0c3d3e30bfd941328bd7bf424f03a
PID                   : 6648
Status                : 3.6
TempDrive             : Q:
OutDrive              : D:
LogPath               : C:\Users\me\.chia\mainnet\plotter\PlotoSpawnerLog_30_4_19_12_10e6deb5-6a13-4a0d-9c77-8c65d71
                        7bf6b_Tmp-Q_Out-D.txt
StatLogPath           : C:\Users\me\.chia\mainnet\plotter\PlotoSpawnerLog_30_4_19_12_10e6deb5-6a13-4a0d-9c77-8c65d71
                        7bf6b_Tmp-Q_Out-D@Stat.txt
PlotSizeOnDisk        : 48.03 GB
cpuUsagePercent       : 0.49
memUsageMB            : 2676
CompletionTimeInHours : Still in progress
```

To get a better Overview, select the Proprties you want to see and use Format-Table:
```powershell
Get-PlotoJobs -PerfCounter | ? {$_.Status -ne "Completed"} | select PID, Status, TempDrive, OutDrive, cpuUsage, memUsageMB, PlotSizeOnDisk | ft
```

```
PID  Status       TempDrive OutDrive cpuUsagePercent memUsageMB PlotSizeOnDisk
---  ------------ --------- -------- --------------- ---------- --------------
8144 3.5          Q:        K:                  0.58       2130 89.46 GB
6648 3.6          Q:        D:                  6.24       2676 48.03 GB
5444 3.6          Q:        D:                  3.29       2676 48.03 GB
```


## Stop Jobs
1. Open a PowerShell session and import Module "Ploto" or use an existing one.
2. Get PlotJobSpawnerId of Job you want to stop by calling "Get-PlotJobs"
3. Stop the process:

```powershell
Stop-PlotoJob -JobId cfff29b8-fdee-4988-ae89-9db035d809bc
```
or if you want to Stop all Jobs that are aborted:
```powershell
Remove-AbortedPlotoJobs
```

## Move Plots
As you may have noticed in my ref setup: I have little OutDrive storage capacity (1TTB roughly).
This is only possible as I continously move the final Plots to my farming machine with lots of big drives. 

I do this by moving plots to a external drive and plug that into my farmer, and sometimes I also transfer plots across my network (not the fatest, thats why I kind of have to do  the running around approach)

PlotoMover helps to automate this process.

If you want to move your plots to a local/external drive just once:
1. Launch a PowerShell session and Import Ploto Module
2. Launch Move-PlotoPLots

```powershell
Move-PlotoPlots -DestinationDrive "P:" -OutDriveDenom "out" -TransferMethod Move-Item
```

If you want to move your plots to a UNC path just once:
1. Launch a PowerShell session and Import Ploto Module
2. Launch Move-PlotoPLots

```powershell
Move-PlotoPlots -DestinationDrive "\\Desktop-xxxxx\d" -OutDriveDenom "out" -TransferMethod Move-Item
```

Please be aware that if you use UNC paths as Destination, PlotoMover cannot grab the free space there and just fires off.

## But I want it do it continously!
Sure, just use Start-PlotoMove with your needed params:

```powershell
Move-PlotoPlots -DestinationDrive "\\Desktop-xxxxx\d" -OutDriveDenom "out" -TransferMethod Move-Item
```

Please be aware that if you use UNC paths as Destination, PlotoMover cannot grab the free space there and just fires off.

## Check Farm Logs
If you want to peek at your farm logs you can use Check-PLotoFarmLogs:
1. Launch a PowerShell session and Import Ploto Module
2. Launch Check-PlotoFarmLogs with your desired LogLevel. It accepts EligiblePlots, Error and Warning.
```powershell
Get-PlotoFarmLog -LogLevel error
```

```
2021-05-03T23:34:45.532 full_node full_node_server        : ERROR    Exception:  <class 'concurrent.futures._base.CancelledError'>, closing connection {'host': '127.0.0.1', 'port': 8449}. Traceback (most recent call last):
concurrent.futures._base.CancelledError
2021-05-03T23:34:48.454 full_node full_node_server        : ERROR    Exception:  <class 'concurrent.futures._base.CancelledError'>, closing connection {'host': '127.0.0.1', 'port': 8449}. Traceback (most recent call last):
concurrent.futures._base.CancelledError
2021-05-03T23:34:52.548 full_node full_node_server        : ERROR    Exception:  <class 'concurrent.futures._base.CancelledError'>, closing connection {'host': '127.0.0.1', 'port': 8449}. Traceback (most recent call last):
concurrent.futures._base.CancelledError
2021-05-03T23:35:03.845 full_node full_node_server        : ERROR    Exception:  <class 'concurrent.futures._base.CancelledError'>, closing connection {'host': '127.0.0.1', 'port': 8449}. Traceback (most recent call last):
concurrent.futures._base.CancelledError
2021-05-03T23:35:08.720 full_node full_node_server        : ERROR    Exception:  <class 'concurrent.futures._base.CancelledError'>, closing connection {'host': '127.0.0.1', 'port': 8449}. Traceback (most recent call last):
concurrent.futures._base.CancelledError
2021-05-03T23:35:15.360 full_node full_node_server        : ERROR    Exception:  <class 'concurrent.futures._base.CancelledError'>, closing connection {'host': '127.0.0.1', 'port': 8449}. Traceback (most recent call last):
concurrent.futures._base.CancelledError
```

# FAQ
> Can I shut down the script when I dont want Ploto to spawn more Plots?

Yep. The individual Chia Plot Jobs wont be affected by that.


# Knowns Bugs and Limitations
Please be aware that Ploto was built for my specific setup. I try to generalize it as much as possible, but thats not easy-breezy.
So what works for me, may not ultimately work for you. 
Please also bear in mind that unexpoected beahviour and or bugs are possible.

These are known:
* Only works when plotting in root of drives
* Only works when Drives are dedicated to plotting (dont hold any other files)
* Can only display and stop PlotJobs that have been spawned using PlotoSpawner
* Using the -PerfCounter param on Get-PlotoJobs takes a while to load
* PlotoMover is very limited right now, may break copy jobs at times (Bits)
* PlotoMover does not check for available free space on network drives as its unaware of it (only does for local drives)
* If you have more than 1x version of chia within your C:\Users\Me\AppData\Local\chia-blockchain folder, Ploto wont be able to determine the version and will fail.
  Make sure theres only one available folder with chia.exe (eg. app-1.1.3)




## Get-PlotoOutDrives
Gets all Windows Volumes that match the -OutDriveDenom parameter and checks if free space is greater than 107 GB (amount currently used by final chia plots).
It wraps all the needed information of the volume like DriveLetter, ChiaDriveType, VolumeName, a bool IsPlootable, and the calculated amount of plots to hold into a object and returns the collection of objects as the result of that function.

#### Example:

```powershell
Get-PlotoOutDrives -OutDriveDenom "out"
```

#### Output:

```
DriveLetter         : D:
ChiaDriveType       : Out
VolumeName          : ChiaOut2
FreeSpace           : 363.12
IsPlottable         : True
AmountOfPlotsToHold : 3

DriveLetter         : K:
ChiaDriveType       : Out
VolumeName          : ChiaOut3
FreeSpace           : 364.24
IsPlottable         : True
AmountOfPlotsToHold : 3
```

#### Parameters:
| Name          | Required | Type   | Description                                                                                                                              |
|---------------|----------|--------|------------------------------------------------------------------------------------------------------------------------------------------|
|OutDriveDenom  | Yes      | String | A common denominator for all your drives used as out drives. All drives with that denom in name will be used to store done plots.



## Get-PlotoTempDrives
Gets all Windows Volumes that match the -TempDriveDenom parameter and checks if free space is greater than 270 GB (amount currently used by chia plots as temp storage).
It wraps all the needed information of the volume like DriveLetter, ChiaDriveType, VolumeName, a bool IsPlootable, and the calculated amount of plots to temp, whether it has a plot in porgress (determined by checking if the drive contains any file) into a object and returns the collection of objects as the result of that function.

#### Example:

```powershell
 Get-PlotoTempDrives -TempDriveDenom "plot"
```
#### Output:

```
DriveLetter             : E:
ChiaDriveType           : Temp
VolumeName              : ChiaPlot 3 Evo 860 512GB
FreeSpace               : 437.28
TotalSpace              : 465.76
hasFolder               : False
IsPlottable             : True
HasPlotInProgress       : False
AmountOfPlotsInProgress : 0
AmountOfPlotsToTempMax  : 1
AvailableAmountToPlot   : 1
PlotInProgressID        :

DriveLetter             : F:
ChiaDriveType           : Temp
VolumeName              : ChiaPlot4 NVME FullDisk 1
FreeSpace               : 446.76
TotalSpace              : 465.75
hasFolder               : False
IsPlottable             : True
HasPlotInProgress       : False
AmountOfPlotsInProgress : 0
AmountOfPlotsToTempMax  : 1
AvailableAmountToPlot   : 1
PlotInProgressID        :


## Invoke-PlotoJob
Calls Get-PlotoTempDrives to get all Temp drives that are plottable. For each tempDrive it determines the most appropriate OutDrive (using Get-PlotoOutDrives function), stitches together the ArgumentList for chia and fires off the chia plot job using chia.exe. For each created PlotJob the function creates an Object and appends it to a collection of objects, which are returned upon the function call. 

#### Example:

```powershell
Invoke-PlotoJob -OutDriveDenom "out" -TempDriveDenom "plot" -WaitTimeBetweenPlotOnSeparateDisks 0.1 -WaitTimeBetweenPlotOnSameDisk 0.1 -MaxParallelJobsOnAllDisks 2 -MaxParallelJobsOnSameDisk 1 -EnableBitfield $false -Verbose
```
#### Output:

```
PlotoSpawnerJobId : 49ab3c48-532b-4f17-855d-3c5b4981528b
ProcessID       : 9024
OutDrive        : D:
TempDrive       : H:
ArgumentsList   : plots create -k 32 -t H:\ -d D:\
ChiaVersionUsed : 1.1.2
LogPath           : C:\Users\me\.chia\mainnet\plotter\PlotoSpawnerLog_30_4_0_49_49ab3c48-532b-4f17-855d-3c5b4981528b_Tmp-E_Out-K.txt
StartTime       : 4/29/2021 1:55:50 PM
```

## Start-PlotoSpawns
Main function that nests all else.
Continously calls Invoke-PlotoJob and states progress and other information. It runs until it created the amount of specified Plot by using the -InputAmountToSpawn param.

#### Example:

```powershell
Start-PlotoSpawns -InputAmountToSpawn 36 -OutDriveDenom "out" -TempDriveDenom "plot" -WaitTimeBetweenPlotOnSeparateDisks 0.1 -WaitTimeBetweenPlotOnSameDisk 0.1 -MaxParallelJobsOnAllDisks 2 -MaxParallelJobsOnSameDisk 1 -EnableBitfield $false 
```

#### Output:

```
PlotoManager @ 4/29/2021 1:45:38 PM : Amount of spawned Plots in this iteration: 2
PlotoManager @ 4/29/2021 1:45:38 PM : Overall spawned Plots since start of script: 2
```


# PlotoManage
Allows you to check status of your current plot jobs aswell as stopping them and cleaning the temp drives.

## Get-PlotoJobs
Analyzes the plotter logs (standard chia.exe output redirected, enriched with additional data) and shows the status, pid and drives in use. The function only pick up data of plot logs that have been spawned using PlotoSpawner (as it deploys initial data like PID of process and drives use). The additional logs are stored in the same location as the standrad logs. If you delete those, this function wont be able to read certain properties anymore. 

## Status Codes and their meaning

See below for a definition of what phase coe is associated with which chia.exe log output.
```powershell
"Starting plotting progress into temporary dirs:*" {$StatusReturn = "Initializing"}
"Starting phase 1/4*" {$StatusReturn = "1.0"}
"Computing table 1" {$StatusReturn = "1.1"}
"F1 complete, time*" {$StatusReturn = "1.1"}
"Computing table 2" {$StatusReturn = "1.1"}
"Computing table 3" {$StatusReturn = "1.2"}
"Computing table 4" {$StatusReturn = "1.3"}
"Computing table 5" {$StatusReturn = "1.4"}
"Computing table 6" {$StatusReturn = "1.5"}
"Computing table 7" {$StatusReturn = "1.6"}
"Starting phase 2/4*" {$StatusReturn = "2.0"}
"Backpropagating on table 7" {$StatusReturn = "2.1"}
"Backpropagating on table 6" {$StatusReturn = "2.2"}
"Backpropagating on table 5" {$StatusReturn = "2.3"}
"Backpropagating on table 4" {$StatusReturn = "2.4"}
"Backpropagating on table 3" {$StatusReturn = "2.5"}
"Backpropagating on table 2" {$StatusReturn = "2.6"}
"Starting phase 3/4*" {$StatusReturn = "3.0"}
"Compressing tables 1 and 2" {$StatusReturn = "3.1"}
"Compressing tables 2 and 3" {$StatusReturn = "3.2"}
"Compressing tables 3 and 4" {$StatusReturn = "3.3"}
"Compressing tables 4 and 5" {$StatusReturn = "3.4"}
"Compressing tables 5 and 6" {$StatusReturn = "3.5"}
"Compressing tables 6 and 7" {$StatusReturn = "3.6"}
"Starting phase 4/4*" {$StatusReturn = "4.0"}
"Writing C2 table*" {$StatusReturn = "4.1"}
"Time for phase 4*" {$StatusReturn = "4.2"}
"Renamed final file*" {$StatusReturn = "4.3"}
```


#### Example:
```powershell
Get-PlotoJobs | ft 
```

#### Output:
```
JobId                                PlotId                                                           PID  Status       TempDrive OutDrive LogPath
-----------------                    ------                                                           ---  ------------ --------- -------- -------
2e39e295-ccd9-4abf-94e9-01a854cbfa24 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx None Completed    E:        K:       C:\Users\...
ed2133b0-018c-44db-81e7-61befbda8031 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx None Completed    F:        D:       C:\Users\...
bc3b44b1-b290-4487-a552-c4dda2e11366 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx None Completed    H:        K:       C:\Users\...
10e6deb5-6a13-4a0d-9c77-8c65d717bf6b xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx None Completed    Q:        D:       C:\Users\...
f865e425-ada4-44f0-8537-23a033aef302 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx None Completed    Q:        D:       C:\Users\...
b19eaef4-f9b3-4807-8870-a959e5aa3a21 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx None Completed    I:        D:       C:\Users\...
278615e9-8e4d-4af4-bfcc-4665412aae89 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx None Completed    Q:        K:       C:\Users\...

```


## Remove-AbortedJobs
Gets all Jobs from Get-PlotoJobs that are aborted (where no process runs to PID) and foreach call Stop-PlotoJob

#### Example:

```powershell
Remove-AbortedJobs
```

#### Output:
```
PlotoRemoveAbortedJobs @ 5/1/2021 6:42:24 PM : Found aborted Jobs to be deleted: 6cfb4e4a-cb71-4f2a-9387-17a8049ce625 85b08573-054d-46f0-b7e3-755f9ce021bc cbab519c-2f26-41c9-b3fa-8bcb0ba36d3a 2b0ab204-3b0e-4e8c-b04c-a884859ae637 f639cb35-23a8-4010-a1db-ab6186bd117c
PlotoRemoveAbortedJobs @ 5/1/2021 6:42:24 PM : Cleaning up...
PlotoStopJob @ 5/1/2021 6:42:24 PM : ERROR:  Cannot bind parameter 'Id'. Cannot convert value "None" to type "System.Int32". Error: "Input string was not in a correct format."
PlotoStopJob @ 5/1/2021 6:42:24 PM : Found .tmp files for this job to be deleted. Continue with deletion.
PlotoStopJob @ 5/1/2021 6:42:24 PM : Removed temp files on F:
PlotoStopJob @ 5/1/2021 6:42:24 PM : Removed log files for this job.
PlotoStopJob @ 5/1/2021 6:42:25 PM : ERROR:  Cannot bind parameter 'Id'. Cannot convert value "None" to type "System.Int32". Error: "Input string was not in a correct format."
PlotoStopJob @ 5/1/2021 6:42:25 PM : Found .tmp files for this job to be deleted. Continue with deletion.
PlotoStopJob @ 5/1/2021 6:42:25 PM : Removed temp files on H:
PlotoStopJob @ 5/1/2021 6:42:25 PM : Removed log files for this job.
PlotoStopJob @ 5/1/2021 6:42:25 PM : ERROR:  Cannot bind parameter 'Id'. Cannot convert value "None" to type "System.Int32". Error: "Input string was not in a correct format."
PlotoStopJob @ 5/1/2021 6:42:25 PM : Found .tmp files for this job to be deleted. Continue with deletion.
PlotoStopJob @ 5/1/2021 6:42:25 PM : Removed temp files on E:
PlotoStopJob @ 5/1/2021 6:42:25 PM : Removed log files for this job.
PlotoStopJob @ 5/1/2021 6:42:26 PM : ERROR:  Cannot bind parameter 'Id'. Cannot convert value "None" to type "System.Int32". Error: "Input string was not in a correct format."
PlotoStopJob @ 5/1/2021 6:42:26 PM : Found .tmp files for this job to be deleted. Continue with deletion.
PlotoStopJob @ 5/1/2021 6:42:26 PM : Removed temp files on J:
PlotoStopJob @ 5/1/2021 6:42:26 PM : Removed log files for this job.
PlotoStopJob @ 5/1/2021 6:42:26 PM : ERROR:  Cannot bind parameter 'Id'. Cannot convert value "None" to type "System.Int32". Error: "Input string was not in a correct format."
PlotoStopJob @ 5/1/2021 6:42:26 PM : Found .tmp files for this job to be deleted. Continue with deletion.
PlotoStopJob @ 5/1/2021 6:42:26 PM : Removed temp files on I:
PlotoStopJob @ 5/1/2021 6:42:26 PM : Removed log files for this job.
PlotoRemoveAbortedJobs @ 5/1/2021 6:42:26 PM : Removed Amount of aborted Jobs: 5
```
The Error below is known and only says that the process is already closed. This is expected. In the future this error may be surpressed.

```
PlotoStopJob @ 5/1/2021 6:42:26 PM : ERROR:  Cannot bind parameter 'Id'. Cannot convert value "None" to type "System.Int32". Error: "Input string was not in a correct format."
```

# PlotoMove
Continously searches for final Plots on your OutDrives and moves them to your desired location. I do this for transferring plots from my plotting machine to my farming machine.

## Get-PlotoPlots
Searches defined Outdrives for Final Plots (file that end upon .plot) and returns an array with final plots.

#### Example:
```powershell
Get-PlotoPlots -OutDriveDenom "out"
```

#### Output:
```
Iterating trough Drive:  @{DriveLetter=D:; ChiaDriveType=Out; VolumeName=ChiaOut2; FreeSpace=59.04; TotalSpace=0; IsPlottable=False; AmountOfPlotsToHold=0}
Checking if any item in that drive contains .PLOT as file ending...
Found a Final plot:  plot-k32-2021-04-30-18-52-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot
Found a Final plot:  plot-k32-2021-04-30-19-02-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot
Found a Final plot:  plot-k32-2021-04-30-19-12-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot
Found a Final plot:  plot-k32-2021-04-30-19-42-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot
Iterating trough Drive:  @{DriveLetter=K:; ChiaDriveType=Out; VolumeName=ChiaOut3; FreeSpace=262.86; TotalSpace=0; IsPlottable=True; AmountOfPlotsToHold=2}
Checking if any item in that drive contains .PLOT as file ending...
Found a Final plot:  plot-k32-2021-04-30-18-57-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot
Found a Final plot:  plot-k32-2021-04-30-19-12-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot
--------------------------------------------------------------------------------------------------

FilePath                                                                                           Name
--------                                                                                           ----
D:\plot-k32-2021-04-30-18-52-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot plot-k32-2021-04-...
D:\plot-k32-2021-04-30-19-02-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot plot-k32-2021-04-...
D:\plot-k32-2021-04-30-19-12-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot plot-k32-2021-04-...
D:\plot-k32-2021-04-30-19-42-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot plot-k32-2021-04-...
K:\plot-k32-2021-04-30-18-57-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot plot-k32-2021-04-...
K:\plot-k32-2021-04-30-19-12-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot plot-k32-2021-04-...
```

## Move-PlotoPlots
Grabs the found plots from Get-PlotoPlots and moves them to either a local/external drive using Move-Item cmdlet or to a UNC path using Background Intelligence TRansfer Service (Bits). You can define the OutDrives to search for Plots, TransferMethod and Destination.

Make sure TrasnferMethod and Destination match. 

#### Example:

```powershell
Move-PlotoPlots -DestinationDrive "\\Desktop-xxxxx\d" -OutDriveDenom "out" -TransferMethod BITS
```

#### Output:
```
PlotoMover @ 5/1/2021 7:08:09 PM : Moving plot:  D:\plot-k32-2021-04-30-18-52-dxxxxxxxxxxxxxxxxxxxxxxxxxxxxx00454fxxxxxxxxxxxxxxxxxxxxxxxxxx64.plot to \\Desktop-xxxxx\d using BITS
```

