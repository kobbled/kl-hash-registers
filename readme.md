# Hash Registers

This repository uses a hash table for managing register sets on FANUC robot controllers.

## Creating a Karel environment hash from TP+ environment

In the `robot.ini` file define a file as your environment file

```
[WinOLPC_Util]
Env=C:\path\to\env.tpp
```

Then in the `package.json` file define the key/value below:

```
"tpp_compile_env" : {"name" : "<environment-hash-filename>", "clear" : true, "config" : "<karel-enum-filename>"}
```
where `<environment-hash-filename>` is the name of the karel file to store the hash of the evironment variables, and `<karel-enum-filename>` is the **.klt** file to store an enum macro list of registers for use when programming in karel.

If `"clear" : true` all of the register names and values will be cleared on the controller. This can be disabled by setting `"clear" : false`.

Build the package with rossum
```
rossum -w -o
ninja
```

This will output two files: *\<environment-hash-filename\>.kl*, and *\<karel-enum-filename\>.kl*. Move these two files into another repository, and compile the *\<environment-hash-filename\>.kl* file to `.pc`. Run this file on the controller to reset all of your registers.

> [!**NOTE**]
> It is best to delete this file after use on the controller to conserve memory. Only load it on when wanting to reset register names/values.


