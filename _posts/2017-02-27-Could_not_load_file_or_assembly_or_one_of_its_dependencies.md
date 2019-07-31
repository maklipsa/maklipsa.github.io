---
layout: post
title: Could not load file or assembly or one of its dependencies.
description: "This post is an analysis of 'Could not load file or assembly'. Why it happens and how to diagnose it."
modified: 2017-02-27
tags: [.NET, cookit, Fusion log, assembly, assembly binding, assembly redirect]
image:
  feature: data/2017-02-27-Could_not_load_file_or_assembly_or_one_of_its_dependencies/logo.jpg
---

<link rel="stylesheet" href="/assets/css/tooltips_style.css">

In most cases, .NET manages to solve the [DLL hell problem](https://en.wikipedia.org/wiki/DLL_Hell) pretty well, but sometimes it fails. When it does, in the best-case scenario, we see something like this:

```console
Could not load file or assembly 'XXXX, Version=X.Y.Z.W, Culture=neutral, PublicKeyToken=eb42632606e9261f' or one of its dependencies. 
The located assembly's manifest definition does not match the assembly reference. (Exception from HRESULT: 0x80131040)
```

The much worst case is this:

```console
The method 'XXXX' was not found on the interface/type 'YYYY, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null'.
```

This post is an analysis of why this happens and how to diagnose it.
<!--MORE-->

To state the obvious:

<div class="center">
    <div class="button" >There is a log for that.</div>
</div>

In this case, it only needs to be turned on.

## Enable assembly binding logging (Fusion log)

Assembly binding logging is by default turned off since it affects performance and those logs can eat up a lot of space.

Assembly binding can be turned on using those registry settings:

- `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Fusion\EnableLog` - Type of DWORD
- `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Fusion\ForceLog` - Type DWORD
- `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Fusion\LogFailures` - Type DWORD
- `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Fusion\LogResourceBinds` - Type DWORD
- `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Fusion\LogPath` - Type String

The first four should be set to `1` and the last to an **existing** directory path (it should also end with a `\`).

It can be done manually with the use of `regedit.exe`, but since it can be automated, here are the scripts:

[![Disable Fusion Log](/data/2017-02-27-Could_not_load_file_or_assembly_or_one_of_its_dependencies/reg.png){: .regIco}Disable Fusion Log](/data/2017-02-27-Could_not_load_file_or_assembly_or_one_of_its_dependencies/disable-fusionLog.reg)
[![Enable Fusion Log](/data/2017-02-27-Could_not_load_file_or_assembly_or_one_of_its_dependencies/reg.png){: .regIco}Enable Fusion Log](/data/2017-02-27-Could_not_load_file_or_assembly_or_one_of_its_dependencies/enable-fusionLog.reg)

> CHECK THE FILES CONTENT BEFORE RUNNING! (they are just text files)
> DON'T RUN THIS OR ANY OTHER SCRIPT WITHOUT CHECKING. 
> MODIFYING REGISTRY REQUIRES ADMINISTRATOR PERMISSIONS, SO A LOT OF HARM CAN BE DONE WHEN RUNNING A SCRIPT FORM A NOT TRUSTED SOURCE.
 
After enabling assembly binding logging, there are two ways to continue:

## Analyse assembly binding logs with Fuslogvw 

Windows has a built-in tool called `Fuslogvw.exe`. It should be located in several places, but the pattern is: `C:\Program Files (x86)\Microsoft SDKs\Windows\<<SDK_VERSION>>\bin\NETFX <<RUNTIME_VERSION>> Tools\`.
Any version will do since the tool is available from [.NET version 1.1](https://msdn.microsoft.com/en-us/library/e74a18c4(v=vs.71).aspx), and the version numbers between 4.6.2 and 4.0 differ only in minor version.
The tool is simple, so I won't describe how it works. I prefer the other way:

## Reading assembly binding log files (Fusion Log)

The log may look intimidating, but knowing a few things makes them easy to read.

We first have to identify the assembly binding logs for the assembly causing problems.
Assemblies are identified using a pattern like this:

`[assembly name], Version=[assembly version], Culture=[culture], PublicKeyToken=[public token]`

In our example we are having problems with NLog, so the string we are looking for will be this:

`NLog, Version=4.0.0.0, Culture=neutral, PublicKeyToken=5120e14c03d0593c`

We will find entries that succeeded and ones that failed.
The first scenario is more straightforward, so let's look at it first.


### A successful operation: The operation completed successfully.

First an easy one - the one that succeeded.
<br/>
<pre>
  <article class="fusionLogText" >
  *** Assembly Binder Log Entry  (<span class="hint--top hint--always" aria-label="Date and time when the runtime attempted to locate the assembly.">25.02.2017 @ 13:44:54</span>) ***<br/>
  <br/>
  <span class="hint--right hint--success hint--always" aria-label="The result of this try." >The operation was successful.</span><br/>
  Bind result: hr = 0x0. The operation completed successfully.</a><br/>
  Assembly manager loaded from:  <span class="hint--right hint--always" aria-label="The exact runtime manager responsible for locating the assembly.">C:\Windows\Microsoft.NET\Framework\v4.0.30319\clr.dll</span><br/>
  Running under executable  <span class="hint--right hint--always" aria-label="Executing assembly. This assembly contains the starting point of the process.">d:\src\FusionLogTest\FusionLogRunner\bin\Debug\FusionLogRunner.exe</span><br/>
  <br/>
  --- A detailed error log follows.<br/>
  === Pre-bind state information ===<br/>
  LOG: <span class="hint--right hint--always" aria-label="Exact assembly that is being looked for.">DisplayName = NLog, Version=3.2.1.0, Culture=neutral, PublicKeyToken=5120e14c03d0593c</span><br/>
  (Fully-specified)<br/>
  LOG: <span class="hint--right hint--always" aria-label="Folder of the executing assembly.">Appbase = file:///d:/src/FusionLog/FusionLogRunner/bin/Debug/</span><br/>
  LOG: Initial PrivatePath = NULL<br/>
  LOG: Dynamic Base = NULL<br/>
  LOG: Cache Base = NULL<br/>
  LOG: <span class="hint--right hint--always" aria-label="Name of the executing assembly.">AppName = FusionLogRunner.exe</span><br/>
  <span class="hint--right hint--info hint--always" aria-label="The assembly that has the reference to the currently called assembly. A very useful information when debuging large systems.">Calling assembly : FusionLogLib, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.</span><br/>
  ===
  LOG: This bind starts in default load context.<br/>
  LOG: <span class="hint--right hint--always" aria-label="Configuration file for the executable that initialized the process.">Using application configuration file: d:\src\private\FusionLog\FusionLogRunner\bin\Debug\FusionLogRunner.exe.Config</span><br/>
  LOG: Using host configuration file: <br/>
  LOG: Using machine configuration file from C:\Windows\Microsoft.NET\Framework\v4.0.30319\config\machine.config.<br/>
  LOG: <span id="redirect" class="hint--right hint--info hint--always" aria-label="The runtime found a redirect for this assembly to version 4.0.0.0">Redirect found in application configuration file: 3.2.1.0 redirected to 4.0.0.0.</span><br/>
  LOG: Post-policy reference: NLog, Version=4.0.0.0, Culture=neutral, PublicKeyToken=5120e14c03d0593c<br/>
  LOG: <span class="hint--right hint--success hint--always" aria-label="Info that it managed to locate the redidected assembly, and the location of that assembly that is being loaded.">Binding succeeds. Returns assembly from d:\src\private\FusionLog\FusionLogRunner\bin\Debug\NLog.dll.</span><br/>
  LOG: Assembly is loaded in default load context.<br/>
  </article>
</pre>

### A failed one: The operation failed.

Now for something harder, but more interesting. A failed log file. Also for NLog.

<br/>

<pre>
<article class="fusionLogText" >
*** Assembly Binder Log Entry  (<span class="hint--top hint--always" aria-label="Date and time when the runtime attempted to locate the assembly.">25.02.2017 @ 14:46:28</span>) ***<br/>
<br/>
<span class="hint--right hint--always hint--error" aria-label="Result of the operation.">
The operation failed.</span><br/>
<span class="hint--right hint--always hint--error" aria-label="Error code.">Bind result: hr = 0x80131040. No description available.</span><br/>
Assembly manager loaded from:  <span class="hint--right hint--always" aria-label="The exact runtime manager responsible for locating the assembly.">C:\Windows\Microsoft.NET\Framework\v4.0.30319\clr.dll</span><br/>
Running under executable  <span class="hint--right hint--always" aria-label="Executing assembly. This assembly contains the starting point of the process.">d:\src\FusionLogTest\FusionLogRunner\bin\Debug\FusionLogRunner.exe</span><br/>
<br/>
--- A detailed error log follows. <br/>
<br/>
=== Pre-bind state information ===<br/>
LOG: <span class="hint--right hint--always" aria-label="Exact assembly that is being looked for.">DisplayName = NLog, Version=4.0.0.0, Culture=neutral, PublicKeyToken=5120e14c03d0593c</span><br/>
(Fully-specified)<br/>
LOG: <span class="hint--right hint--always" aria-label="Folder of the executing assembly.">Appbase = file:///d:/src/FusionLog/FusionLogRunner/bin/Debug/</span><br/>
LOG: Initial PrivatePath = NULL<br/>
LOG: Dynamic Base = NULL<br/>
LOG: Cache Base = NULL<br/>
LOG: <span class="hint--right hint--always" aria-label="Name of the executing assembly.">AppName = FusionLogRunner.exe</span><br/>
<span class="hint--right hint--info hint--always" aria-label="The assembly that has the reference to the currently called assembly. A very useful information when deugging large systems.">Calling assembly : FusionLogLib, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.</span><br/>
===<br/>
LOG: This bind starts in default load context.<br/>
LOG: <span class="hint--right hint--always" aria-label="Configuration file for the executable that initialized the process.">Using application configuration file: d:\src\private\FusionLog\FusionLogRunner\bin\Debug\FusionLogRunner.exe.Config</span><br/>
LOG: Using host configuration file: <br/>
LOG: Using machine configuration file from C:\Windows\Microsoft.NET\Framework\v4.0.30319\config\machine.config.<br/>
LOG: <span class="hint--right hint--always" aria-label="The runtime found a redirect for this assembly to version 4.0.0.0">Redirect found in application configuration file: 3.2.1.0 redirected to 4.0.0.0.</span><br/>
LOG: Post-policy reference: NLog, Version=4.0.0.0, Culture=neutral, PublicKeyToken=5120e14c03d0593c<br/>
LOG: <span class="hint--right hint--always" aria-label="By this time the runtime knows that the executable doesn't contain exact version, so it tries to look in the GAC, and fails.">GAC Lookup was unsuccessful.</span><br/>
LOG: <span class="hint--right hint--always" aria-label="No exact version was found, so the runtime checks the executable directory">Attempting download of new URL file:///d:/src/FusionLog/FusionLogRunner/bin/Debug/NLog.DLL.</span><br/>
LOG: <span class="hint--right hint--always" aria-label="The runtime tries to load this file to satisfy the assembly requierment.">Assembly download was successful. Attempting setup of file: d:\src\FusionLog\FusionLogRunner\bin\Debug\NLog.dll</span><br/>
LOG: Entering run-from-source setup phase.<br/>
LOG: <span class="hint--right hint--always" aria-label="Info about the assembly found in the executable directory.">Assembly Name is: NLog, Version=3.2.1.0, Culture=neutral, PublicKeyToken=5120e14c03d0593c</span><br/>
<span class="hint--right hint--warning hint--always" aria-label="Found assembly has a version mismatch. This can still be fixed using assembly redirect.">WRN: Comparing the assembly name resulted in the mismatch: Major Version</span><br/>
<span class="hint--right hint--error hint--always" aria-label="Because there was no assembly rediredct version mismatch is an error. ">ERR: The assembly reference did not match the assembly definition found.</span><br/>
<span class="hint--right hint--error hint--always" aria-label="Error code">ERR: Run-from-source setup phase failed with hr = 0x80131040.</span><br/>
<span class="hint--right hint--error hint--always" aria-label="Finall error code">ERR: Failed to complete setup of assembly (hr = 0x80131040). Probing terminated.</span><br/>
</article>
</pre>

## The problem

We can see that dot net runtime is looking for version `4.0.0.0`, but found version `3.2.1.0`. Because of this, it reports a warning `Comparing the assembly name resulted in the mismatch: Major Version` and fails in the next line.
We know the problem. Now for the solution.
 
# How to fix "Could not load file or assembly"?

There can be many causes why the assembly fails to load, but 99% of them can boil down to two solutions presented below:

## Update the reference

One project in the solution has a different version of the assembly. 
The build process copies a lot of files. The dlls can be copied with their dependencies to the destination folder overwriting the new version of the dependency with the old one.

A good practice is to have all references pointing to a single version of a given assembly. 
The easiest way to make sure that this is the case is to use Visual Studio `Package Manager` (right-click on the solution):

![](/data/2017-02-27-Could_not_load_file_or_assembly_or_one_of_its_dependencies/Package Explorer.png)

And then:
![](/data/2017-02-27-Could_not_load_file_or_assembly_or_one_of_its_dependencies/ManagePackagesForSoution.png)

It displays all package versions used in the solution.
Update them, build, test, commit and push.<br/>
And you should be good.
Things may not always be that as easy, because we can't control references that external libraries. This is where the second solution comes in:

## Add assembly binding redirect

Look at the succeded log file, and <a href="#redirect" >the entry about a found redirect</a>.
The runtime can be instructed to use another version of the assembly than the one that was called. 
In other words, it can be redirected. 
Binding redirects have to be defined in the config file of the **executing assembly**.
The runtime will ignore binding redirects present in not executing assemblies.

In case of a Windows application, it's the `app.config` file, and in a case of an ASP app the config file structure (web.configs and down)<br/>
An example:

```xml
<configuration>
  <runtime>
    <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
      <dependentAssembly>
        <assemblyIdentity name="NLog" publicKeyToken="5120e14c03d0593c" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-4.0.0.0" newVersion="4.0.0.0" />
      </dependentAssembly>
    </assemblyBinding>
  </runtime>
</configuration>
```

This entry is saying that all assemblies matching all the rules:

- named `NLog`
- having a public key token with value `5120e14c03d0593c`
- culture equal to `neutral`
- having version between `0.0.0.0` and `4.0.0.0`

Should be redirected to the assemble with version `4.0.0.0`.

## Troubleshooting FusionLog

### Q: I don't see the logs from my application

**A:** The application **has to be restarted** after enabling the log. When talking about IIS application the whole IIS process has to be restarted.

### Q: I don't see the folder

**A:** You have to create the folder manually

### Q: The application runs slower

**A:** The logging adds some overhead, but not enough for it to be seen with a bare eye. Maybe You are doing a lot of dynamic assembly loading? 
 
### Q: Something is eating up my disc space

**A:** Disable the log. When enabled it creates a lot of small files. The result is that they are taking up more disc space than they actual size.  

### Q: The logs are still being created despite disabling the logging

**A:** Restart the application process.

### Q: Why is it called fusion log?

**A:** I suspect because [fusion is the process of combining two atoms into one.](https://en.wikipedia.org/wiki/Fusion)

<style>
.regIco{
    height:100px;
}
.fusionLogText{
    margin: 1em 0px 1em;
    background-color: white;
    border-color: lightgray;
    border-width: 1px;
    border-style: solid;
    padding: 1em;
}
</style>