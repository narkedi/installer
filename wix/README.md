## Requirements for build environment:

1. Windows Installer XML (WiX) toolset, 3.11 or later, http://wixtoolset.org/releases/
2. Install "Windows SDK for Desktop C++ amd64 Apps" feature from Windows SDK 10 (https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk) for building multi-lingual setups.
3. Digital signature service if the MSI should be signed (optional). If you plan to sign the MSI, you need to install the Windows SDK 10 feature "Windows SDK Signing Tools for Desktops Apps".
4. For reviewing the MSI setup or creating custom MST transforms you can install feature "MSI Tools" from Windows SDK 10 (optional).


## How to upgrade to a new OpenJDK version:

1. Download latest OpenJDK zip to the SourceDir directory.

2. Extract the content and setup the expected file structure:

```batch
call powershell.exe ./CreateSourceFolder.AdoptOpenJDK.ps1
```

3. Export the following environment variables:

  Example:
  ```batch
  SET PRODUCT_MAJOR_VERSION=11
  SET PRODUCT_MINOR_VERSION=0
  SET PRODUCT_MAINTENANCE_VERSION=2
  SET PRODUCT_PATCH_VERSION=8
  SET ARCH=x64|x86|arm64 or all "x64 x86 arm64"
  SET JVM=hotspot|openj9|dragonwell or both JVM=hotspot openj9
  SET PRODUCT_CATEGORY=jre|jdk (only one at a time)
  cmd /c Build.OpenJDK_generic.cmd
  ```

  To customize branding information you can export the following environment variables to oveeride the default values. The default values are listed below:
  ```batch
  set VENDOR=Temurin
  set VENDOR_BRANDING=Eclipse Temurin
  set PRODUCT_HELP_LINK=https://github.com/adoptium/adoptium-support/issues/new/choose
  set PRODUCT_SUPPORT_LINK=https://adoptium.net/support.html
  set PRODUCT_UPDATE_INFO_LINK=https://adoptium.net/releases.html
  set VENDOR_BRANDING_LOGO=$(var.SetupResourcesDir)\logo.ico
  set VENDOR_BRANDING_BANNER=$(var.SetupResourcesDir)\wix-banner.bmp
  set VENDOR_BRANDING_DIALOG=$(var.SetupResourcesDir)\wix-dialog.bmp
  ```

 `Build.OpenJDK_generic.cmd` statically depend on this SDK version (edit if needed ):
  ```batch
  SET WIN_SDK_MAJOR_VERSION=10
  SET WIN_SDK_FULL_VERSION=10.0.17763.0
  ```

4. Run `Build.OpenJDK_generic.cmd` to create the MSI setup in "ReleaseDir":

```batch
call Build.OpenJDK_generic.cmd
```

## Deploy via Active Directory GPO.

Installation optional parameters:

#### `INSTALLLEVEL`
- 1 = (Add to PATH + Associate jar)
- 2 = (Add to PATH + Associate jar) + define JAVA_HOME + JavaSoft reg keys

usage sample: 
```batch
msiexec /i OpenJDK8-jdk_xxx.msi INSTALLLEVEL=1
msiexec /i OpenJDK8-jdk_xxx.msi INSTALLLEVEL=2
```

