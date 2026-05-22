# Common API

## Description

## API Schema

```
namespace common


abstraction Page<$$T> {
    @@immutable data: list<$$T>
    @@immutable size: int32
    @@immutable pageIndex: int32

    bool hasNext()
    bool isFirst()

    @@async
    @@throws(mirror-node-error)
    Page<$$T> next()

    @@async
    @@throws(mirror-node-error)
    Page<$$T> first()
}

```

## Questions & Comments
