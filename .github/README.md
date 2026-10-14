<img src="logo.png">

This repository contains an SDK that can be used to create modifications ("mods") for (or port Doom3 mods to) [KROOM 3](https://github.com/FriskTheFallenHuman/KROOM-3).



It contains (mostly) the same source files as the gpl release of Doom 3 BFG, but these are taken from KROOM 3 and **are licensed under GPLv3, not the SDK license**.

## Compiling Instructions

Compiling its straight forward following these steps:

- For Windows 10/11:
  
  * Install Visual Studio 2022 Community Edition With C++ Workspace.
  * For x64/x86 run **cmake_msvc.cmd** bat file.

- For Windows 7 (Untested):
  
  * Install Visual Studio 2019/2017
  * [Download and Install DirectX SDK June 2010](https://www.microsoft.com/en-us/download/details.aspx?id=6812)
  * For x64/x86 run **cmake_msvc.cmd** bat file.

- For Debian base distros
  
  * Open a terminal in **neo/** folder and run:
    
    ```
    chmod +x cmake_linux.sh
    ```
  
  * In the same terminal run:
    
    ```
    ./cmake_linux.sh gcc debug
    ```

# 
