---
title: SSH连接不上Vmware Ubuntu服务器
description: vscode remote-ssh 连接不上 vmwaer ubuntu 虚拟机服务器，原因是 ubuntu没有ipv4地址
tags:
created: 2026-06-30
updated: 2026-06-30
number headings: first-level 1, start-at 1, max 3, 1.1, auto, contents toc, off
---
使用 Vscode 的 remote-ssh 插件，连接不上 Vmware Ubuntu 虚拟机

错误日志：
```bash
[18:21:36.053] Log Level: 2
[18:21:36.071] VS Code version: 1.126.0
[18:21:36.072] Remote-SSH version: remote-ssh@0.124.0
[18:21:36.072] win32 x64
[18:21:36.087] SSH Resolver called for "ssh-remote+are-vm-ubuntu", attempt 1
[18:21:36.102] remote.SSH.useLocalServer = false
[18:21:36.103] remote.SSH.useExecServer = true
[18:21:36.103] remote.SSH.bindHost = {}
[18:21:36.104] remote.SSH.showLoginTerminal = false
[18:21:36.105] remote.SSH.remotePlatform = {"devenvc_ha51c8.7f6cbb64037e46928b908ca985eeb123.0":"linux","are-vm-ubuntu":"linux"}
[18:21:36.107] remote.SSH.path = 
[18:21:36.107] remote.SSH.configFile = 
[18:21:36.108] remote.SSH.useFlock = true
[18:21:36.108] remote.SSH.lockfilesInTmp = false
[18:21:36.109] remote.SSH.localServerDownload = auto
[18:21:36.109] remote.SSH.remoteServerListenOnSocket = false
[18:21:36.110] remote.SSH.defaultExtensions = []
[18:21:36.110] remote.SSH.defaultExtensionsIfInstalledLocally = []
[18:21:36.111] remote.SSH.loglevel = 2
[18:21:36.111] remote.SSH.enableDynamicForwarding = true
[18:21:36.112] remote.SSH.enableRemoteCommand = false
[18:21:36.112] remote.SSH.serverPickPortsFromRange = {}
[18:21:36.113] remote.SSH.serverInstallPath = {}
[18:21:36.114] remote.SSH.permitPtyAllocation = false
[18:21:36.115] remote.SSH.preferredLocalPortRange = undefined
[18:21:36.115] remote.SSH.useCurlAndWgetConfigurationFiles = false
[18:21:36.116] remote.SSH.experimental.chat = true
[18:21:36.117] remote.SSH.experimental.enhancedSessionLogs = true
[18:21:36.118] remote.SSH.httpProxy = {"*":""}
[18:21:36.118] remote.SSH.httpsProxy = {"*":""}
[18:21:36.133] SSH Resolver called for host: are-vm-ubuntu
[18:21:36.133] Setting up SSH remote "are-vm-ubuntu"
[18:21:36.145] Using commit id "7e7950df89d055b5a378379db9ee14290772148a" and quality "stable" for server
[18:21:36.146] Extensions to install: 
[18:21:36.155] Install and start server if needed
[18:21:36.161] Checking ssh with "C:\Program Files\Git\usr\local\bin\ssh.exe -V"
[18:21:36.164] Got error from ssh: spawn C:\Program Files\Git\usr\local\bin\ssh.exe ENOENT
[18:21:36.164] Checking ssh with "C:\Program Files\Git\bin\ssh.exe -V"
[18:21:36.165] Got error from ssh: spawn C:\Program Files\Git\bin\ssh.exe ENOENT
[18:21:36.166] Checking ssh with "C:\Windows\system32\ssh.exe -V"
[18:21:36.167] Got error from ssh: spawn C:\Windows\system32\ssh.exe ENOENT
[18:21:36.168] Checking ssh with "C:\Windows\ssh.exe -V"
[18:21:36.169] Got error from ssh: spawn C:\Windows\ssh.exe ENOENT
[18:21:36.170] Checking ssh with "C:\Windows\System32\Wbem\ssh.exe -V"
[18:21:36.171] Got error from ssh: spawn C:\Windows\System32\Wbem\ssh.exe ENOENT
[18:21:36.171] Checking ssh with "C:\Windows\System32\WindowsPowerShell\v1.0\ssh.exe -V"
[18:21:36.173] Got error from ssh: spawn C:\Windows\System32\WindowsPowerShell\v1.0\ssh.exe ENOENT
[18:21:36.174] Checking ssh with "C:\Windows\System32\OpenSSH\ssh.exe -V"
[18:21:36.210] > OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2

[18:21:36.217] Running script with connection command: "C:\Windows\System32\OpenSSH\ssh.exe" -T -D 53955 "are-vm-ubuntu" sh
[18:21:36.218] Generated SSH command: 'type "C:\Users\hjy\AppData\Local\Temp\vscode-linux-multi-line-command-are-vm-ubuntu-314323438.sh" | "C:\Windows\System32\OpenSSH\ssh.exe" -T -D 53955 "are-vm-ubuntu" sh'
[18:21:36.220] Using connect timeout of 17 seconds
[18:21:36.221] Terminal shell path: C:\WINDOWS\System32\cmd.exe
[18:21:36.397] > 
[18:21:36.398] Got some output, clearing connection timeout
[18:21:36.414] > 
[18:21:57.509] > ssh: connect to host 192.168.236.129 port 22: Connection timed out
> 过程试图写入的管道不存在。
[18:21:57.817] "install" terminal command done
[18:21:57.818] Install terminal quit with output: 过程试图写入的管道不存在。
[18:21:57.819] Received install output: 过程试图写入的管道不存在。
[18:21:57.821] WARN: $PLATFORM is undefined in installation script output.  Errors may be dropped.
[18:21:57.822] Failed to parse remote port from server output
[18:21:57.823] Resolver error: Error
    at y.Create (c:\Users\hjy\.vscode\extensions\ms-vscode-remote.remote-ssh-0.124.0\out\extension.js:2:722235)
    at t.handleInstallOutput (c:\Users\hjy\.vscode\extensions\ms-vscode-remote.remote-ssh-0.124.0\out\extension.js:2:720316)
    at t.tryInstall (c:\Users\hjy\.vscode\extensions\ms-vscode-remote.remote-ssh-0.124.0\out\extension.js:2:842913)
    at async c:\Users\hjy\.vscode\extensions\ms-vscode-remote.remote-ssh-0.124.0\out\extension.js:2:801927
    at async t.withShowDetailsEvent (c:\Users\hjy\.vscode\extensions\ms-vscode-remote.remote-ssh-0.124.0\out\extension.js:2:805164)
    at async A (c:\Users\hjy\.vscode\extensions\ms-vscode-remote.remote-ssh-0.124.0\out\extension.js:2:798392)
    at async t.resolve (c:\Users\hjy\.vscode\extensions\ms-vscode-remote.remote-ssh-0.124.0\out\extension.js:2:802578)
    at async c:\Users\hjy\.vscode\extensions\ms-vscode-remote.remote-ssh-0.124.0\out\extension.js:2:1095642
[18:21:57.831] ------
```


原因：Ubuntu虚拟机没有获取到 ipv4 地址

见[[Ubuntu虚拟机不显示ipv4地址]]