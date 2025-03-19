# C# CLI Template With MSIX Packaging

## Rationale for the Project

The goals of this project are

1. Determine whether to use [MSIX](https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-packaging-dot-net) for distributing a C# application, specically a CLI.
2. Whether C# is viable for command line applications or if people should use something else like Rust or the Go language.

For _desktop_ apps, it's clear as per the [built-in template and documentation](https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-packaging-dot-net), that MSIX is the modern way to distribute C# desktop apps.

### Results

- Difficulty before this template: Hard. For reference, something harder than this is debugging C++ cross-compilation linking bugs
- Difficult after this template: Moderate
- Was able to build an installable MSIX
- Was able to run the MSIX from the terminal using an alias (App name is "Your App Name", project name is jibberish, and I could use it from the terminal via `nvm2` as intended)
- Works with Nuget Dependencies. I used Spectre.Console
- Had trouble compiling for Neutral / AnyCPU, so stuck with x64
    - I'm sure AMD64 would work as well
    - Ahead-of-Time compilation doesn't work when x86 is included as a target anyways
- Binary Size
    - AOT off and with Spectre.Console
        - MSIX: 34,097,592 bytes
        - Reported installed size: 71.6MB
    - AOT on and with Spectre.Console (Amazing)
        - MSIX size of with Spectre.Console: 1,074,175 bytes
        - Reported installed size: 2.29MB
    - AOT off and Spectre.Console uninstalled
        - MSIX size: 33,688,658 bytes
        - Reported installed size: 70.6MB
        - Spectre.Console has a footprint of 1MB
    - AOT on and Spectre.Console uninstalled
        - MSIX size: 623,758 bytes
        - Reported installed size: 1.26MB
    - AOT on and Spectre.Console usage removed (not uninstalled)
        - MSIX size: 623,765 bytes
        - Reported installed size: 1.26MB
   
### Takeaway

AOT is amazing and exactly what C# needed for deeloeprs to venture into tooling instead of using it just for Windows Desktop apps, ASP.NET (love it), and Blazor.

### Future Plans

- Linux packaging ([AppImage](https://kuiper.zone/publish-appimage-dotnet/),[Flatpak](https://gist.github.com/ssokolow/db565fd8a82d6002baada946adb81f68) GUI installer?)
- I have my doubts about using C# for desktop, but it's definitely the king of the Windows Installer experience
    - But self publishing!
- MacOS packaging ... just kidding

### Why MSIX?

Desktop-oriented.
It targets Windows 10+ which is the minimum Windows version that is still supported. We know it works for non-administraters. "Resource optimized".

Some issues with MSIX are that you can't depend on the install location. This is fine.

### Concerns

One of my conerns is code-signing. In the future, the template should demonstrate how to code-sign + create an installer.

### Story Time

I was recently thinking about how I would go about making a cross-platform or Windows CLI and I landed on either using Rust or C#.
The downside of using Rust is that it takes a long time to compile (lower productivity) and it's also not as much of a challenge compared to C#, which on the positive, works with Visual Studio, and compiles very quickly.
The downside of C# however is that by default, it requires a Runtime installed.

So in the end, the problem isn't really about which language, but what is the best practice to package our code to run on non C# programmers' machines.

As a young programmer, I picked InnoSetup. It was stupid easy (copy paste basically), however my installer was not perfect. For example, you can only install per user, you can't change the install location, and my app itself
stored settings in the installation location instead of AppData. Very unprofessional. It was also not straight forward to package for Microsoft Store a few years later. It's probably easier now, but still.

Then I started Tauri, which uses Wix by default but the problem with Wix is that it requires administrative priviledges to run the installer by default. Tauri has also added NSIS support now, but I haven't worked with it yet.
I really don't want to learn something new that I am not fully sold so then I came accross MSIX which I had heard about already and Windows Installer (MSI).

I was curious as to how Microsoft has made the DX for C#, so I found Microsoft Visual Studio Installer Projects 2022

Lastly, I searched on GitHub for a something that uses MSIX and C#, and couldn't find anything so here we are.

## Reproducibility

1. Install Visual Studio with C#/UWP desktop app pre-requisite, the latest Windows SDK, and Windows 10 version 1803 SDK
2. Create a console or deskto application. For the former, you **must** create Release x64 configuration as packaging will fail 
3. [Follow this guide](https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-packaging-dot-net)
	- Set minimum version to Windows 10 version 1803, as that is the earliest version to support auto-updating
    - If Console Application, set minimum to version 1903 so that you can use uap8 extensions
4. Modify Package.appxmanfiest
	- Visual assets stored in Images directory
	- Packaing window needs a unique GUID not template's
5. In `Configure Startup Projects...` under `Configuration Properties`, ensure that both projects are targeting the same architecutre (i.e. both should target x64)

To create the installer, simply right click the packaging project, and select `Publish > Create App Packages`. When testing, self-sign certificate (Create / Select from store) and **install** this certificate to be able to install the MSIX that is created.

Remember that when you are ready to publish, either publish to the store, or

- Check create app bundle only if needed (for console applications at least)

If the package has been installed, use "debug installed app package"

### Console Applications

Since MSIX is for C# Desktop apps, where does that leave users like us where the App does not open in its own terminal? Well let's see what the Terminal app does.


```xml
<uap3:Extension Category="windows.appExecutionAlias" Executable="wt.exe" EntryPoint="Windows.FullTrustApplication">
    <uap3:AppExecutionAlias>
        <desktop:ExecutionAlias Alias="wt.exe" />
    </uap3:AppExecutionAlias>
</uap3:Extension>
```

```wapproj
<PropertyGroup Label="Configuration">
    <OCExecutionAliasName Condition="'$(OCExecutionAliasName)'==''">wt</OCExecutionAliasName>
</PropertyGroup>
```

Interesting. What are these [uap3:Extensions](https://learn.microsoft.com/en-us/uwp/schemas/appxpackage/uapmanifestschema/element-uap3-extension-manual)?
Further hunting landed me [here](https://learn.microsoft.com/en-us/uwp/schemas/appxpackage/uapmanifestschema/element-uap3-appexecutionalias).
I can't wait till we can train AI on documentation and get feedback quickly instead of manually reading the docs.

`Child of <Applications>`

```xaml
<Extensions>
    <uap3:Extension Category="windows.appExecutionAlias" >
        <uap3:AppExecutionAlias>
        <uap8:ExecutionAlias Alias="nvm2.exe" />
        </uap3:AppExecutionAlias>
    </uap3:Extension>
</Extensions>
```
