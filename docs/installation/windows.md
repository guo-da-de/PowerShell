# Package Installation Instructions

## MSI

To install PowerShell on Windows Full SKU (works on Windows 7 SP1 and later), download either the MSI from [AppVeyor][] for a nightly build,
or a released package from our GitHub [releases][] page.

Once downloaded, double-click the installer and follow the prompts.

There is a shortcut placed in the Start Menu upon installation.

* By default the package is installed to `$env:ProgramFiles\PowerShell\`
* You can launch PowerShell via the Start Menu or `$env:ProgramFiles\PowerShell\pwsh.exe`

### Prerequisites

To enable PowerShell remoting over WinRM, the following prerequisites need to be met:

* Install the [Universal C Runtime](https://www.microsoft.com/download/details.aspx?id=50410) on Windows versions prior to Windows 10.
  It is available via direct download or Windows Update.
  Fully patched (including optional packages), supported systems will already have this installed.
* Install the Windows Management Framework (WMF) [4.0](https://www.microsoft.com/download/details.aspx?id=40855)
  or newer ([5.0](https://www.microsoft.com/download/details.aspx?id=50395),
  [5.1](https://www.microsoft.com/download/details.aspx?id=54616)) on Windows 7.

## ZIP

PowerShell binary ZIP archives are provided to enable advanced deployment scenarios.
Be noted that when using the ZIP archive, you won't get the prerequisites check as in the MSI package.
So in order for remoting over WinRM to work properly on Windows versions prior to Windows 10,
you need to make sure the [prerequisites](#prerequisites) are met.

## Deploying on Windows IoT

Windows IoT already comes with Windows PowerShell which we will use to deploy PowerShell Core 6.

* Create `PSSession` to target device

```powershell
$s = New-PSSession -ComputerName <deviceIp> -Credential Administrator
```

* Copy the zip package to the device

```powershell
# change the destination to however you had partitioned it with sufficient space for the zip and the unzipped contents
# the path should be local to the device
Copy-Item .\PowerShell-6.0.1-win-arm32.zip -Destination u:\users\administrator\Downloads -ToSession $s
```

* Connect to the device and expand the archive

```powershell
Enter-PSSession $s
cd u:\users\administrator\downloads
Expand-Archive .\PowerShell-6.0.1-win-arm32.zip
```

* Setup remoting to PowerShell Core 6

```powershell
cd .\PowerShell-6.0.1-win-arm32
# Be sure to use the -PowerShellHome parameter otherwise it'll try to create a new endpoint with Windows PowerShell 5.1
.\Install-PowerShellRemoting.ps1 -PowerShellHome .
# You'll get an error message and will be disconnected from the device because it has to restart WinRM
```

* Connect to PowerShell Core 6 endpoint on device

```powershell
# Be sure to use the -Configuration parameter.  If you omit it, you will connect to Windows PowerShell 5.1
Enter-PSSession -ComputerName <deviceIp> -Credential Administrator -Configuration powershell.6.0.1
```

## Deploying on Nano Server

These instructions assume that Windows PowerShell is running on the Nano Server image and that it has been generated by the [Nano Server Image Builder](https://technet.microsoft.com/windows-server-docs/get-started/deploy-nano-server).
Nano Server is a "headless" OS and deployment of PowerShell Core binaries can happen in two different ways:

1. Offline - Mount the Nano Server VHD and unzip the contents of the zip file to your chosen location within the mounted image.
1. Online - Transfer the zip file over a PowerShell Session and unzip it in your chosen location.

In both cases, you will need the Windows 10 x64 Zip release package and will need to run the commands within an "Administrator" PowerShell instance.

### Offline Deployment of PowerShell Core

1. Use your favorite zip utility to unzip the package to a directory within the mounted Nano Server image.
1. Unmount the image and boot it.
1. Connect to the inbox instance of Windows PowerShell.
1. Follow the instructions to create a remoting endpoint using the [another instance technique](#executed-by-another-instance-of-powershell-on-behalf-of-the-instance-that-it-will-register).

### Online Deployment of PowerShell Core

The following steps will guide you through the deployment of PowerShell Core to a running instance of Nano Server and the configuration of its remote endpoint.

* Connect to the inbox instance of Windows PowerShell

```powershell
$session = New-PSSession -ComputerName <Nano Server IP address> -Credential <An Administrator account on the system>
```

* Copy the file to the Nano Server instance

```powershell
Copy-Item <local PS Core download location>\powershell-<version>-win-x64.zip c:\ -ToSession $session
```

* Enter the session

```powershell
Enter-PSSession $session
```

* Extract the Zip file

```powershell
# Insert the appropriate version.
Expand-Archive -Path C:\powershell-<version>-win-x64.zip -DestinationPath "C:\PowerShellCore_<version>"
```

* Follow the instructions to create a remoting endpoint using the [another instance technique](#executed-by-another-instance-of-powershell-on-behalf-of-the-instance-that-it-will-register).

## Instructions to Create a Remoting Endpoint

Beginning with 6.0.0-alpha.9, the PowerShell package for Windows includes a WinRM plug-in (pwrshplugin.dll) and an installation script (Install-PowerShellRemoting.ps1).
These files enable PowerShell to accept incoming PowerShell remote connections when its endpoint is specified.

### Motivation

An installation of PowerShell can establish PowerShell sessions to remote computers using `New-PSSession` and `Enter-PSSession`.
To enable it to accept incoming PowerShell remote connections, the user must create a WinRM remoting endpoint.
This is an explicit opt-in scenario where the user runs Install-PowerShellRemoting.ps1 to create the WinRM endpoint.
The installation script is a short-term solution until we add additional functionality to `Enable-PSRemoting` to perform the same action.
For more details, please see issue [#1193](https://github.com/PowerShell/PowerShell/issues/1193).

### Script Actions

The script

1. Creates a directory for the plug-in within %windir%\System32\PowerShell
1. Copies pwrshplugin.dll to that location
1. Generates a configuration file
1. Registers that plug-in with WinRM

### Registration

The script must be executed within an Administrator-level PowerShell session and runs in two modes.

#### Executed by the instance of PowerShell that it will register

``` powershell
Install-PowerShellRemoting.ps1
```

#### Executed by another instance of PowerShell on behalf of the instance that it will register

``` powershell
<path to powershell>\Install-PowerShellRemoting.ps1 -PowerShellHome "<absolute path to the instance's $PSHOME>"
```

For Example:

``` powershell
& 'C:\Program Files\PowerShell\6.0.1\Install-PowerShellRemoting.ps1' -PowerShellHome 'C:\Program Files\PowerShell\6.0.1\'
```

**NOTE:** The remoting registration script will restart WinRM, so all existing PSRP sessions will terminate immediately after the script is run. If run during a remote session, this will terminate the connection.

## How to Connect to the New Endpoint

Create a PowerShell session to the new PowerShell endpoint by specifying `-ConfigurationName "some endpoint name"`. To connect to the PowerShell instance from the example above, use either:

``` powershell
New-PSSession ... -ConfigurationName "powershell.6.0.1"
Enter-PSSession ... -ConfigurationName "powershell.6.0.1"
```

Note that `New-PSSession` and `Enter-PSSession` invocations that do not specify `-ConfigurationName` will target the default PowerShell endpoint, `microsoft.powershell`.

## Artifact Installation Instructions

We publish an archive with CoreCLR bits on every CI build with [AppVeyor][].

[releases]: https://github.com/PowerShell/PowerShell/releases
[signing]: ../../tools/Sign-Package.ps1
[AppVeyor]: https://ci.appveyor.com/project/PowerShell/powershell

## CoreCLR Artifacts

* Download zip package from **artifacts** tab of the particular build.
* Unblock zip file: right-click in File Explorer -> Properties ->
  check 'Unblock' box -> apply
* Extract zip file to `bin` directory
* `./bin/pwsh.exe`
