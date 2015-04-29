# go-fd-mutex

This fd-mutex is code extracted from the go/net package:
- http://golang.org/src/pkg/net/fd_mutex.go
- http://golang.org/src/pkg/net/fd_mutex_test.go

It's great, and designed specifically for low-level IO in Go. It's used by `netFD` (http://golang.org/src/pkg/net/fd_unix.go), the thing all `net.Conn` implementations use underneath, to manage their sockets.

I want to use it in [go-udtwrapper](https://github.com/jbenet/go-udtwrapper/udt), and other net protocol implementations. But, sadly, I can't use it because it relies on these odd lines:

```Go
// Implemented in runtime package.
func runtime_Semacquire(sema *uint32)
func runtime_Semrelease(sema *uint32)
```

These are functions without a body. But it turns out that they get special-cased. They are satisfied by the semaphore implementation in http://golang.org/src/pkg/runtime/sema.goc. Go patches it weirdly. This is C code, with a middle-dot `·` character as a var name delimiter, to mimic the `.` pkg notation (good idea).

```Go
// TODO(dvyukov): move to netpoll.goc once it's used by all OSes.
void net·runtime_Semacquire(uint32 *addr)
{
  runtime·semacquire(addr, true);
}

void net·runtime_Semrelease(uint32 *addr)
{
  runtime·semrelease(addr);
}
```

And turns out that this C code with `.goc` extension is compiled using the `dist` tool, which includes http://golang.org/src/cmd/dist/goc2c.c BUT! it will only compile such `.goc` code if it exists in the Go runtime package:

From: http://golang.org/src/cmd/dist/build.c#L934
```Go
// Generate .c files from .goc files.
if(streq(dir, "pkg/runtime")) {
  for(i=0; i<files.len; i++) {
    p = files.p[i];
    if(!hassuffix(p, ".goc"))
      continue;
    // b = path/zp but with _goos_goarch.c instead of .goc
    bprintf(&b, "%s%sz%s", bstr(&path), slash, lastelem(p));
    b.len -= 4;
    bwritef(&b, "_%s_%s.c", goos, goarch);
    goc2c(p, bstr(&b));
    vadd(&files, bstr(&b));
  }
  vuniq(&files);
}
```

After finding all this out, trudging through the source, I gave up. If the go team is trying so hard to keep all this private and unused by others, it's likely that it makes lots of assumption I'd break by ripping it out and putting it elsewhere.

This repo is up here for posterity, and just in case I change my mind again.
