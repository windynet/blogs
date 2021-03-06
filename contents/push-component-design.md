# 推送组件设计

### 目标

+ 需要支持单一推送、分组推送、全局广播
+ 支持多种通信方式，例如 TCP、WebSocket
+ 启动后，分组可以改变
+ 推送失败后支持多种策略，例如丢弃数据、延迟几次后重发、告警、消息持久化
+ 如果某段时间内，例如 1 秒内，需要向某个连接发送多条消息时，支持把这段时间内的消息合并成数组再发送，或只推送这段时间内最新的那条消息

### 一种实现思路

+ 提供 `subscribe` 接口，在客户端建立连接后订阅需要的消息，或动态订阅更多的消息
+ 提供 `unsubscribe` 接口，在客户端的连接断开后取消自己所有的订阅，或动态取消已有的订阅
+ 提供 `publish` 接口，为数据源提供数据发布服务

### 相应的代码

```ts
type Handle = (message: any) => void;

const mapper = new Map<string, Set<Handle>>();

function subscribe(tagName: string, handle: Handle) {
    let handlers = mapper.get(tagName);
    if (!handlers) {
        handlers = new Set<Handle>();
        mapper.set(tagName, handlers);
    }
    handlers.add(handle);
}

export function subscribeGroup(groupName: string, groupId: string | number, handle: Handle) {
    subscribe(`${groupName}_${groupId}`, handle);
}

export function subscribeGroups(groupName: string, groupIds: (string | number)[], handle: Handle) {
    for (const groupId of groupIds) {
        subscribeGroup(groupName, groupId, handle);
    }
}

export function subscribeGlobal(handle: Handle) {
    subscribe("*", handle);
}

function unsubscribe(tagName: string, handle: Handle) {
    const handles = mapper.get(tagName);
    if (handles) {
        handles.delete(handle);
        if (handles.size === 0) {
            mapper.delete(tagName);
        }
    }
}

export function unsubscribeGroup(groupName: string, groupId: string | number, handle: Handle) {
    unsubscribe(`${groupName}_${groupId}`, handle);
}

export function unsubscribeGroups(groupName: string, groupIds: (string | number)[], handle: Handle) {
    for (const groupId of groupIds) {
        unsubscribeGroup(groupName, groupId, handle);
    }
}

export function unsubscribeGlobal(handle: Handle) {
    unsubscribe("*", handle);
}

function publish(tagName: string, message: any) {
    let handlers = mapper.get(tagName);
    if (handlers) {
        for (const handle of handlers) {
            handle(message);
        }
    }
}

export function publishToGroup(groupName: string, groupId: string | number, message: any) {
    publish(`${groupName}_${groupId}`, message);
}

export function publishToGlobal(message: any) {
    publish("*", message);
}
```

### 使用场景

先引入 pusher：

```ts
import * as pusher from "./pusher";
```

某用户的 id 是 100，关注 topic a，订阅操作如下：

```ts
function handle1(message: any) {
    console.log(`user 100 accepted: ${message}`);
}
pusher.subscribeGroup("userid", 100, handle1);
pusher.subscribeGroup("topic", "a", handle1);
pusher.subscribeGlobal(handle1);
```

某用户的 id 是 101，也关注 topic a，订阅操作如下：

```ts
function handle2(message: any) {
    console.log(`user 101 accepted: ${message}`);
}
pusher.subscribeGroup("userid", 101, handle2);
pusher.subscribeGroup("topic", "a", handle2);
pusher.subscribeGlobal(handle2);
```

某用户的 id 是 102，关注 topic b，订阅操作如下：

```ts

function handle3(message: any) {
    console.log(`user 102 accepted: ${message}`);
}
pusher.subscribeGroup("userid", 102, handle3);
pusher.subscribeGroup("topic", "b", handle3);
pusher.subscribeGlobal(handle3);
```

当数据源希望向用户 100 推送数据，推送操作如下：

```ts
pusher.publishToGroup("userid", 100, "a message to user 100 only");
```

结果提示：

```bash
user 100 accepted: a message to user 100 only
```

当数据源希望向 topic a 推送数据，推送操作如下：

```ts
pusher.publishToGroup("topic", "a", "a message to topic a");
```

