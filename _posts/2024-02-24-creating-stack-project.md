<!--                             LINE LENGTH: 80                             -->
# Creating a Stack Project
This guide was originally written as a Discord message for students. It's not a
complete guide - *nor a very good one* - but it serves to just introduce the
concept and allow students to figure the rest out.
## Generating the Project
To create a Stack project, `cd` into the location you want to *create the
folder* and do
```console
stack new <projectname> new-template
```
`new-template` is the default template, but I'm adding it here just in case
(older versions have a different one). Example:
`stack new minihaskell new-template`
This will create a folder in that location called <projectname> (minihaskell
in the example). This is the folder of your workspace - so open this up in your
editor/IDE.
## Configuring the Stack Project
The most important file is package.yaml. This is where you configure your
project.
### Dependencies
The `dependencies` section near the top of the file is where you list external
packages that your project requires.
Let's say I want to use [`Data.Map`](
https://hackage.haskell.org/package/containers-0.4.0.0/docs/Data-Map.html) in
my project. Hackage says that it comes only from `containers`. I also want to
use [`Data.Array`](
https://hackage.haskell.org/package/array-0.5.6.0/docs/Data-Array.html),
which says (at the top) it comes from `array`. So my `dependencies` section
would become:
```yaml
dependencies:
- base >= 4.7 && < 5
- array
- containers
```
(or whatever version it says your base is - just leave that line in there).
You may come across errors when attempting to import certain things in your
project, for example:
```console
error:
    Could not load module 'Data.Array'
    It is a member of the hidden package 'array-0.5.6.0'.
    Perhaps you need to add 'array' to the build-depends in your .cabal file.
```
**Do not add this to your .cabal file** - add `array` to the `dependencies`
section in package.yaml.
### Build Tools
To add build tools (alex and happy), locate the
`executables -> <projectname>-exe` section in your package.yaml. In here, add a
section called `build-tool-depends`. Each item in `build-tool-depends` has the
following layout:
`package:executable`
For alex and happy, this is simply `alex:alex` and `happy:happy` respectively.
Stack already knows what these are and will source them for you. If these are
the only two build tools you need, then this will look like:
```yaml
executables:
  <packagename>-exe:
    ...
    build-tool-depends:
    - alex:alex
    - happy:happy
```
### Other info
Feel free to edit other info like the version, author(s), etc. at the top of
the file.
**Do not edit the .cabal file**. When you build (compile) your project, Stack
will automatically edit this with all the info you need.
## Building the project
### Alex and Happy
To reiterate, ***you should not run alex and/or happy manually***. This can
mess up your Stack project, as Stack will then use the generated .hs file
instead of running Alex itself (meaning if you make changes, you'll have to
manually run Alex each time). Also, Stack will not create the .hs (it does
this internally temporarily to compile the project).
### Compiling
To build the project, ensure you are in the project directory in your terminal
(such that doing `ls` will list `package.yaml`), and run
```console
stack build
```
This will automatically run Alex, Happy, and compile your project. This is
where you will see any errors you have made in your code.
## Running the Project
To run the project, you need to use a specific command:
```console
stack exec -- <projectname>-exe [args]
```
The `--` and the space following are **essential**. For example:
`stack exec -- minihaskell-exe program1.mhs`
### Locating the executable
If you wish to find the actual executable file Haskell has made (.exe on
Windows), run:
```console
stack exec -- whereis <projectname>-exe
```
and this'll output the location.
