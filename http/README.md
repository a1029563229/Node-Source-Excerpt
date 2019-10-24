# http 模块

`http` 模块位于 `/lib/http.js`，先看我们经常使用的一个方法是 `createServer`

```js
function createServer(opts, requestListener) {
  return new Server(opts, requestListener);
}
```

`createServer` 的作用是创建了一个 `Server` 类，该文件位于 `/lib/_http_server.js`

```js
function Server(options, requestListener) {
  if (!(this instanceof Server)) return new Server(options, requestListener);

  if (typeof options === 'function') {
    requestListener = options;
    options = {};
  } else if (options == null || typeof options === 'object') {
    options = { ...options };
  } else {
    throw new ERR_INVALID_ARG_TYPE('options', 'object', options);
  }

  // options 允许传两个参数，分别是 IncomingMessage 和 ServerResponse
  // IncomingMessage 是客户端请求的信息，ServerResponse 是服务器响应的数据
  // IncomingMessage 和 ServerResponse 允许继承后进行一些自定义的操作
  // IncomingMessage 主要负责将 HTTP 报文序列化，将报文头报文内容取出序列化
  // ServerResponse 提供了一些快速响应客户端的接口，设置响应报文头报文内容的接口
  this[kIncomingMessage] = options.IncomingMessage || IncomingMessage;
  this[kServerResponse] = options.ServerResponse || ServerResponse;

  // Server 类仍然是继承于 net.Server 类
  net.Server.call(this, { allowHalfOpen: true });

  if (requestListener) {
    // 触发 request 请求事件后，调用 requestListener 函数
    this.on('request', requestListener);
  }

  // Similar option to this. Too lazy to write my own docs.
  // http://www.squid-cache.org/Doc/config/half_closed_clients/
  // http://wiki.squid-cache.org/SquidFaq/InnerWorkings#What_is_a_half-closed_filedescriptor.3F
  this.httpAllowHalfOpen = false;

  this.on('connection', connectionListener);

  // 超时时间，默认禁用
  this.timeout = 0;
  // 对连接超出一定时间不活跃时，断开 TCP 连接，重新建立连接要重新建立三次握手
  this.keepAliveTimeout = 5000;
  // 最大响应头数，默认不限制
  this.maxHeadersCount = null;
  this.headersTimeout = 40 * 1000; // 40 seconds
}
```

我们需要进一步向下剖析，查看一下 `net.Server` 类是怎么样的，该文件位于 `/lib/net.js`

```js
function Server(options, connectionListener) {
  if (!(this instanceof Server))
    return new Server(options, connectionListener);

  // 继承于 EventEmitter 类，这个类属于事件类
  EventEmitter.call(this);

  if (typeof options === 'function') {
    connectionListener = options;
    options = {};
    this.on('connection', connectionListener);
  } else if (options == null || typeof options === 'object') {
    options = { ...options };

    if (typeof connectionListener === 'function') {
      this.on('connection', connectionListener);
    }
  } else {
    throw new ERR_INVALID_ARG_TYPE('options', 'Object', options);
  }

  this._connections = 0;

  Object.defineProperty(this, 'connections', {
    get: deprecate(() => {

      if (this._usingWorkers) {
        return null;
      }
      return this._connections;
    }, 'Server.connections property is deprecated. ' +
       'Use Server.getConnections method instead.', 'DEP0020'),
    set: deprecate((val) => (this._connections = val),
                   'Server.connections property is deprecated.',
                   'DEP0020'),
    configurable: true, enumerable: false
  });

  this[async_id_symbol] = -1;
  this._handle = null;
  this._usingWorkers = false;
  this._workers = [];
  this._unref = false;

  this.allowHalfOpen = options.allowHalfOpen || false;
  this.pauseOnConnect = !!options.pauseOnConnect;
}
Object.setPrototypeOf(Server.prototype, EventEmitter.prototype);
Object.setPrototypeOf(Server, EventEmitter);
```

从代码可以看出，`net.Server` 类本身是一个事件分发中心，具体的实现是由其多个内部方法所组成，我们先看一个我们最常用的方法 `listen`

