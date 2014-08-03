# Stick: A Language for Statically Typed Filesystem Manipulation
Stick is a structurally typed, statically checked language that can be used to operate on filesystems. Stick's type system is implemented via a plugin system itself. While the language is statically typed, its type system is dependent on system state. Path references, like `/usr/bin`, are bound statically to the program's behavior. 

## Justification and Example

Today's POSIX-based filesystems are confusing, sprawling domains that lead system maintainers to the drink. How many times have you run into problems where:

* A file you expected to be present wasn't there.
* A file you expected to be there wasn't executable by you.
* A symlink that used to resolve doesn't anymore.
* An executable you expect to be on the $PATH isn't.
* A popular language's interpreter binary is the wrong version for the script you're running. 
* A file you expect to have a particular format is corrupted / of a different format.
* A process you expect to generate a file doesn't.

... and many more similar issues.

Stick attempts to solve this problem by attacking the underlying problems in filesystem management and creating a rigidly defined, statically typed domain for manipulating the filesystem.

Here are two code samples for curling and unpacking a tar, generating some conf files and executing a process. The first is a `bash` script, the second is a Stick script.

```bash
set -ve
curl https://mybin.github.io/downloads/1.2.tar.gz > /tmp/mybin-12.tar.gz
tar -xf /tmp/mybin-1.2.tar.gz -C /var/lib/mybin
cp /var/lib/mybin/conf/mybin.cnd.sample /etc/mybin.cnf
mkdir -p /var/service/mybin/run
cp /var/lib/mybin/bin/run /var/service/mybin/run
runsv mybin
```

Here is that same script run using Stick
```
import "github.com/anthonybishopric/mybin"
import "github.com/stick/runit"
curl.get("https://mybin.github.io/downloads/1.2.tar.gz") > mybin.Tar(/tmp/mybin-12.tar.gz)
/var/lib/mybin := /tmp/mybin-12.tar.gz -> extract
cp /var/lib/mybin/conf/mybin.cnd.sample /etc/mybin.cnf
/var/service/mybin := /var/service -> mkstruct
cp /var/lib/mybin/bin/run /var/service/mybin/run
path/runsv { mybin }
```

Both scripts achieve the same thing, but unlike the `bash` script, the Stick script will validate many things, many of which occur _before even running_:

