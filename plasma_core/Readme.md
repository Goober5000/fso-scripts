
# Plasma Core FSO Script Management Library
## Version 1.0.0

Plasma Core is intended as a foundational system to keep libraries, modules, and scripted gameplay systems of all stripes tidy and enable live-reloading of their code without restarting the game or wiping state. Plasma Core should work with standard Lua modules just as well as Fennel modules, so long as they are structured for it.

The system has the following objectives:
* Streamline common aspects of FSO scripting
* Gently encourage a consistent structure for modules, so I'll stop
   reinventing the wheel every time I make a new one
* Support interactive development with the live reloading of modules
   following that structure

This code deliberately breaks with Fennel style and with many existing modules and uses snake_case exclusively. Amoung other reasons, this is intended to make the compiled code cleaner and easier for Lua coders to interact with. Besides a few expected names the system does not enforce any style on modules loaded through it.

## Installing
Provided are both .lua and .fennel files for plasma core. If you use the fennel files you must also use the fennel support package included in the fso scripters repository.

The Codekeys files included are both an example of how a simple module can be structured, and a way to make use of the live development reload capabilties of the module. This workflow works best with features added during the FSO 22.3 development cycle

## Dependencies
* (optional) Fennel 1.3.0
* Reqver 1.0.0

## Usage
A module designed from plasma core should be a file that returns a table of functions. Fennel convention is to declare all functions as locals and then build a table of all the ones to be exported, but it is equally valid to create the table ahead and create the functions as members of it intially.

Note: You only need to export functions that are going to be called from outside the module, such as the framework functions listed below, or ones that will be used as sexp, hook or override functions. This is required to have them reload properly if changed.

Outside of those cases it is perfectly valid to keep functions local to the module and not export them, so long as the above rules are followed.

If a local function needs to access the module table for some reason and you don't want to pass it in, use get_module_namespace to obtain a safe reference.

### Special member names:
All of these members are optional, but if they exist they will be treated in specific ways.

>*table* state: 

Contain all gameplay state in this table. It is protected from most reloads.

>*table* config: 

A table of all the configuration values set in user files. This is updated on reloads, but otherwise shouldn't change

>*function* initalize: 

Sets up the initial state. Ideally it should also clean up any existing state on reloads, when that state includes things like active engine handles

> *function* configure: 

Reads all configuration files and populates the config table. This should use the load_modular_configs function if possible.
Also set SEXP actions in `configure()`, using plasma core's `add_sexp`, as they can safely be reassigned on reloads.

> *function* hook: 

Creates hooks using plasma core's `add_hook`. `add_hook` creates fully reloadable hooks if used properly, and has support for all the features of -sct.tbm files.
Hook is only ever run the first time a module is loaded, since the engine does not provide a way to remove or replace hooks. Consider using the codekeys module if you need to add a new hook at runtime.
An example of a hook function
```lisp
(fn hook [self core]
  (core.add_hook(self, "clear_all", "On Mission About To End")
  (core.add_hook(self, "message_send", "On Message Received"))
```

or in lua:

```lua
local function hook(self, core)
  core.add_hook(self, "clear_all", "On Mission About To End")
  core.add_hook(self, "message_send", "On Message Received")
end
```

If you put any non-function members in the module anywhere but the state or config tables they may not be loaded or reloaded properly.

## Loading
A standard plasma core module has no `-sct.tbm` file. On game startup the framework scans `data/scripts/plasma_modules` and attempts to load any fennel or lua files it finds there as modules. It can also load files in subfolders of that location.

Then it sets up any hooks within the module's hook() function. However, it is perfectly fine and reasonable to set the hooks up in the sct table instead, provided they follow the structure described in the usage section.

Any code that references a module should use a local populated with `get_module()`, to ensure standard behavior. Internal handling and storage of modules could conceivably change in the future, but get_module should always work as it does, so direct access to the plasma_core modules table is strongly discouraged.

Due both to the intricacies of live reloads and some oddities of the hooks system it is highly recommended you use the automatic loading and `hook()`+`add_hook()` method to set up your module. However if you are set on having a `-sct.tbm` table, this is what the basic boilerplate of it would look like.

```lua
#Conditional Hooks
$Application: FS2_Open
$On Game Init:
[
require 'add_fennel' --omit if using precompilled lua.
local core = require 'plasma_core'
core:get_module('modulename')
]
#end
```


