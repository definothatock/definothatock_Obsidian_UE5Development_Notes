#### _**-> LIVE CODING ONLY FOR EDITS INSIDE OF A FUNCTION**_

- you can use live coding as long as there is no great change in builds and binaries, e.g. edit is within a function, which within a cpp file.
    
- you still have to build from the compiler after you are done developing
    

#### _**-> DISABLE HOT LOAD ENTIRELY**_

- obsolete hotshit
    

#### _**-> Files NOT WITHIN private/public hierarchy will BRICK UE**_

- Remember to select what exposure! no custome file in cpp root (Source)
    

→ For quicker dev prototyping, use _BP to try out_ first, Cpp is _**extremely**_ tedious! ←

→ For small changes within existing Cpp files, ‌Build/Binaries nuke is not a must; a Simple rebuild should be able to handle. ←


Thx to @.petraefa from DuridMech Game Dev,
## FULL C++ Work flow:

Create
- _Start here if_ - Adding a _**NEW**_ Cpp file:
    
    - Tools->Add C++, create the Class and choose _**Private/Public**_ (if needed, create extra dir via the text box, NOT in file explorer)
        
    - wait for the generation, then _**Close Editor**_
        
- _Start here if_ - Cpp file edit:
    
    - In File Explorer, project dir (skippable if small edit) :
        
        - **Remove** _Binaries, Intermediate, Saved_ (references pointers)
            
        - **Generate** VS pj files from .uproject (right click)
            
    - In VS:
        
        - Find the project folder, choose the folder as Startup Project (right click)
            
        - The actual coding (skip if not needed)
            
        - when finished, **Build** the project (ctrl + shift + B)
            
        - Check any fail from log ( e.g. Build: 1 succeeded, 0 failed, …)
            
        - Optional, press Ctrl + F5 to start Editor from VS

Remove

- delete Cpp file:
    
    - In VS:
        
        - select the files and directory, remove their entries by delete (file is NOT physically deleted)
            
    - In File Explorer, project dir:
        
        - go into this dir: Source/ ”pj name”/ ”_**BOTH PUBLIC AND PRIVATE**_”/ ”your files”
            
        - physically delete all your unwanted files
            
        - **Remove** _Binaries, Intermediate, Saved_ (references pointers)
            
            - also VS related files if compile error
                
        - **Generate** VS pj files from .uproject (right click)

Rename

- Renaming technically is auto-copy and auto-delete. HIGHTLY _**NOT**_ RECOMAMDED (BP / Asset may not recognize it).
    
    - If there is such need, better do it by replacement (manual rename):
        
        - create new Cpp file, same setting
            
        - copy and paste code segments
            
        - reassigned ALL pointers / relations towards the old file
            
        - delete the old file
- Alternatively, if you are using Rider, Just right click and choose Refactor (Renaming). Mostly Save.

==================

### Setup VS C++ (? - just what I think):

- Install .NET
    
- Install things in the check page
    
- _**DISABLE HOT LOAD (@.petraefa)**_
    
- Set force recompile on start
    


https://www.youtube.com/watch?v=MKBFG69UnFY


[https://forums.unrealengine.com/t/private-and-public-folder-structure/269996/2](https://forums.unrealengine.com/t/private-and-public-folder-structure/269996/2)

https://dev.epicgames.com/documentation/en-us/unreal-engine/setting-up-visual-studio-development-environment-for-cplusplus-projects-in-unreal-engine
https://dotnet.microsoft.com/en-us/download/dotnet/8.0

[https://www.bilibili.com/opus/623713793518015070](https://www.bilibili.com/opus/623713793518015070)