```js
Server.prototype.listen = function(...args) {
  const normalized = normalizeArgs(args);
  // 序列化以后 options 里面包含了 port 和 host 字段（如果填写了的话）
  var options = normalized[0];
  const cb = normalized[1];

  if (this._handle) {
    throw new ERR_SERVER_ALREADY_LISTEN();
  }

  if (cb !== null) {
    // 给回调函数绑定了一次性事件，当 listening 事件触发时调用回调函数
    this.once('listening', cb);
  }
  
  // ... 

  var backlog;
  if (typeof options.port === 'number' || typeof options.port === 'string') {
    if (!isLegalPort(options.port)) {
      throw new ERR_SOCKET_BAD_PORT(options.port);
    }
    backlog = options.backlog || backlogFromArgs;
    // start TCP server listening on host:port
    if (options.host) {
      // 我们假设 Host 为 localhost， port 为 3000，那么调用的就是 lookupAndListen 方法
      lookupAndListen(this, options.port | 0, options.host, backlog,
                      options.exclusive, flags);
    } else { // Undefined host, listens on unspecified address
      // Default addressType 4 will be used to search for master server
      listenInCluster(this, null, options.port | 0, 4,
                      backlog, undefined, options.exclusive);
    }
    return this;
  }

  // ...
};

function lookupAndListen(self, port, address, backlog, exclusive, flags) {
  if (dns === undefined) dns = require('dns');
  // 期间使用 dns 模块对 host 进行解析，目的是为了得到一个 ip 地址
  dns.lookup(address, function doListen(err, ip, addressType) {
    if (err) {
      self.emit('error', err);
    } else {
      addressType = ip ? addressType : 4;
      // 拿到 ip 地址后执行了 listenInCluster
      listenInCluster(self, ip, port, addressType,
                      backlog, undefined, exclusive, flags);
    }
  });
}

function listenInCluster(server, address, port, addressType,
                         backlog, fd, exclusive, flags) {
  exclusive = !!exclusive;

  // cluster 是 Node 中的集群模块，可以创建共享服务器端口的子进程
  // 在 listen 中的作用是启用一个进程监听 http 请求
  if (cluster === undefined) cluster = require('cluster');

  if (cluster.isMaster || exclusive) {
    // Will create a new handle
    // _listen2 sets up the listened handle, it is still named like this
    // to avoid breaking code that wraps this method
    // 这个方法是最终执行的方法
    server._listen2(address, port, addressType, backlog, fd, flags);
    return;
  }

  const serverQuery = {
    address: address,
    port: port,
    addressType: addressType,
    fd: fd,
    flags,
  };

  // Get the master's server handle, and listen on it
  cluster._getServer(server, serverQuery, listenOnMasterHandle);

  function listenOnMasterHandle(err, handle) {
    err = checkBindError(err, port, handle);

    if (err) {
      var ex = exceptionWithHostPort(err, 'bind', address, port);
      return server.emit('error', ex);
    }

    // Reuse master's server handle
    server._handle = handle;
    // _listen2 sets up the listened handle, it is still named like this
    // to avoid breaking code that wraps this method
    server._listen2(address, port, addressType, backlog, fd, flags);
  }
}

// server._listen2 指向的是一个 setupListenHandle 方法，setupListenHandle 最终指向的是 createServerHandle 方法
// Returns handle if it can be created, or error code if it can't
function createServerHandle(address, port, addressType, fd, flags) {
  var err = 0;
  // Assign handle in listen, and clean up if bind or listen fails
  var handle;

  var isTCP = false;
  if (typeof fd === 'number' && fd >= 0) {
    try {
      handle = createHandle(fd, true);
    } catch (e) {
      // Not a fd we can listen on.  This will trigger an error.
      debug('listen invalid fd=%d:', fd, e.message);
      return UV_EINVAL;
    }

    err = handle.open(fd);
    if (err)
      return err;

    assert(!address && !port);
  } else if (port === -1 && addressType === -1) {
    handle = new Pipe(PipeConstants.SERVER);
    if (process.platform === 'win32') {
      var instances = parseInt(process.env.NODE_PENDING_PIPE_INSTANCES);
      if (!Number.isNaN(instances)) {
        handle.setPendingInstances(instances);
      }
    }
  } else {
    // 最终启动了一个 TCP 服务，当 TCP 端口收到数据时将会推送给 HTTP 进程
    // 具体实现可能是 TCP 内部发出了 emit 进行事件通知，同时将字节流转换为 HTTP 协议所能接受的报文格式（协议规范）
    handle = new TCP(TCPConstants.SERVER);
    isTCP = true;
  }

  if (address || port || isTCP) {
    debug('bind to', address || 'any');
    if (!address) {
      // Try binding to ipv6 first
      err = handle.bind6(DEFAULT_IPV6_ADDR, port, flags);
      if (err) {
        handle.close();
        // Fallback to ipv4
        return createServerHandle(DEFAULT_IPV4_ADDR, port);
      }
    } else if (addressType === 6) {
      err = handle.bind6(address, port, flags);
    } else {
      // 这一步进行主机和端口的绑定
      err = handle.bind(address, port);
    }
  }
  
  return handle;
}
```

从上面我们可以看出，`http.createServer` 的调用过程如下：
  - `http.createServer` 实际上是返回了一个 `Server` 类，而这个类本质上是继承于 `EventEmitter`，用于接收和分发事件以通知外部调用，此时并没有启动进程。
  - `http.createServer().listen()` 这一步在底层，调用了 `Node` 的内建模块 `TCP` 启动了一个 `TCP` 进程并被动打开等待连接。
  - 当 `TCP` 端口接收到连接时，接收来自发送端的数据，然后将其接收到的字节流传递给应用层后通过 `http_parser` 模块分解得到符合 `http` 协议规范的报文格式，以事件(`request`)通知的形式通知给 `Server` 类。
  - `Server` 类在接收到事件通知时，将接收到的数据使用 `IncomingMessage` 类进行解析，将HTTP 报文序列化，将报文头报文内容取出序列化设置为 `Javascript` 的对象格式，同时使用 `ServerResponse` 类提供了一些快速响应客户端的接口，设置响应报文头报文内容的接口，然后将这两项作为 `listen` 的回调参数中的 `(req, res) => {}` 两个参数传入，触发该回调函数。