# Modular configs

Modular configs are an attempt to mimic FSO's modular tables. The intent is that a parent mod, like for instance the MVPs, can include a script and configure it for the assets they provide, and then downstream mods can modify or agument just like they would normal tables. This system is a bit of a work in progress

In current form the loader must be provided a function that takes a filename and returns a table, so it is flexible in what file formats it can handle. Loading functions for lua tables and fennel tables are included in the library, though the fennel loader requires the fennel compiler be present. There's no reason a json loader couldn't be built, but I haven't yet. 

The function is passed a prefix and file extension to search for as well, and reads from data/config. Anything FSO will index in that folder can be read.

A value with the key 'priority' in the table controls which config is final in case of overlaps, with higher numbers being the final say. The cfile api functions don't expose the ordering typically provided by mod load order, so order is undefined if priority is not provided.

# Function listing
In accordance with fennel style, anything marked with a ? is an optional parameter that can be nil. Anything else will cause an error if nil. In the lua versions of the library the assertions tied to this have been removed as keeping the line numbers in sync is impractical, and the paramter names have had the ? replaced with "opt"

# General utility functions

## warn_once
> *any* id

> *string* text

> *table* memory

Allows a module to show a warning the first time something happens, using the id to index into the memory table to check and store if it has repeated. The memory table is left to the modules to manage so they can modify it when is appropriate to them.

## print
>*string* message

>*string* ?label

A safe wrapper around the engine print function, prints each on it's own line.

## safe_subtable
> *table* parent

> *string* key

Get a table from inside a table, even if it doesn't exist.

## safe_global_table

> *string* key

Get a global table, even if it doesn't exist.

## recursive_table_print
> *string* tablename

> *table* to_print

> *number* ?depth

Prints a whole table recursively, in a loosely lua table format. Depth should never be passed in and is only for recursion purposes.

## is_value_in
> *table* self

> *any* value

> *table* target

This function shouldn't be used as it is redundant with a core library function and will be removed once I clean up the code that uses it.

## merge_tables_recursive
> *table* self

> *table* source

> *table* target

> *bool* ?replace

> *array* ?ignore

Combines two tables.
Leaves overlapping non-table members alone unless replace is set. Always merges members that are tables.
Ignore takes an array of keys to skip.

## verify_table_keys
> *table* t 

> *table* required 

> *table* optional 

> *string* ?label

Optional label parameter is used for debug output

Checks a table to ensure only a valid set of keys is present and/or enforce a set of required keys.

Useful to guard against typos in config tables.

# FSO interface helpers

## add_order

> *string* name

> *table* host

> *function* enter

> *function* frame

> *function* ?still_valid

> *function* ?can_target

Attaches functions to a LuaAI SEXP's action hooks.
Host is the self of the module that the attached function belongs to. Consequently all functions being used should be ones with a self paramter.

## add_sexp

> *string* name

> *table* host

> *function* action

  Attaches functions to a Lua SEXP's action hook
Host is the self of the module that the attached function belongs to. Consequently all functions being used should be ones with a self paramter.

# Module management functions

## get_module 

> *table* self

> *string* file_name

> *bool* ?reload

> *bool* ?reset

> *table* ?version_spec 

> *bool* ?optional

Gets or loads a module by filename.

If reload is true, it will reload the module's functions and configuration

If reset is true, the module's state will also be reinitalized

If ?version_spec is present, it is passed to reqver as a version specifier table. Otherwise no version checking is done

If ?optional is present and ?version_spec is also passed, this sets reqver's checking to optional mode. Assumed false if omitted.

get_module only attaches a module's hooks on first load, as there is currently no way to replace existing hooks. Modules should design hooks around this limitation. Actions for orders and sexps are re-attached each load as that is a safe replacement.

## get_module_namespace

> *table self

> *string* file_name

Gets access to a module table, even if the module has not yet been loaded.

Intended for letting local functions reference the module state without adding syntactic bloat or issues on reload.

## reload

> *table* self

Reloads the core functions, then reloads all modules.

## load_modular_configs

> *table* self

> *string* prefix 

> *string* ext

> *function* loader  

Builds and returns a table by evaluating files of a given prefix and extension using the loading function.


## config_loader_lua
> *string* file_name 

Loads lua format configuration tables

## config_loader_fennel
> *string* file_name 

Loads fennel format configuration tables