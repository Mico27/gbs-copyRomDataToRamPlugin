# gbs-copyRomDataToRamPlugin
 Custom events to read banked data into variables and compile tileset or scene arrays.

Warning: this plugin must require knowledge of how data is handled

This plugin can read in any banked data if you know its symbol (usualy is the name of the file/array/etc)
It also contains events to compile an array of tileset or scene far pointers in ROM to be read via index.

Events:
- Copy ROM data to variable

  <img width="524" height="210" alt="image" src="https://github.com/user-attachments/assets/efb3f77d-75e5-4fbf-991b-875f8b83d6e2" />

Will store in the Variable the value inside ROM data (specified by the custom data symbol) at the specified data offset/index.

- Compile tileset array

<img width="589" height="497" alt="image" src="https://github.com/user-attachments/assets/7620f64a-3991-45ef-a357-3f85b32ccf7d" />

Can fetch the tileset pointer via an index, for example to pass on the Replace Tileset Tiles Ex event from https://github.com/Mico27/gbs-replaceTilesetTilesPlugin

<img width="550" height="854" alt="image" src="https://github.com/user-attachments/assets/5d7c013a-740d-4fb3-a57e-817df2bbe0e0" />

- Compile scene array

<img width="587" height="399" alt="image" src="https://github.com/user-attachments/assets/48a0446b-6004-4b72-a1a9-cd1948b87431" />

same as the compile tileset array but for scenes. Can be used to change scene via an index with GBVM or with the Submapping events from https://github.com/Mico27/GBS-SubmappingExPlugin

<img width="551" height="1102" alt="image" src="https://github.com/user-attachments/assets/c8514a7b-6a9a-4bad-877c-5898724cc79b" />

You can also create your own ROM data by creating the c file manualy. 

Note, if you specify a Data Length in the event higher than 2 (2 bytes / 16 bits) the data copied will overflow to the next variable after the one specified.
Useful if you want to load data that you want to load in multiple variable at once.
