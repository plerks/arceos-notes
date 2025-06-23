[arceos仓库地址](https://github.com/arceos-org/arceos)

主要想知道是怎么和硬件设备交流的，accept()肯定不是忙等待的，是怎么实现的？

## 如何运行httpserver
运行命令为: `make run A=examples/httpserver NET=y`

[编译qemu](https://www.qemu.org/download/)时，要配置打开user网络后端：
```shell
make clean

./configure   --target-list=x86_64-softmmu,riscv64-softmmu,aarch64-softmmu,loongarch64-softmmu   --enable-user   --enable-slirp

make
```

## arceos网络的accept()函数是如何实现的
accept()函数是否是忙等待的？

否

api/arceos_posix_api/src/imp/net.rs sys_accept() 调用了 api/arceos_posix_api/src/imp/net.rs Socket.accept()，然后调用了 modules/axnet/src/smoltcp_impl/tcp.rs TcpSocket.accept()

```Rust
// modules/axnet/src/smoltcp_impl/tcp.rs

pub fn accept(&self) -> AxResult<TcpSocket> {
    if !self.is_listening() {
        return ax_err!(InvalidInput, "socket accept() failed: not listen");
    }

    // SAFETY: `self.local_addr` should be initialized after `bind()`.
    let local_port = unsafe { self.local_addr.get().read().port };
    self.block_on(|| {
        let (handle, (local_addr, peer_addr)) = LISTEN_TABLE.accept(local_port)?;
        debug!("TCP socket accepted a new connection {}", peer_addr);
        Ok(TcpSocket::new_connected(handle, local_addr, peer_addr))
    })
}
```
这个函数的实现为：
```Rust
/// Block the current thread until the given function completes or fails.
///
/// If the socket is non-blocking, it calls the function once and returns
/// immediately. Otherwise, it may call the function multiple times if it
/// returns [`Err(WouldBlock)`](AxError::WouldBlock).
fn block_on<F, T>(&self, mut f: F) -> AxResult<T>
where
    F: FnMut() -> AxResult<T>,
{
    if self.is_nonblocking() {
        f()
    } else {
        loop {
            SOCKET_SET.poll_interfaces();
            match f() {
                Ok(t) => return Ok(t),
                Err(AxError::WouldBlock) => axtask::yield_now(),
                Err(e) => return Err(e),
            }
        }
    }
}
```
会去调yield_now()，不是busy loop的。

SocketSetWrapper.poll_interfaces()函数：
```Rust
// modules/axnet/src/smoltcp_impl/mod.rs

pub fn poll_interfaces(&self) {
    ETH0.poll(&self.0);
}
```

InterfaceWrapper.poll()函数：
```Rust
// modules/axnet/src/smoltcp_impl/mod.rs

pub fn poll(&self, sockets: &Mutex<SocketSet>) {
    let mut dev = self.dev.lock();
    let mut iface = self.iface.lock();
    let mut sockets = sockets.lock();
    let timestamp = Self::current_time();
    iface.poll(timestamp, dev.deref_mut(), &mut sockets);
}
```

Interface.poll()函数：
```Rust
// modules/axnet/src/smoltcp_impl/mod.rs

/// Transmit packets queued in the given sockets, and receive packets queued
/// in the device.
///
/// This function returns a boolean value indicating whether any packets were
/// processed or emitted, and thus, whether the readiness of any socket might
/// have changed.
pub fn poll<D>(
    &mut self,
    timestamp: Instant,
    device: &mut D,
    sockets: &mut SocketSet<'_>,
) -> bool
where
    D: Device + ?Sized,
{
    self.inner.now = timestamp;

    #[cfg(feature = "_proto-fragmentation")]
    self.fragments.assembler.remove_expired(timestamp);

    match self.inner.caps.medium {
        #[cfg(feature = "medium-ieee802154")]
        Medium::Ieee802154 =>
        {
            #[cfg(feature = "proto-sixlowpan-fragmentation")]
            if self.sixlowpan_egress(device) {
                return true;
            }
        }
        #[cfg(any(feature = "medium-ethernet", feature = "medium-ip"))]
        _ =>
        {
            #[cfg(feature = "proto-ipv4-fragmentation")]
            if self.ipv4_egress(device) {
                return true;
            }
        }
    }

    let mut readiness_may_have_changed = false;

    loop {
        let mut did_something = false;
        did_something |= self.socket_ingress(device, sockets);
        did_something |= self.socket_egress(device, sockets);

        #[cfg(feature = "proto-igmp")]
        {
            did_something |= self.igmp_egress(device);
        }

        if did_something {
            readiness_may_have_changed = true;
        } else {
            break;
        }
    }

    readiness_may_have_changed
}
```

### 如何与硬件交流的？
看下来都是在Rust语言内部的，没看到具体怎么和设备交流的，应该是在axdriver里，后面再看。

可以问这个ai：[deepwiki](https://deepwiki.com/arceos-org/arceos)