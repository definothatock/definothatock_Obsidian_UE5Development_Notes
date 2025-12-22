
https://eik.betide.studio/getting-started/plugin-installation

### Difference: GitHub vs. Fab (Marketplace)

The "EIK" plugin (EOS Integration Kit) is developed by Betide Studio.

- **GitHub Version (Free/Open Source):**
    
    - **Source Code:** You get the raw C++ code (like the files you uploaded).
    - **Maintenance:** It is often the "bleeding edge." It might contain new features that haven't been packaged yet, but it might also have bugs that haven't been ironed out.
    - **Setup:** You must compile it yourself. You need to put it in your `Plugins` folder and regenerate project files. If you are using a Blueprint-only project, this forces you to convert it to a C++ project.
    - **Support:** Support is usually community-based (Discord, GitHub issues).
- **Fab / Marketplace Version (Paid):**
    
    - **Convenience:** It comes pre-compiled. You can install it directly into the engine via the Epic Games Launcher. You do _not_ need a C++ project to use it.
    - **Stability:** The marketplace version is usually a "stable release" that has been tested more thoroughly.
    - **Support:** Paying often grants you a verified role in their Discord or priority support from the developer.
    - **Updates:** Updates are handled through the Launcher, which is easier than pulling new code from Git and recompiling.

**In short:** The code inside is 95-100% identical. You are paying for **convenience, stability, and support.**

---

### 2. Comparison: EIK vs. Standard VOIP (Built-in)

You asked if you should "just use VOIP." Unreal Engine has a built-in VOIP system, but it is very different from EOS Voice (which EIK uses).

|Feature|Standard UE VOIP (Steam/Null)|EOS Voice (via EIK)|
|---|---|---|
|**Technology**|Peer-to-Peer (P2P).|Server-Based (RTC).|
|**Bandwidth**|High. Audio is sent from every player to every other player directly. (10 players = 90 connections).|Low. Audio goes to one server, which mixes/routes it to others.|
|**Quality**|Okay, but relies on player internet speeds.|High quality, reliable, handled by Epic's servers.|
|**Cross-Platform**|Difficult. Steam VOIP only works on Steam.|**Excellent.** Works on PC, Console, Mobile seamlessly.|
|**Setup Difficulty**|Medium (Config files).|Low (Plugin) to High (Manual C++).|
|**Cost**|Free.|Free (Epic Online Services is free).|

**Verdict:** **Do not use the standard UE VOIP** unless you are making a very simple LAN game or a Steam-only game with max 4 players. EOS Voice (EIK) is superior in almost every way for modern development.

---

### 3. Recommendation: What should you do?

Here is my advice based on your experience level:

#### **Option A: Buy the Plugin on Fab (Recommended)**

**Do this if:**

- You want to focus on making your _game_, not building engine tools.
- You are not comfortable with C++, compiling plugins, or debugging linker errors.
- You want it to "just work" with Blueprints.
- You have the budget (it saves you 20-40 hours of work).

#### **Option B: Use the GitHub Version**

**Do this if:**

- You have $0 budget.
- You are comfortable with Visual Studio and C++.
- You don't mind fixing occasional setup errors yourself.
- You want to modify the code (e.g., change how the 3D audio attenuation works).

#### **Option C: Build it Yourself (The "Do all these" option)**

**Do this if:**

- You are learning C++ and want to understand exactly how the EOS SDK works.
- You need a _tiny_, lightweight implementation and don't want the bloat of a massive plugin.

### Final Verdict

**Buy the plugin (or use the free GitHub version if you know C++).**

Recreating the code I analyzed in the previous prompt is a great learning exercise, but for a production game, using the full EIK plugin is much safer. It handles edge cases (like disconnecting, microphone switching, and login failures) that you would likely forget to implement yourself.

https://poe.com/s/dVdTVeqP1wpafb0PmxiR