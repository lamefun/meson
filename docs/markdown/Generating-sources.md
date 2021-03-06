---
short-description: Generation of source files before compilation
...

# Generating sources

  Sometimes source files need to be preprocessed before they are passed to the actual compiler. As an example you might want build an IDL compiler and then run some files through that to generate actual source files. In Meson this is done with [`generator()`](https://github.com/mesonbuild/meson/wiki/Reference-manual#generator) or [`custom_target()`](https://github.com/mesonbuild/meson/wiki/Reference-manual#custom_target).

## Using custom_target()

Let's say you have a build target that must be built using sources generated by a compiler. The compiler can either be a built target:

```meson
mycomp = executable('mycompiler', 'compiler.c')
```

Or an external program provided by the system, or script inside the source tree:

```meson
mycomp = find_program('mycompiler')
```

Custom targets can take zero or more input files and use them to generate one or more output files. Using a custom target, you can run this compiler at build time to generate the sources:

```meson
gen_src = custom_target('gen-output',
                        input : ['somefile1.c', 'file2.c'],
                        output : ['out.c', 'out.h'],
                        command : [mycomp, '@INPUT@',
                                   '--c-out', '@OUTPUT0@',
                                   '--h-out', '@OUTPUT1@'])
```

The `@INPUT@` there will be transformed to `'somefile1.c' 'file2.c'`. Just like the output, you can also refer to each input file individually by index.

Then you just put that in your program and you're done.

```meson
executable('program', 'main.c', gen_src)
```

## Using generator()

Generators are similar to custom targets, except that we define a *generator*, which defines how to transform an input file into one or more output files, and then use that on as many input files as we want.

Note that generators should only be used for outputs that will only be used as inputs for a build target or a custom target. When you use the processed output of a generator in multiple targets, the generator will be run multiple times to create outputs for each target. Each output will be created in a target-private directory `@BUILD_DIR@`.

If you want to generate files for general purposes such as for generating headers to be used by several sources, or data that will be installed, and so on, use a [`custom_target()`](https://github.com/mesonbuild/meson/wiki/Reference-manual#custom_target) instead.


```meson
gen = generator(mycomp,
                output  : '@BASENAME@.c',
                arguments : ['@INPUT@', '@OUTPUT@'])
```

The first argument is the executable file to run. The next file specifies a name generation rule. It specifies how to build the output file name for a given input name. `@BASENAME@` is a placeholder for the input file name without preceding path or suffix (if any). So if the input file name were `some/path/filename.idl`, then the output name would be `filename.c`. You can also use `@PLAINNAME@`, which preserves the suffix which would result in a file called `filename.idl.c`. The last line specifies the command line arguments to pass to the executable. `@INPUT@` and `@OUTPUT@` are placeholders for the input and output files, respectively, and will be automatically filled in by Meson. If your rule produces multiple output files and you need to pass them to the command line, append the location to the output holder like this: `@OUTPUT0@`, `@OUTPUT1@` and so on.

With this rule specified we can generate source files and add them to a target.

```meson
gen_src = gen.process('input1.idl', 'input2.idl')
executable('program', 'main.c', gen_src)
```

Generators can also generate multiple output files with unknown names:

```meson
gen2 = generator(someprog,
                 outputs : ['@BASENAME@.c', '@BASENAME@.h'],
                 arguments : ['--out_dir=@BUILD_DIR@', '@INPUT@'])
```

In this case you can not use the plain `@OUTPUT@` variable, as it would be ambiguous. This program only needs to know the output directory, it will generate the file names by itself.
