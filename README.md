# go-pcap

This is a native go packet processing library. It performs a function very similar to
[libpcap](https://github.com/the-tcpdump-group/libpcap), or, for that matter, [github.com/google/gopacket/pcap](github.com/google/gopacket/pcap), except that
it is 100% native go. That means that:

* you can build it with `CGO_ENABLED=0`
* cross-compiling is simple

It also include a simple binary called `pcap` that exercises the library by capturing packets and outputting some of the data.

## Operating Systems

The current support is for Linux and macOS/Darwin. Eventually, we will port to other OSes (and happily will take
Pull Requests).

## How To Use It

### Library
The library has single primary entrypoint, [pcap.OpenLive](https://godoc.org/github.com/packetcap/go-pcap#OpenLive).
This will return a [pcap.Handle](https://godoc.org/github.com/packetcap/go-pcap#Handle) that can be used to loop
and read packets.

```go
if handle, err = pcap.OpenLive(iface, 1600, true, 0); err != nil {
	log.Fatal(err)
}
for {
	data, captureInfo, error := handle.ReadPacketData()
}
```

`ReadPacketData()` will block until there is packet information available, or until `timeout` is reached. You can set an infinite timeout with `0`.

The returned information will be the packet bytes themselves, excluding the system-defined headers, i.e. the Ethernet frame and all contents.

`Handle` is 100% compatible with [gopacket.Handle](https://godoc.org/github.com/google/gopacket#Handle); you can use it to process packets, analyze layers,
and anything else you would want.

```go
import (
	pcap "github.com/packetcap/go-pcap"
	"github.com/google/gopacket"
)

if handle, err = pcap.OpenLive(iface, 1600, true, 0); err != nil {
        log.Fatal(err)
}
packetSource := gopacket.NewPacketSource(handle, layers.LinkTypeEthernet)
for packet := range packetSource.Packets() {
        processPacket(packet)
}
```

If you know you don't want all of that overhead, you can use the [Listen](https://godoc.org/github.com/packetcap/go-pcap#Listen) interface,
which returns a `chan` to which it will send packets.


```go
// c is a channel of type pcap.Packet
if c, err = pcap.Listen(iface, 65536, true, true, 0); err != nil {
        log.Fatal(err)
}
for packet := range c {
        processPacket(packet.B)
}
```

`pcap.Listen` will start a separate goroutine, so you do not have to. `pcap.Listen` is a one-shot, "open a socket, listen for packets, send
them down my channel" convenience.

#### Efficiency

The Linux implementation supports both syscall-based packet reads and mmap-based packet reads. The syscall read is fine for just a few packets, or a lightly loaded
system. However, making a syscall to retrieve each packet can get very slow, very quickly. For faster purposes, you can use a shared mmap buffer with the kernel.

The `OpenLive()` call uses mmap by default.

### CLI

There is a sample command-line utility included. To build it:

```sh
$ make build
```

Or for an alternate OS:

```sh
$ make build OS=linux
```

The binaries will be output as `dist/pcap-<os>-<arch>`. Additionally, if you are building for your local OS+arch, a binary named `pcap` will be
deposited in the current directory, so you can just do `./pcap`.

For options, run `./pcap --help`.

## TODO

* add BPF filter support, so we can do `"pcap udp and port 23"`
* more efficient packet retrieval support on macOS
