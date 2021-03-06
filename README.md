![Grakkit Logo](./grakkit.png)

**It's the fusion of GraalVM and Bukkit.** Grakkit is designed to allow the use of JavaScript within Minecraft -- a simple concept with infinite potential.

**This plugin is NOT for beginners!** If you are new to JavaScript, we highly recommend you try [ScriptCraft](https://github.com/walterhiggins/ScriptCraft) as its ecosystem is geared towards newcomers.

![Code Demo](./demo.gif)

[![Build Status](https://travis-ci.org/grakkit/grakkit.svg?branch=master)](https://travis-ci.org/grakkit/grakkit)

# Installation

## GraalVM (Windows)

Open a new CMD window and copy/paste the following commands:
```bat
@echo off
cls
set "target=%temp%\GraalVM-%RANDOM%"
powershell -command wget "https://github.com/graalvm/graalvm-ce-builds/releases/download/vm-20.0.0/graalvm-ce-java11-windows-amd64-20.0.0.zip" -OutFile "%target%.zip"
powershell -command Expand-Archive -Force "%target%.zip" "%target%"
cd %appdata%
if exist GraalVM rd /q /s GraalVM
ren "%target%\graalvm-ce-java11-20.0.0" "%appdata%\GraalVM"
del "%target%.zip"
exit
```

A traditional server start script may look something like this:
```bat
java -jar server.jar
```

With GraalVM installed, your start script should look like this:
```bat
"%appdata%\GraalVM\bin\java" -jar server.jar
```

If you're still confused, ask a developer on our [discord server](https://discord.gg/e682hwR) for help.

## GraalVM (Mac/Linux)
TBA

## Plugin
Once you have GraalVM set up with your server, the installation process is just as simple as any other plugin installation. Head on over to the [releases page](https://github.com/grakkit/grakkit/releases) and download the latest version's JAR file.

# Basics

## The User Script
Once you've installed the plugin, a `user.js` file will be created within `plugins/grakkit`. This file is your entry point to the world of JavaScript in Minecraft. You can use ES import/export syntax to bring other files into the mix. This file is executed during a refresh, reload, or restart.

## IntelliSense Support
The first time you boot up the server, typescript definition files will be downloaded. These files are used to bring full IntelliSense support, tested with VS code. Using `server`, `core.event`, and `core.type` as your entry points, your code will tab-complete valid methods for bukkit, spigot, and paper classes.

If you're unfamiliar with the server API, experiment around with whatever you can find here, see what does what. A few great places to start are `server.getOnlinePlayers()`, `server.getWorld('world')`, and `server.selectEntities(server.getConsoleSender(), '@e')`.

## The Scripts Folder
Any files in the top level of this folder will be executed directly after the user script. This makes it easy to import files from elsewhere by just dropping them in this folder.

## Custom Commands
Commands are created and registered with the `core.command` function. Feel free to use the examples below to get started with your own commands.

*Note: Commands will only show tab-completions if they are registered synchronously when the plugin loads.*
```javascript
core.command({
   name: 'smite',
   // permission required
   permission: 'example.command.smite',
   // error that displays if you don't have permissions
   error: '§cWho do you think you are? Thor?',
   execute: (player, target) => {
      if (target) {
         target = server.getPlayer(target);
         if (target) {
            target.getWorld().strikeLightning(target.getLocation());
            core.send(player, '§6Your target has been smitten.');
         } else {
            core.send(player, '§cThat player is offline or does not exist!');
         }
      } else {
         core.send(player, '§cUsage: /smite <target>');
      }
   },
   tabComplete: (player, ...args) => {
      if (args.length < 2) {
         const players = [ ...server.getOnlinePlayers() ].map((player) => player.getName());
         return players.filter((name) => name.includes(args[0]));
      }
   }
});

// urban dictionary definition grabber
// permission node: (none)
core.command({
   name: 'urban',
   execute: (player, term) => {
      if (term) {
         player.sendMessage('§7Searching...');
         try {
            const response = core.fetch(`https://api.urbandictionary.com/v0/define?term=${term}`);
            const json = response.json();
            if (json) {
               const entry = json.list[0];
               if (entry) {
                  core.send(player, `§6${entry.definition.split('\r\n')[0].replace(/[\[\]]/g, '')}`);
               } else {
                  core.send(player, '§cThat word does not have a definition!');
               }
            } else {
               core.send(player, '§cAn error occured in the urban dictionary database!');
            }
         } catch (error) {
            core.send(player, `§cAn error (HTTP ${error}) occured!`);
         }
      } else {
         core.send(player, '§cUsage: /urban <word>');
      }
   }
});
```

## Event Listeners
Event listeners are just that -- they listen for in-game events, like player deaths, entity spawns, etc. -- and execute code if and when those events take place.

For example, you could log useful information about a player to the console when they connect to the server:
```javascript
// detect player join
core.event('org.bukkit.event.player.PlayerJoinEvent', (event) => {
   const player = event.getPlayer();
   const location = player.getLocation();
   const world = location.getWorld().getName();
   const lines = [
      '-----------------------------------',
      `Name: ${player.getName()}`,
      `IP Address: ${player.getAddress().getHostName()}`,
      `Game Mode: ${player.getGameMode().toString()}`,
      `Location: { x: ${location.getX()}, y: ${location.getY()}, z: ${location.getZ()}, world: ${world} }`,
      `Health: ${player.getHealth()}/${player.getMaxHealth()}`,
      `Is Flying: ${player.isFlying() ? 'Yes' : 'No'}`,
      '-----------------------------------'
   ];
   lines.forEach((line) => console.log(line));
});
```

This event listener will allow players "retain" a certain percentage of their items upon death, with items to drop being chosen at random. `/gamerule keepInventory` must be enabled for this to work properly.
```javascript
const percentage = 30;
core.event('org.bukkit.event.player.PlayerRespawnEvent', (event) => {
   const player = event.getPlayer();
   const inventory = player.getInventory();
   inventory.setContents(
      [ ...inventory.getContents() ].map((item) => {
         if (percentage / 100 > Math.random()) {
            return item;
         } else {
            item && player.getWorld().dropItemNaturally(player.getLocation(), item);
            return null;
         }
      })
   );
});
```

## Persistent Data Storage
Let's say you want to store some data, and be able to access that data across a refresh, reload, or restart. The data function accepts a path-like string, such as the following:
```javascript
const data = core.data('example/data/path');
```
That `data` object is now linked to that path. When you refresh, reload, or restart the server, any data assigned to the object will be saved to a JSON file. If the data includes any circular references, they will be parsed out. If your data contains any value that is not an array, object, string, number, boolean, null, or undefined, it will also be parsed out.

In the following example, **the `grakkit/jx` module is used** to store a player's inventory with JSON while they're offline. When they log in, the stored data is parsed back into their inventory.

```javascript
const $ = core.import('@grakkit/jx');

