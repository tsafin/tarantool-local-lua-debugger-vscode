# Local Lua Debugger for Visual Studio Code

A simple Lua debugger which requires no additional dependencies.

---
## Notice of Breaking Change
Beginning in version 0.3.0, projects which use sourcemaps to debug code transpiled from another language (such as TypescriptToLua), **must** specify the [`scriptFiles`](#scriptFiles) launch configuration option in order to use breakpoints in the original source files. This allows these to be resolved at startup instead of at runtime which allows for a significant performance increase.

---
## Features
- Debug Lua using stand-alone interpretor or a custom executable
- Supports Lua versions 5.1, 5.2, 5.3 and [LuaJIT](https://luajit.org/)
- Basic debugging features (stepping, inspecting, breakpoints, etc...)
- Conditional breakpoints
- Debug coroutines as separate threads
- Basic support for source maps, such as those generated by [TypescriptToLua](https://typescripttolua.github.io/)

---
## Usage

### Lua Stand-Alone Interpreter
To debug a Lua program using a stand-alone interpreter, set `lua-local.interpreter` in your user or workspace settings:

!["lua-local.interpreter": "lua5.1"](resources/settings.png '"lua-local.interpreter": "lua5.1"')

Alternatively, you can set the interpreter and file to run in `launch.json`:
```json
{
  "configurations": [
    {
      "type": "lua-local",
      "request": "launch",
      "name": "Debug",
      "program": {
        "lua": "lua5.1",
        "file": "main.lua"
      }
    }
  ]
}
```

### Custom Lua Environment
To debug using a custom Lua executable, you must set up your `launch.json` with the name/path of the executable and any additional arguments that may be needed.
```json
{
  "configurations": [
    {
      "type": "lua-local",
      "request": "launch",
      "name": "Debug Custom Executable",
      "program": {
        "command": "executable"
      },
      "args": [
        "${workspaceFolder}"
      ]
    }
  ]
}
```
You must then manually start the debugger in your Lua code:
```lua
require("lldebugger").start()
```
Note that the path to `lldebugger` will automatically be appended to the `LUA_PATH` environment variable, so it can be found by Lua.

---
## Requirements & Limitations
- The Lua environment must support communication via stdio.
  - Some enviroments may require command line options to support this (ex. Corona requires `/no-console` flag)
  - Use of `io.read` or other calls that require user input will cause problems
- The Lua environment must be built with the `debug` library, and no other code should attempt to set debug hooks.
- You cannot manually pause debugging while the program is running.
- In Lua 5.1 and LuaJIT, the main thread cannot be accessed while stopped inside of a coroutine.

---
## Tips
- For convenience, a global reference to the debugger is always stored as `lldebugger`.
- You can detect that the debugger extension is attached by inspecting the environment variable `LOCAL_LUA_DEBUGGER_VSCODE`. This is useful for conditionally starting the debugger in custom environments.
    ```lua
    if os.getenv("LOCAL_LUA_DEBUGGER_VSCODE") == "1" then
      require("lldebugger").start()
    end
    ```
- Some custom environments will not break on uncaught runtime errors. To catch a runtime error, you can wrap the code with `lldebugger.call()`:
    ```lua
    lldebugger.call(function()
    --code causing runtime error
    end)
    ```
- Some environments will not load required files from the standard filesystem. In these cases, you may be able to load the debugger manually using the file path stored in `LOCAL_LUA_DEBUGGER_FILEPATH`:
    ```lua
    package.loaded["lldebugger"] = assert(loadfile(os.getenv("LOCAL_LUA_DEBUGGER_FILEPATH")))()
    require("lldebugger").start()
    ```

---
## Additional Configuration Options
#### `scriptRoots`

A list of alternate paths to find Lua scripts. This is useful for environments like LÖVE, which use custom resolvers to find scripts in other locations than what is in `package.config`.

#### `scriptFiles`

A list of glob patterns identifying where to find Lua scripts in the workspace when debugging. This is required for placing breakpoints in sourcemapped files (ex. 'ts' scripts when using TypescriptToLua), as the source files must be looked up ahead of time so that breakpoints can be resolved.

Example: `scriptFiles: ["**/*.lua"]`

#### `breakInCoroutines`

Break into the debugger when errors occur inside coroutines.
- Coroutines created with `coroutine.wrap` will always break, regardless of this option.
- In Lua 5.1, this will break where the coroutine was resumed and the message will contain the actual location of the error.

#### `stopOnEntry`

Automatically break on first line after debug hook is set.

#### `cwd`

Specify working directory to launch executable in. Default is the project directory.

#### `args`

List of arguments to pass to Lua script or custom environment when launching.

#### `env`

Specify environment variables to set when launching executable.

#### `verbose`

Enable verbose output from debugger. Only useful when trying to identify problems with the debugger itself.

---
## Custom Environment Examples

### LÖVE

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Love",
      "type": "lua-local",
      "request": "launch",
      "program": {
        "command": "love"
      },
      "args": [
        "game"
      ],
      "scriptRoots": [
        "game"
      ]
    }
  ]
}
```
```lua
if os.getenv("LOCAL_LUA_DEBUGGER_VSCODE") == "1" then
  require("lldebugger").start()
end

function love.load()
  ...
```

**Note** that `console` must be set to `false` (the default value) in `conf.lua`, or the debugger will not be able to communicate with the running program.

**game/conf.lua**

```lua
function love.conf(t)
  t.console = false
end
```

### Busted

Note that even when using busted via a lua interpreter, it must be set up as a custom environment to work correctly.

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Busted CLI",
      "type": "lua-local",
      "request": "launch",
      "program": {
        "command": "busted"
      },
      "args": [
        "test/start-cli.lua"
      ]
    },
    {
      "name": "Debug Busted via Lua Interpreter",
      "type": "lua-local",
      "request": "launch",
      "program": {
        "command": "lua"
      },
      "args": [
        "test/start-interpreter.lua"
      ]
    }
  ]
}
```

**test/start-cli.lua**

```lua
if os.getenv("LOCAL_LUA_DEBUGGER_VSCODE") == "1" then
  require("lldebugger").start()
end

describe("a test", function()
  ...
end)
```

**test/start-interpreter.lua**

```lua
--busted should be required before hooking debugger to avoid double-hooking
require("busted.runner")()

if os.getenv("LOCAL_LUA_DEBUGGER_VSCODE") == "1" then
  require("lldebugger").start()
end

describe("a test", function()
  ...
end)
```

### TypescriptToLua (Custom Environment)

```json
{
  "name": "Debug TSTL",
  "type": "lua-local",
  "request": "launch",
  "program": {
    "command": "my_custom_environment"
  },
  "args": [
    // ...
  ],
  "scriptFiles": ["**/*.lua"] // Required for breakpoints in ts files to work
}
```

**tsconfig.json**

```ts
{
  "compilerOptions": {
    "sourceMap": true,
    ...
  },
  "tstl": {
    "noResolvePaths": ["lldebugger"] // Required so TSTL ignores the missing dependency
  }
}
```
