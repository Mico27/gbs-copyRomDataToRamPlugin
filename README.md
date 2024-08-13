# gbs-copyRomDataToRamPlugin
 Custom event to read banked data into variables

Warning: this plugin must require knowledge of how data is handled

This plugin can read in any banked data if you know its symbol (usualy is the name of the file/array/etc)

Example of use in script:
![image](https://github.com/user-attachments/assets/5fdc2824-cc87-4c01-8697-47443a06509e)

You can see the example banked data in copyRomDataToRamPlugin\engine\src\data and modify/add your owns in it.
If you add any custom struct, you can modify/add it in copyRomDataToRamPlugin\engine\include\custom_types.h.

Note, if you specify a Data Length in the event higher than 2 (2 bytes / 16 bits) the data copied will overflow to the next variable after the one specified.
Useful if you want to load data that you want to load in multiple variable at once.
