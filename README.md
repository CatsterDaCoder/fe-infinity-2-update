Please refer to SETUP.md for everything when it comes to setting things up.
99.9999% of the credit goes to https://github.com/i-am-neon for building Fe Infinity. I'm just some dumbass who ported it to windows while relying a bit toooo much on Claude.
Works with local models (use Ollama). Takes significantly longer to generate, however, but its free.
Tested with chatgpt-4o-mini. 4.1 mini works, but does so worse than regular 4o mini (dunno why).
Generating longer hacks is less reliable, but you should be able to get a workable hack in 1-3 attempts.
Please DM me on Discord if you have any questions, Ideas or need assistance.
When creating maps in FEmapCreator, please only use fe8 tilesets for now. 
Please clone FE Infinity, don't just directly download it as a zip off of github.

Known bugs: 
Certain tilesets sometimes shit themselves and die when used. Don't know why this happens.
Tileset/map importing is finicky at best. working on that now.
FEBuilderGBA does NOT like generated chapters and characters. difficult to read and edit data.
rarely ever generated recruits as enemies to be recruited. Truly have no idea why this happens.
Ai puts player starting positions inside of impassable terrain. Fixable in FEBuilder.
Chapter sometimes ends after random but highly specific actions (X unit attacks boss, X unit talks to Y unit, ect). Also don't know why this happens.
Any bug that occurs during generation is usually at fault of the model you're using. 4o is the most tested and most reliable. 

Whats being worked on:
Importing portraits
Bug fixes
Importing music
Creating reinforcements and more complex map objectives
A pure windows version (not using WSL). Don't hold your breath for this, though.

As much as i would love to provide a clean fe8 rom, I cannot. I'm sure that you can easily find tools on reddit or the internet archive that help you dump the rom off of your Fe8 cartridge that you totally have.

Commands to save somewhere in a text file for ease of use:

cd ~

git clone https://github.com/i-am-neon/fe-infinity.git

cd fe-infinity

git apply fe-infinity-webui-and-prompt-editor.patch

cd server

deno task webui 
