bambam: auto-generate capnproto schema from your golang source files.
======

Adding [capnproto serialization](https://github.com/glycerine/go-capnproto) to an existing Go project used to mean writing alot of boilerplate.

Not anymore.

Given a set of golang (Go) source files, bambam will generate a [capnproto](http://kentonv.github.io/capnproto/) schema. Even better: bambam will also generate translation functions to readily convert between your golang structs and the new capnproto structs.

prereqs
-------

You'll need a recent (up-to-date) version of go-capnproto. If you installed go-capnproto before, you'll want to update it [>= f9f239fc7f5ad9611cf4e88b10080a4b47c3951d  / 16 Nov 2014].

[Capnproto](http://kentonv.github.io/capnproto/) and [go-capnproto](https://github.com/glycerine/go-capnproto) should both be installed and on your PATH.

to install
--------
~~~
# be sure go-capnproto and capnpc are installed first.

$ go get -t github.com/glycerine/bambam  # the -t pulls in the test dependencies.

# ignore the initial compile error about 'undefined: LASTGITCOMMITHASH'. `make` will fix that.
$ cd $GOPATH/src/github.com/glycerine/bambam
$ make  # runs tests, build if all successful
$ go install
~~~


use
---------

~~~
use: bambam -o outdir -p package myGoSourceFile.go myGoSourceFile2.go ...
     # Bambam makes it easy to use Capnproto serialization[1] from Go.
     # Bambam reads .go files and writes a .capnp schema and Go bindings.
     # options:
     #   -o="odir" specifies the directory to write to (created if need be).
     #   -p="main" specifies the package header to write (e.g. main, mypkg).
     #   -X exports private fields of Go structs. Default only maps public fields.
     #   -version   shows build version with git commit hash
     #   -OVERWRITE modify .go files in-place, adding capid tags (write to -o dir by default).
     # required: at least one .go source file for struct definitions. Must be last, after options.
     #
     # [1] https://github.com/glycerine/go-capnproto 
~~~

demo
-----

See rw.go.txt. To see all the files compiled together in one project: (a) comment out the defer in the rw_test.go file; (b) run `go test`; (c) then `cd testdir_*` and look at the sample project files there. (d). run `go build` in the testdir_ to rebuild the binary. Notice that you will need all three .go files to successfully build: 

~~~
rw.go  schema.capnp.go  translateCapn.go
~~~

example:

~~~
jaten@c03:~/go/src/github.com/glycerine/bambam:master$ cd testdir_884497362/
jaten@c03:~/go/src/github.com/glycerine/bambam/testdir_884497362:master$ go build
jaten@c03:~/go/src/github.com/glycerine/bambam/testdir_884497362:master$ ls
go.capnp  rw.go  rw.go.txt  schema.capnp  schema.capnp.go  testdir_884497362  translateCapn.go
jaten@c03:~/go/src/github.com/glycerine/bambam/testdir_884497362:master$ ./testdir_884497362
Load() data matched Saved() data.
jaten@c03:~/go/src/github.com/glycerine/bambam/testdir_884497362:master$ # run was successful
~~~

Here is what it looks like to use the Save()/Load() methods. You end up with a Save() and Load() function for each of your structs. Simple.

~~~
package main

import (
    "bytes"
)

//
// By default bambam will add the `capid` tags
// to a copy of your source in the output directory.
// Use bambam -OVERWRITE to modify files directly in-place.
// The capid tags control the @0, @1, field numbering 
// in the generated capnproto schema. If you change
// your go structs, the capid tags let your schema
// stay backwards compatible with prior serializations.
//
type MyStruct struct {
	Hello    []string  `capid:"0"`
	World    []int     `capid:"1"`
}

func main() {

	rw := MyStruct{
		Hello:    []string{"one", "two", "three"},
		World:    []int{1, 2, 3},
	}

    // any io.ReadWriter will work here (os.File, etc)
	var o bytes.Buffer

	rw.Save(&o)
    // now we have saved!


    rw2 := &MyStruct{}
	rw2.Load(&o)
    // now we have restored!

}

~~~

what Go types does bambam recognize?
----------------------------------------

Supported: structs, slices, and primitve/scalar types are supported. Structs that contain structs are supported. You have both slices of scalars (e.g. `[]int`) and slices of structs (e.g. `[]MyStruct`) available.

We handle `[][]T`, but not `[][][]T`, where `T` is a struct or primitive type. The need for triply nested slices is expected to be rare. Interpose a struct after two slices if you need to go deeper.

Currently unsupported (pull requests welcome): Go maps.  

Also: pointers to structs to be serialized work, but pointers in the inner-most struct do not. This is not a big limitation, as it is rarely meaningful to pass a pointer value to a different process.


capid tags on go structs
--------------------------

When you run `bambam`, it will generate a modified copy of your go source files in the output directory.

These new versions include capid tags on all public fields of structs. You should inspect the copy of the source file in the output directory, and then replace your original source with the tagged version.  You can also manually add capid tags to fields, if you need to manually specify a field number (e.g. you are matching an pre-existing capnproto definition).

If you are feeling especially bold, `bambam -OVERWRITE my.go` will replace my.go with the capid tagged version. For safety, only do this on backed-up and version controlled source files.

By default only public fields (with Captial first letter in their name) are tagged. The -X flag ignores the public/private distinction, and tags all fields.

The capid tags allow the capnproto schema evolution to function properly as you add new fields to structs. If you don't include the capid tags, your serialization code won't be backwards compatible as you change your structs.

Deleting fields from your go structs isn't (currently) particularly well-supported. We could potentially allow fields to be // commented out in the go source and yet still parse the comments and use that parse to keep the schema correct, but that's not a trivial bit of work.

example of capid annotion use
~~~
type Job struct { 
   C int `capid:"2"`  // we added C later, thus it is numbered higher.
   A int `capid:"0"`
   B int `capid:"1"` 
}
~~~

other tags
----------

Also available tags: `capid:"skip"` or `capid:"-1"` (any negative number): this field will be skipped and not serialized or written to the schema.

~~~
// capname:"Counter"
type number struct {
   A int
}
~~~

The above struct will be mapped into capnproto as:

~~~
struct Counter {
  a @0: Int64;
}
~~~

Without the `// capname:"Counter"` comment, you would get:

~~~
struct NumberCapn {
  a @0: Int64;
}
~~~

Explanation: Using a `// capname:"newName"` comment on the line right before a struct definition will cause `bambam` to use 'newName' as the name for the corresponding struct in the capnproto schema. Otherwise the corresponding struct will simply uppercase the first letter of the orignal Go struct, and append "Capn". For example: a Go struct called `number` would induce a parallel generated capnp struct called `NumberCapn`.

special ignored field names
---------------------------
`XXX_unrecognized` fields are currently ignored by default, to make the transition from other systems easier.

-----
-----

Copyright (c) 2014, Jason E. Aten, Ph.D.