// detect player join
$('*playerJoin').do((event) => {

   // get player data
   const player = $(event.getPlayer());
   const data = core.data(`player/${player.uuid()}/inventory`);

   // load inventory
   data.items && player.inventory($(data.items));
});

// detect player quit
$('*playerQuit').do((event) => {

   // get player data
   const player = $(event.getPlayer());
   const data = core.data(`player/${player.uuid()}/inventory`);

   // save inventory
   data.items = player.inventory().serialize();
});
```

# Modules

## Using Modules
Modules are the secret sauce of Grakkit. To add a module, use `/module add <repo>`. If needs be, you can update a module to the latest version with `/module update <repo>` or update all installed modules with `/module update *`. To remove a module, use `/module remove <repo>`.

Once you've added a module, you can use `core.import('@<repo>')` to import it.

## Create Your Own
Modules are hosted on GitHub. You can fork [this repository](https://github.com/grakkit/example) to get a head start, or follow the example below.

In your `index.js` file, use the `core.export` function to export your module, like so:
```javascript
core.export('hello world');
```
Given the above code is hosted at the `grakkit/test` repository, the following will be true:
```javascript
core.import('@grakkit/test') === 'hello world';
```

Now, the `core.export` function is ONLY used to export your module. In-module file loading is done with ES module syntax. Let's say you have another file in the same folder as `index.js`, for example, `crypto.js`:
```javascript
const SecureRandom = Java.type('java.security.SecureRandom');
const crypto = new SecureRandom();

export const random = () => {
   return (crypto.nextInt() + 2147483648) / 4294967296;
}
```
Given the above, you can use the following to import and call the `random` function from within the `index.js` file:
```javascript
import { random } from './crypto.js';

console.log(`A cryptographically-secure random number: ${random()}`);
```

JSON can also be imported this way. Just format your JSON file like this:
```json
export default {
   "a": "x",
   "b": [
      "y"
   ],
   "c": {
      "z": "?"
   }
}
```

And import like so:
```javascript
import test from './example.json';

console.log(`Imported JSON: ${test}`);
```

# Miscellaneous

## Legacy Support
Grakkit theoretically supports Minecraft versions going back to beta 1.4, given you have bukkit installed. This legacy support does not extend to modules or typescript definitions, and likely never will. If you do intend to use grakkit on a beta or legacy release version, you may run into unexpected problems, however the `index.js` and `user.js` files should work properly in their default state.

## A note about ScriptCraft
We have a module just for those who are looking to transition from ScriptCraft. With Grakkit installed, use `/module add grakkit/scriptcraft` to install it, and add the following at the top of your `user.js` file:
```javascript
core.import('@grakkit/scriptcraft');
```

If you have a pre-existing ScriptCraft directory, you should temporarily rename or move it so that the modified compat version can be installed, then copy in your custom scripts and modules.

Anyways, once you've got that, reload the server. The scriptcraft directory should generate itself within your server's folder, and from that point on, things should work just like they did with ScriptCraft.

You can then copy your plugins and modules back into the folder. You may run into a few small bugs, but once those are fixed, you should be good. 90-95% of ScriptCraft is supported in Grakkit.

We fully respect and support ScriptCraft and what it did. For us, at least in part, Grakkit is a continuation of what ScriptCraft aimed to do, with plenty of our own ideas and features thrown in. **We don't intend to start a rivalry between supporters of the two plugins.**

---

Grakkit is owned and maintained by RepComm and hb432.