结果提示：

```bash
user 100 accepted: a message to topic a
user 101 accepted: a message to topic a
```

当数据源希望向所有连接推送数据，推送操作如下：

```ts
pusher.publishToGlobal("a message to all users");
```

结果提示：

```bash
user 100 accepted: a message to all users
user 101 accepted: a message to all users
user 102 accepted: a message to all users
```

### TCP 连接的场景

```ts
import * as net from "net";
net.createServer(socket => {
    // todo: validate user permission, get use id, and the topics he is interested in
    const userId = 103;
    const topics = ["c", "d"];
    function handle(message: any) {
        socket.emit("data", message);
    }
    pusher.subscribeGroup("userid", userId, handle);
    pusher.subscribeGroups("topic", topics, handle);
    pusher.subscribeGlobal(handle);
    socket.on("close", hadError => {
        pusher.unsubscribeGroup("userid", userId, handle);
        pusher.unsubscribeGroups("topic", topics, handle);
        pusher.unsubscribeGlobal(handle);
    });
}).listen(8000);
```

### WebSocket 连接的场景

```ts
import * as WebSocket from "ws";
WebSocket.createServer({ port: 8000 }, ws => {
    // todo: validate user permission, get use id, and the topics he is interested in
    const userId = 103;
    const topics = ["c", "d"];
    function handle(message: any) {
        ws.send(message);
    }
    pusher.subscribeGroup("userid", userId, handle);
    pusher.subscribeGroups("topic", topics, handle);
    pusher.subscribeGlobal(handle);
    ws.on("close", code => {
        pusher.unsubscribeGroup("userid", userId, handle);
        pusher.unsubscribeGroups("topic", topics, handle);
        pusher.unsubscribeGlobal(handle);
    });
});
```

### 消息推送失败时的处理策略

```ts
function handle(message: any) {
    // 1st trial
    ws.send(message, error1 => {
        if (error1) {
            setTimeout(() => {
                // 2nd trial
                ws.send(message, error2 => {
                    if (error2) {
                        setTimeout(() => {
                            // 3rd trial
                            ws.send(message, error3 => {
                                if (error3) {
                                    // alarm or persist data to database
                                }
                            });
                        }, 1000);
                    }
                });
            }, 1000);
        }
    });
}
```

### 消息合并

```ts
let timer: number | undefined;
let messages = [];

function handle1(message: any) {
    messages.push(message);
    if (timer === undefined) {
        timer = setTimeout(() => {
            timer = undefined;
            console.log(`user 100 accepted: ${JSON.stringify(messages, null, "  ")}`);
            messages = [];
        }, 1000);
    }
}
pusher.subscribeGroup("userid", 100, handle1);

let index = 0;
setInterval(() => {
    pusher.publishToGroup("userid", 100, `message ${index++}`);
}, 400);
```

### 只推送最新消息

```ts
let timer: number | undefined;
let latestMessage: string | undefined = undefined;

function handle1(message: any) {
    latestMessage = message;
    if (timer === undefined) {
        timer = setTimeout(() => {
            timer = undefined;
            console.log(`user 100 accepted: ${latestMessage}`);
            latestMessage = undefined;
        }, 1000);
    }
}
pusher.subscribeGroup("userid", 100, handle1);

let index = 0;
setInterval(() => {
    pusher.publishToGroup("userid", 100, `message ${index++}`);
}, 400);
```

### 另一种实现思路

对于这种多输入、多输出的场景，非常适合使用 rxjs 来实现，性能会有少量损失，不过代码量会缩小很多，会提高可维护性。

### rxjs 思路的相应实现

先引入 rxjs，并创建场景对应的主题：

```ts
import { Subject } from "rxjs";

const useridSubject = new Subject<{ userid: number, message: any }>(); // 按 userid 单一推送
const topicSubject = new Subject<{ name: string, message: any }>(); // 按 topic 分组推送
const globalSubject = new Subject<any>(); // 全局广播
```

某用户的 id 是 100，关注 topic a，订阅操作如下：

```ts
function handle1(message: any) {
    console.log(`user 100 accepted: ${message}`);
}
useridSubject.filter(s => s.userid === 100).subscribe(v => handle1(v.message));
topicSubject.filter(s => s.name === "a").subscribe(v => handle1(v.message));
globalSubject.subscribe(v => handle1(v));
```

