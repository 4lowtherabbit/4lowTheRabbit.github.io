# Debug .NET Core In-Process Dumps

Since ASP.NET Core version 2.2, the web application is hosted in-process in w3wp.exe by default, instead of out-of-process in dotnet.exe as a child process of w3wp.exe before. This brings the new challenge for memory dump debugging, since now the debugger shows the following error when I am trying to dump out the managed call stack for a .NET Core thread:
<pre>
0:035> !CLRStack
OS Thread Id: 0x362c (35)
Child SP       IP Call Site
GetFrameContext failed: 1
00000000 00000000 
</pre>

## Cause
The issue is due to .NET Framework clr.dll is loaded into w3wp too, together with .NET Core's coreclr.dll, which confuses the debugger.

## Resolution
<pre>
0:035>.unload
...
0:035>.cordll -D
...
0:035> lmvm coreclr
...
    Image path: D:\Program Files (x86)\dotnet\shared\Microsoft.NETCore.App\2.2.2\coreclr.dll
...
.cordll -ve -u -I coreclr -lp "C:\Program Files (x86)\dotnet\shared\Microsoft.NETCore.App\2.2.2"
0:035>!clrstack
...
</pre>

Using the above commands to
1. Unload the incorrect one first
2. Disable CLR debugging and unload the CLR debugging modules.
3. Find out the .NET Core version loaded by the debugee process, which is 2.2.2, in this sample.
4. -u to unload the incorrect one first and then load the specified one for coreclr at the specified folder with -I and -lp
You can verify the solution either by
* Running !Threads 
* Or running !clrstack on a .NET Core thread.

This solution can be applied to Function App V2 too, which hosts .NET Core in w3wp.exe too.