#### Features available:
- FeatureMain (Required) Install Adoptium files ( To use with property : INSTALLDIR to set directory where to install for unattended install )
- FeatureEnvironment (PATH)
- FeatureJavaHome (JAVA_HOME)
- FeatureJarFileRunWith (associate .jar)
- FeatureOracleJavaSoft (Registry keys HKLM\SOFTWARE\JavaSoft\) (break Oracle java start launch from PATH when Adoptium is uninstalled, reinstall Oracle if needed to restore Oracle registry keys) (Only available for admin users / machine setup ( normal users can't write to HKLM ))
- FeatureIcedTeaWeb (Install IcedTea-Web)
- FeatureJNLPFileRunWith (associate .jnlp with IcedTea-Web javaws.exe)

usage sample:
```batch
msiexec /i OpenJDK8-jdk_xxx.msi ADDLOCAL=FeatureMain,FeatureJavaHome,FeatureJarFileRunWith INSTALLDIR=D:\testAdopt
```

#### Embeded transform for language :

see list for full/partial language translation available here https://github.com/adoptium/installer/blob/master/wix/Lang/LanguageList.config
(Feel free to make pull request to add/complete translation)

Set property `TRANSFORMS` with `:<code>` where `<code>` is the ID available in `LanguageList.config`.

```batch
msiexec /i OpenJDK8-jdk_xxx.msi ADDLOCAL=FeatureMain,FeatureJavaHome,FeatureJarFileRunWith INSTALLDIR=D:\testAdopt TRANSFORMS=:1036
```

##### Language with GPO :
 You must keep only the language you want use in the MSI.
 Use ORCA and remove all except one language : under view --> Summary Information go to the Languages input field and remove all but one. Okay and save.


#### Per user install

Note : 
- FeatureOracleJavaSoft can't and must not be used per user install as it only write to HKLM. (see https://docs.oracle.com/javase/9/install/installation-jdk-and-jre-microsoft-windows-platforms.htm#JSJIG-GUID-47C269A3-5220-412F-9E31-4B8C37A82BFB)
- Machine PATH is always loaded before User PATH (FeatureEnvironment) ( If another java is installed per machine it will be the default one when using the PATH )


See https://docs.microsoft.com/fr-fr/windows/desktop/Msi/allusers

Windows 7 and later : use MSIINSTALLPERUSER=1
```batch
msiexec /i OpenJDK8-jdk_xxx.msii INSTALLDIR=D:\testAdopt ADDLOCAL=FeatureMain,FeatureEnvironment,FeatureJavaHome,FeatureJarFileRunWith MSIINSTALLPERUSER=1
```

#### Reinstall option :
CF https://docs.microsoft.com/en-us/windows/desktop/msi/reinstallmode

usage sample:
```batch
msiexec /i OpenJDK11-jdk_x64_windows_hotspot-11.0.3.9.msi REINSTALL=ALL /quiet
msiexec /i OpenJDK11-jdk_x64_windows_hotspot-11.0.3.9.msi REINSTALL=ALL REINSTALLMODE=amus /quiet
```

## MSI upgrade limitation :
Upgradable MSI work only for first 3 digit from the build number (due to MSI limitation ) :  (https://docs.microsoft.com/fr-fr/windows/desktop/Msi/productversion )

* Upgradable : 8.0.2.1 -> 8.0.3.1 Yes
* Upgradable : 8.0.2.1 -> 8.0.2.2 No ( You must uninstall previous msi and install new one )
* Upgradable : 8.0.2.1 -> 8.1.2.1 Yes
* Upgradable : 8.0.2.1 -> 11.0.2.1 No  ( Adoptium dont provide upgrade for different major version ( jdk 8 -> jdk 11 ) (You can keep both or uninstall older jdk yourself )

## Troubleshooting

Logfiles created by `msiexec.exe` help diagnosing problems with our MSI installers.

If you are installing an Adoptium MSI using the command line, pass `/l*v %temp%\Adoptium-MSI.log` to write a log to `%temp%\Adoptium-MSI.log`. Example command:

```
C:\WINDOWS\System32\msiexec.exe /i "C:\Users\Administrator\Downloads\OpenJDK11U-jdk_x64_windows_hotspot_11.0.6_10.msi" MSIINSTALLPERUSER=1 INSTALLDIR="C:\Users\Administrator\AppData\Local\Programs\Adoptium" ADDLOCAL=FeatureJavaHome,FeatureEnvironment,FeatureJarFileRunWith /passive /l*v %temp%\Adoptium-MSI.log
```

If you have trouble with the GUI installer, changes to the registry are needed to enable logging (copy into `cmd.exe`):

```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer" /v Debug /t REG_DWORD /d 7 /f 
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer" /v Logging /t REG_SZ /d voicewarmupx! /f
```

The logfiles are written to `%temp%\msi*.log` (`*` denotes a randomly generated string consisting of letters and numbers).

To undo the changes to the registry:

```
reg delete "HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer" /v Debug /f
reg delete "HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer" /v Logging /f
```
