---
theme : "night"
transition: "fade"
highlightTheme: "dracula"
---
## Go Traceroute Clone
Francesco Farina - 
11/10/2018

---

## Traceroute

 - Network diagnostic tool 
 - Display route and transit delays of packets

--

### How does it work?

 - RTT from successive hosts

 <span class="fragment">![Traceroute](https://serverdensity-wpengine.netdna-ssl.com/wp-content/uploads/2017/02/traceroute.png)</span>

--

### Implementations

 - TCP => SYN
 - UDP => Datagram
 - ICMP => Echo request
 - TTL, aka _hop limit_

---

## My Traceroute clone
 
 - Go
 - ICMP traceroute

<span class="fragment">![icmptraceroute](https://telconotes.files.wordpress.com/2013/02/traceroute3.png)</span>

--

### Network library
`golang.org/x/net/`

>These packages are part of the Go Project but outside the main Go tree.

>They are developed under looser compatibility requirements than the Go core. In general, they will support the previous two releases and tip.

<small>https://github.com/golang/go/wiki/SubRepositories</small>

--

### Setup

Resolve the IP Address
```go
ipaddr, err := net.ResolveIPAddr("ip4", address)
```
Listen for incoming ICMP packets
```go
c, err := net.ListenPacket("ip4:icmp", "0.0.0.0")
``` 
Create a new packet connection using `x/net/ipv4.NewPacketConn`
```go
pc := ipv4.NewPacketConn(c)
```

--

### Main Loop
```go
for i := 1; i <= maxTTL; i++ {
    pc.SetTTL(i) // <---- x/net/ipv4
    sendPacket(pc, ipaddr)
    if readPacket(pc) {
        break
    }
}
```

--

### Send Packet

```go
func sendPacket(pc *ipv4.PacketConn, ipaddress *net.IPAddr) {
	p, err := craftPacket()
	checkError(err)
	_, err = pc.WriteTo(p, nil, ipaddress)
	checkError(err)
}
```

--

### Craft a packet
```go
func craftPacket() ([]byte, error) {
	wm := icmp.Message{
		Type: ipv4.ICMPTypeEcho, Code: 0,
		Body: &icmp.Echo{
			ID: os.Getpid() & 0xffff, Seq: 1,
			Data: []byte("GoFF"),
		},
	}
	wb, err := wm.Marshal(nil)
	return wb, err
}
```

--

### Read Packet

```go
func readPacket(pc *ipv4.PacketConn, out chan string) bool {
	buff := make([]byte, 256)
	pc.SetReadDeadline(time.Now().Add(readTimeoutSec * time.Second))
	n, _, peer, err := pc.ReadFrom(buff)
	if err != nil {
		fmt.Printf("Request %ds Timeout\n", readTimeoutSec)
		return false
    }
    
	m, err := icmp.ParseMessage(ipv4.ICMPTypeEcho.Protocol(), buff[:n])
	checkError(err)
	ttl, err := pc.TTL()
    checkError(err)
    
	switch m.Type {
	case ipv4.ICMPTypeTimeExceeded: // hop
		logHop(ttl, peer, m, out)
	case ipv4.ICMPTypeDestinationUnreachable:
		out <- fmt.Sprintf("\t-:\t%v\t\t[%d bytes]\tDESTINATION UNREACHABLE\n", peer, m.Body.Len(1))
	case ipv4.ICMPTypeEchoReply: // destination
		logHop(ttl, peer, m, out)
		return true
	default:
		out <- fmt.Sprintf("%v: %+v", peer, m)
	}
	return false
}
```

---

![live demo](https://memecrunch.com/meme/BGALW/live-demo/image.gif)

---

### Why sudo?

- Raw socket creation

--

### Why sudo?

- `setcap CAP_NET_RAW` on Linux

---

### Further improvements
- TCP or UDP implementation
- https://github.com/indiependente/gotr

---

### Thanks!