某用户的 id 是 101，也关注 topic a，订阅操作如下：

```ts
function handle2(message: any) {
    console.log(`user 101 accepted: ${message}`);
}
useridSubject.filter(s => s.userid === 101).subscribe(v => handle2(v.message));
topicSubject.filter(s => s.name === "a").subscribe(v => handle2(v.message));
globalSubject.subscribe(v => handle2(v));
```

某用户的 id 是 102，关注 topic b，订阅操作如下：

```ts

function handle3(message: any) {
    console.log(`user 102 accepted: ${message}`);
}
useridSubject.filter(s => s.userid === 102).subscribe(v => handle3(v.message));
topicSubject.filter(s => s.name === "b").subscribe(v => handle3(v.message));
globalSubject.subscribe(v => handle3(v));
```

当数据源希望向用户 100 推送数据，推送操作如下：

```ts
useridSubject.next({ userid: 100, message: "a message to user 100 only" });
```

结果提示：

```bash
user 100 accepted: a message to user 100 only
```

当数据源希望向 topic a 推送数据，推送操作如下：

```ts
topicSubject.next({ name: "a", message: "a message to topic a" });
```

结果提示：

```bash
user 100 accepted: a message to topic a
user 101 accepted: a message to topic a
```

当数据源希望向所有连接推送数据，推送操作如下：

```ts
globalSubject.next("a message to all users");
```

结果提示：

```bash
user 100 accepted: a message to all users
user 101 accepted: a message to all users
user 102 accepted: a message to all users
```

### rxjs 思路的 TCP 连接的场景

```ts
import * as net from "net";
net.createServer(socket => {
    // todo: validate user permission, get use id, and the topics he is interested in
    const userId = 103;
    const topics = ["c", "d"];
    function handle(message: any) {
        socket.emit("data", message);
    }
    const useridSubscription = useridSubject.filter(s => s.userid === userId).subscribe(v => handle(v.message));
    const topicSubscription = topicSubject.filter(s => topics.some(t => t === s.name)).subscribe(v => handle(v.message));
    const globalSubscription = globalSubject.subscribe(v => handle(v));
    socket.on("close", hadError => {
        useridSubscription.unsubscribe();
        topicSubscription.unsubscribe();
        globalSubscription.unsubscribe();
    });
}).listen(8000);
```

### rxjs 思路的 WebSocket 连接的场景

```ts
import * as WebSocket from "ws";
WebSocket.createServer({ port: 8000 }, ws => {
    // todo: validate user permission, get use id, and the topics he is interested in
    const userId = 103;
    const topics = ["c", "d"];
    function handle(message: any) {
        ws.send(message);
    }
    const useridSubscription = useridSubject.filter(s => s.userid === userId).subscribe(v => handle(v.message));
    const topicSubscription = topicSubject.filter(s => topics.some(t => t === s.name)).subscribe(v => handle(v.message));
    const globalSubscription = globalSubject.subscribe(v => handle(v));
    ws.on("close", code => {
        useridSubscription.unsubscribe();
        topicSubscription.unsubscribe();
        globalSubscription.unsubscribe();
    });
});
```

### rxjs 思路的消息合并

```ts
function handle1(messages: any) {
    console.log(`user 100 accepted: ${JSON.stringify(messages, null, "  ")}`);
}
useridSubject.filter(s => s.userid === 100)
    .bufferTime(1000)
    .filter(s => s.length > 0)
    .map(s => s.map(m => m.message))
    .subscribe(v => handle1(v));

let index = 0;
setInterval(() => {
    useridSubject.next({ userid: 100, message: `message ${index++}` });
}, 400);
```

### rxjs 思路的只推送最新消息

```ts
function handle1(message: any) {
    console.log(`user 100 accepted: ${message}`);
}
useridSubject.filter(s => s.userid === 100)
    .bufferTime(1000)
    .filter(s => s.length > 0)
    .map(s => s[s.length - 1].message)
    .subscribe(v => handle1(v));

let index = 0;
setInterval(() => {
    useridSubject.next({ userid: 100, message: `message ${index++}` });
}, 400);
```
