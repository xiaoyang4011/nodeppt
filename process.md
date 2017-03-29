title: Nodejs之多进程
speaker: 刘晓阳    
transition: slide3
files: /js/demo.js,/css/demo.css,/js/zoom.js
theme: moon
usemathjax: yes
[slide]
#Nodejs之多进程
######  —刘晓阳
[slide]
## 槽点
* 单进程，单线程，无法利用多核CPU {:&.rollIn}
* 可靠性低
* 对程序健壮性要求比较高

[slide]
## Nodejs如何实现多线程？
[slide]

<pre><code class="javascript">

var http = require('http');

http.createServer(function(req, res){
  res.writeHead(200, {'Content-Type' : 'text/plain'});
  res.end('hello world');
}).listen(3000, '127.0.0.1');

</code></pre>

node app

[slide]
<pre><code class="javascript">

events.js:160
      throw er; // Unhandled 'error' event
      ^

Error: listen EADDRINUSE 127.0.0.1:3000
    at Object.exports._errnoException (util.js:896:11)
    at exports._exceptionWithHostPort (util.js:919:20)
    at Server._listen2 (net.js:1246:14)
    at listen (net.js:1282:10)
    at net.js:1392:9
    at _combinedTickCallback (internal/process/next_tick.js:77:11)
    at process._tickCallback (internal/process/next_tick.js:98:9)
    at Function.Module.runMain (module.js:577:11)
    at startup (node.js:159:18)
    at node.js:444:3
</code></pre>

[slide]
## Nginx proxy

<div class="columns1">
    <img src="http://7xppjy.com1.z0.glb.clouddn.com/Class%20Diagram.png">
</div>


[slide]

## master-worker

<div class="columns1">
    <img src="http://7xppjy.com1.z0.glb.clouddn.com/TB1XNnNJVXXXXanXpXXQzA.9VXX-447-300.png">
</div>

[slide]

## demo
<pre><code class="javascript">
//master.js
const net = require('net');
const fork = require('child_process').fork;

var handle= net._createServerHandle('0.0.0.0', 3000);

for(let a =0; a < 4; a++){
  fork('./worker.js').send({}, handle);
}
</code>
</pre>
[slide]

## demo
<pre><code class="javascript">
//worker.js
const net = require('net');
process.on('message', function(m, server){
    server.listen();
    server.onconnection = function(err, handle){
    console.log('got a connection on worker, pid = %d', process.pid);
    var socket = new net.Socket({
      handle: handle
    });
    socket.readable = socket.writable = true;
    socket.end('hello js');
  }
});
</code>
</pre>

[slide]

## 多进程监听同一端口的进程模型

<div class="columns1">
    <img src="http://7xppjy.com1.z0.glb.clouddn.com/TB1bexvKpXXXXaMXXXX3GwW0VXX-426-298.png">
</div>

[slide]
## 有问题吗？

* 多个进程会竞争 accpet 一个连接。 {:&.rollIn}
* 无法控制请求由哪个进程去处理，导致各个worker之间负载不均衡。

[slide]

## 由master进程负责监听和调度任务

[slide]

## demo
<pre><code class="javascript">
//master.js
const net = require('net');
const fork = require('child_process').fork;

var workers = [];

for(let i = 0; i < 4; i++) {
  workers.push(fork('./worker.js'));
}

var serverHandle = net._createServerHandle('0.0.0.0', 3000);
serverHandle.listen();

serverHandle.onconnection = function(err, handle) {
  var worker = workers.pop();
  worker.send({}, handle);
  workers.unshift(worker);
}
</code>

[slide]
## demo
<pre><code class="javascript">
const net = require('net');
process.on('message', function(m, handle){
  console.log('got a connection on work , pid = %d', process.pid);

  var socket = new net.Socket({
    handle : handle
  });

  socket.end('hello world');
});
</code>
[slide]

## 进程模型如下图
<div class="columns1">
    <img src="http://7xppjy.com1.z0.glb.clouddn.com/TB1kT6gKpXXXXbyXXXXvNGU0VXX-533-352.png">
</div>

[slide]
## 进程守护

master 进程除了负责接收新的连接，分发给各 worker 进程处理之外，还得像天使一样默默地守护着这些 worker 进程，保障整个应用的稳定性。一旦某个 worker 进程异常退出就 fork 一个新的子进程顶替上去。

[slide]
## 关于 uncaughtException

*  我们常说的程序崩了，什么是崩了 {:&.rollIn}
*  什么是uncaughtException
*  如何优雅的退出
[slide]
## 线上问题

### 如果线上出了这样的FATAL会发生什么？
<div class="columns1">
    <img src="http://7xppjy.com1.z0.glb.clouddn.com/lALOK21qUM0BZc0D5w_999_357.png">
</div>

[slide]
## demo

<pre><code class="javascript">
function demo (req, res) {
	res.json({code : 1, msg : 'err'});
	return res.json({code : 1, msg : 'err2'});
}
</code>

[slide]
## demo

<pre><code class="javascript">
function demo (req, res) {

	request_api_lib.request_get(url, function(){
		if(err) {
			res.json({code : 1, msg : 'err'});
		}

		return res.json({code : 1, msg : 'err2'});
	});
}
</code>
[slide]
## 后果很严重
<div class="columns1">
    <img src="http://7xppjy.com1.z0.glb.clouddn.com/lALOK21n6VXNBCE_1057_85.png">
</div>

* 6000 * 3 = 18000 {:&.rollIn}

[slide]

## Seq
<div class="columns1">
    <img src="http://7xppjy.com1.z0.glb.clouddn.com/lALOK3WVoc0Bjs0DAg_770_398.png">
</div>

* 有问题吗？ {:&.rollIn}
[slide]
## undefined.name
* customer && customer.name || '' {:&.rollIn}
* _.get(customer, 'name', '')

[slide]
## 当我们比较语言优劣，容易局限在语言本身，而忽视了配套的一些关键因素
[slide]
## Thank you for guys
[slide]