* All uninitialized path references must exist prior to running the program. In this example, `/tmp`, `/etc`, `/var`, `/var/service`, and `/var/lib` must all be valid objects before the program may run. Stick considers the filesystem to be part of its program scope.
* All operations on uninitialized references must be valid prior to the program running. For example, `Stick` will ensure that the path `/var/service` is of a type that can respond to the operation `mkstruct`. (`Stick` is able to achieve this by bootstrapping your existing filesystem with metadata that assign pre-known types - see [Pre-Compilation](#pre-compilation) for more details)
* * Paths initialized with `:=` during the script must not exist prior to the start of the script.
* The `import` builtin allows custom-packages. The `runit` package is written to support [Runit](http://smarden.org/runit/) and enforces that programs using Stick to write runit configurations are valid.
* The `mybin` package defines several types and operations that are used throughout this script. For instance, `extract`, which is a member of the `mybin.Tar` type. Implicitly, the path `/var/lib/mybin` takes on the type `mybin.Pack`. Stick makes heavy use of structural typing to make scripting as straightforward as writing a script in Ruby, Bash or Go, while achieving a greater degree of safety. 

## Types

Stick's base type system has the following properties:

* All objects have exactly one type.
* All object assignments are type-checked statically.
* Each type defines its own rules when performing casts.

Types are defined as structures that can have properties, operations, and casters. Types are bound to paths at compilation-time. An object without a type is considered 'unbound' and cannot be assigned to a path until a type has been assigned.

Lets create a naive Stick package that defines a `BinDir` type, to reflect the general structure of directories containing executables. This example is primarily for demonstration purposes; we can use more consise syntax to do the same thing, which will be shown later.

```
$ cat github.com/anthonybishopric/mytypes/bindir.st
type BinDir
  def run(child string, argv *string)
    return Set(self.path).get(child).exec *argv
  end

  cast Set(set)
    set.each do |inner|
        fail("{} is not executable" % inner.path) unless inner.path.executable
    end
  end

  cast Object(obj)
    fail("{} is not a Set" % obj.path)
  end
end

```

This type can now be bound to paths on your filesystem:
```
$ cat hello_bin.st
import "github.com/anthonybishopric/mytypes"
bin := mytypes.BinDir(/usr/local/bin)
bin -> run("ls", "-l" , "/var/lib")
```

At compilation, the `hello_bin.st` script will import our custom `BinDir` type, identify the unitialized path `/usr/local/bin`, infer the most specific type for the path it can, and call the corresponding `cast` function on the `BinDir` type. Assuming that `/usr/local/bin` is a directory, the `cast` function used will be `cast Set(set)`. The variable `bin` is not a relative path, but rather a variable pointer that is in scope in the program that points to the `BinDir` instance. In Stick, paths are always absolute and must always start with `/` or can be expanded by an object (in this case, `bin`).

While we can now be confident that `/usr/local/bin` is a directory that contains only executables, we have a different problem. The definition of `run()` takes a string and assumes that the string refers to an existing binary. We can do better. 

At compilation time, Stick knows the contents of the `/usr/local/bin` directory and makes the assumption that no other files are created or destroyed except during the run of the Stick program itself. This assumption, while constraining, lets us make guarantees about the program's correctness. (See [Filesystem Assumptions](#filesystem-assumptions) for more information)

```
$ cat github.com/anthonybishopric/mytypes/bindir.st
type BinDir : Set[Executable]
end

```

Above, we've omitted the `Object` and `Set` cast operations as they are already provided by the `Set[Executable]` parent class. The parameterized type `Executable` allows lookups on the set to be immediately executed. Our script can now be written this way:

```
$ cat hello_bin.st
import "github.com/anthonybishopric/mytypes"
bin := mytypes.BinDir(/usr/local/bin)
bin/ls { -l /var/lib }
```

Now we do not need a `run` function either. Stick allows direct execution of paths that are guaranteed to be `Executable`. Execution options are passed inside of curly braces. At compile-time, Stick can guarantee that `/usr/local/bin/ls` exists and is executable.

If we wanted to go further and ensure that callers of ls are actually using the right options, Stick lets us write custom types that define `Executable`s by their options. If APIs can have specifications, there's no reason why executables can't either.

```
type Ls : Executable
  option { -l }
  required Set
end
```

We define the type `Ls` as an `Executable` that has one one option `-l` followed by a required path bound to a valid `Set`. This syntax can significantly constrain the ways programs can be invoked, but also means that we can statically check the correctness of a command at compile-time. We now modify our definition of `BinDir` to use the `Ls` type.

```
type BinDir : Set[Executable]
  member ls Ls
end
```

When binding `BinDir` to `/usr/local/bin`, now not only will the compiler check that every file inside the directory is executable (thanks to `Set[Executable]`'s ')

The types `Bin` and `Ls` are actually Stick builtins and are available in the `_` global namespace.

## Running Stick Programs

Stick relies heavily on system state to do its compile-time safety checks. A single Stick program is divided into two bits:

1. The scripts that mutate the filesystem in desirable ways.
2. The underlying types that define valid structures and transformations described in Stick scripts.

To optimally use Stick, you might run `/usr/local/bin/stickd`, passing a [typepath](#typepath) that will be used to load all known Stick types. Scripts are then submitted to be run by `stickd` by the `stick` CLI. Running stick is not unlike submitting programs to a running virtual machine for processing. 

## Implementation

So far this document is just a brainstorm, but it seems like something needs to be done to make filesystem management not just about aggregating thousands of commands into a monolithic configuration management system (*cough cough puppet*)

At the moment, I'm considering Rust as the implementation language of choice. The main reason to use Rust is to be able to easily build a FUSE frontend to `stickd` that can assign Stick correctness checks to ordinary Bash scripts. 