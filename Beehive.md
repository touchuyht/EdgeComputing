# Beehive

## Beehive概述

Beehive是一个基于go channel的消息框架，它用于沟通KubeEdge的各个模块。在Beehive中注册的模块可以通过已经注册的模块的group名或者该模块的名字与之进行通信。Beehive支持如下的操作：

1、添加module

2、将module添加到某个group

3、CleanUp（从beehive core和所有groups中删除一个module）

Beehive支持对消息进行如下的操作：

1、发送到一个module/group

2、被某个module接收

3、发送sync到一个module/group

4、发送对sync的response

## 消息格式

消息由三部分组成：

1、Header:

​	1）ID：消息ID，string类型

​	2）ParentID：出现在对sync的response消息中，string类型

​	3）TimeStamp：消息生成的时间，int类型

​	4）Sync：用于说明消息是否为Sync类型的标志位，bool类型

2、Route：

​	1）Source：消息源，string类型

​	2）Group：消息被广播到的group，string类型

​	3）Operation：对资源的操作？，string类型

​	4）Resource：指定了要操作的资源，string类型

3、Content：消息的内容，interface{}类型

## Module的注册

1、edgecore在启动的同时，各个模块就开始尝试向beehive core注册自己；

2、beehive core有一个map，用于保存注册的module，其中module的name是键，module的实现接口是值；

3、当一个模块尝试向beehive core注册自己时，beehive从已经加载的module.yaml配置文件检测该模块是否启用。如果该模块被启用，则将其加入modules map，否则将其加入disabled modules map。

## 通道的上下文结构字段（Channel Context Structure Fields）

此部分对于理解beehive的操作重要。

1、**channels**：channels是一个map[string]chan message类型的map，其中module的名字是键值，消息channel，该channel可以用于发送消息到各个模块；

2、**chsLock**：控制channels字典的锁

3、**typeChannels**：typeChannels是map[string]string类型的字典，group name是键，map[string]chan message是值（group中每个module的name是键，与对应module连接的channel是值）

4、**typeChsLock**：控制typeChannels字典的锁

5、**anonChannels**：anonChannels是map[string] chan message类型的map，其中parentid是键，消息通道是值，该字段的作用是作为sync类型消息的response

6、**anonChsLock**：控制anonChannels字典的锁

```go
// ChannelContext is object for Context channel
type ChannelContext struct {
	//ConfigFactory goarchaius.ConfigurationFactory
	channels     map[string]chan model.Message
	chsLock      sync.RWMutex
	typeChannels map[string]map[string]chan model.Message
	typeChsLock  sync.RWMutex
	anonChannels map[string]chan model.Message
	anonChsLock  sync.RWMutex
}
```

## **Module的操作**

### Add Module

1、添加module的操作首先会创建一个chan message类型的channel；

2、然后module的名字作为键，上一步创建的channel作为值会被添加到channels中（map[string] chan message）

e.g.添加edged module

```go
coreContext.Addmodule("edged")
```

```go
// AddModule adds module into module context
func (ctx *ChannelContext) AddModule(module string) {
	channel := ctx.newChannel()
	ctx.addChannel(module, channel)
}

// addChannel return chan
func (ctx *ChannelContext) addChannel(module string, moduleCh chan model.Message) {
	ctx.chsLock.Lock()
	defer ctx.chsLock.Unlock()

	ctx.channels[module] = moduleCh
}
```

### Add Module to Group

1、添加module到group的操作首先会从channels字典字段得到一个模块的channel；

2、然后module和上一步得到的channel到typeChannels字典中，其中group name是键，map[string]chan message类型的字典是值

e.g.添加edged module到edged group中。下面的例子中，参数1是模块的名字，参数2是group的名字

```go
coreContext.AddModuleGroup("edged","edged")
```

```go
// SendToGroup send msg to modules. Todo: do not stuck
func (ctx *ChannelContext) SendToGroup(moduleType string, message model.Message) {
	// avoid exception because of channel closing
	// TODO: need reconstruction
	defer func() {
		if exception := recover(); exception != nil {
			klog.Warningf("Recover when sendToGroup message, exception: %+v", exception)
		}
	}()

	send := func(ch chan model.Message) {
		select {
		case ch <- message:
		default:
			klog.Warningf("the message channel is full, message: %+v", message)
			select {
			case ch <- message:
			}
		}
	}
	if channelList := ctx.getTypeChannel(moduleType); channelList != nil {
		for _, channel := range channelList {
			go send(channel)
		}
		return
	}
	klog.Warningf("Get bad module type:%s when sendToGroup message, do nothing", moduleType)
}
```

### CleanUp

1、CleanUp从channels map中删除此module，并且从所用groups中将其删除(typeChannels字典)；

2、与此module进行通信的channel被关闭

e.d. CleanUp edged module

```go
coreContext.CleanUp("edged")
```

```go
// Cleanup close modules
func (ctx *ChannelContext) Cleanup(module string) {
	if channel := ctx.getChannel(module); channel != nil {
		ctx.delChannel(module)
		// decrease probable exception of channel closing
		time.Sleep(20 * time.Millisecond)
		close(channel)
	}
}

// deleteChannel by module name
func (ctx *ChannelContext) delChannel(module string) {
	// delete module channel from channels map
	ctx.chsLock.Lock()
	_, exist := ctx.channels[module]
	if !exist {
		klog.Warningf("Failed to get channel, module:%s", module)
		return
	}
	delete(ctx.channels, module)

	ctx.chsLock.Unlock()

	// delete module channel from typechannels map
	ctx.typeChsLock.Lock()
	for _, moduleMap := range ctx.typeChannels {
		if _, exist := moduleMap[module]; exist {
			delete(moduleMap, module)
			break
		}
	}
	ctx.typeChsLock.Unlock()
}
```

## Message的操作

### Send to a Module

1、从channels字典中得到与要通信模块的对应channel；

2、消息被发送到channel中

e.g. 发送消息到edged module

```go
coreContext.Send("edged",message)
```

```go
// Send send msg to a module. Todo: do not stuck
func (ctx *ChannelContext) Send(module string, message model.Message) {
	// avoid exception because of channel closing
	// TODO: need reconstruction
	defer func() {
		if exception := recover(); exception != nil {
			klog.Warningf("Recover when send message, exception: %+v", exception)
		}
	}()

	if channel := ctx.getChannel(module); channel != nil {
		channel <- message
		return
	}
	klog.Warningf("Get bad module name :%s when send message, do nothing", module)
}

// getChannel return chan
func (ctx *ChannelContext) getChannel(module string) chan model.Message {
	ctx.chsLock.RLock()
	defer ctx.chsLock.RUnlock()

	if _, exist := ctx.channels[module]; exist {
		return ctx.channels[module]
	}

	klog.Warningf("Failed to get channel, type:%s", module)
	return nil
}
```

### Send to a Group

1、SendToGroup从typeChannels字典中得到所有模块的channels字典；

2、在得到的字典中迭代，然后向每个模块发送消息

e.g. 发送消息到edged group的所有模块

```go
coreContext.SendToGroup("edged",message)
```

```go
// SendToGroup send msg to modules. Todo: do not stuck
func (ctx *ChannelContext) SendToGroup(moduleType string, message model.Message) {
	// avoid exception because of channel closing
	// TODO: need reconstruction
	defer func() {
		if exception := recover(); exception != nil {
			klog.Warningf("Recover when sendToGroup message, exception: %+v", exception)
		}
	}()

	send := func(ch chan model.Message) {
		select {
		case ch <- message:
		default:
			klog.Warningf("the message channel is full, message: %+v", message)
			select {
			case ch <- message:
			}
		}
	}
	if channelList := ctx.getTypeChannel(moduleType); channelList != nil {
		for _, channel := range channelList {
			go send(channel)
		}
		return
	}
	klog.Warningf("Get bad module type:%s when sendToGroup message, do nothing", moduleType)
}

// getTypeChannel return chan
func (ctx *ChannelContext) getTypeChannel(moduleType string) map[string]chan model.Message {
	ctx.typeChsLock.RLock()
	defer ctx.typeChsLock.RUnlock()

	if _, exist := ctx.typeChannels[moduleType]; exist {
		return ctx.typeChannels[moduleType]
	}

	klog.Warningf("Failed to get type channel, type:%s", moduleType)
	return nil
}
```

### Receive by a Module

1、Receive得到从channels字典中得到一个module的channel；

2、然后它就等待在这个channel上，以期消息的到达并返回。同时如果有Error也会返回。

e.g. 从edged module中接收消息

```go
msg, err:=coreContext.Receive("edged")
```

```go
// Receive msg from channel of module
func (ctx *ChannelContext) Receive(module string) (model.Message, error) {
	if channel := ctx.getChannel(module); channel != nil {
		content := <-channel
		return content, nil
	}

	klog.Warningf("Failed to get channel for module:%s when receive message", module)
	return model.Message{}, fmt.Errorf("failed to get channel for module(%s)", module)
}

// getChannel return chan
func (ctx *ChannelContext) getChannel(module string) chan model.Message {
	ctx.chsLock.RLock()
	defer ctx.chsLock.RUnlock()

	if _, exist := ctx.channels[module]; exist {
		return ctx.channels[module]
	}

	klog.Warningf("Failed to get channel, type:%s", module)
	return nil
}
```

### SendSync to a Module

1、SendSync接收3个参数(module, message, timeout duration)；

2、SendSync首先从channels字典中得到module的通信channel；

3、messages被发送到channel；

4、一个新的用于发送消息的channel被创建并且被添加到annoChannels中，其中messageID是键；

5、在超时之前它都会在自己创建的annoChannel等待接收response；

6、如果在超时之前收到了消息，则消息被返回，err为nil否则返回time out error。

e.g. 给edged module发送sync类型的消息， 超时间隔设为60s

```go
response, err:=coreContext.SendSync("edged", message, 60*time.Second)
```

```go
// SendSync sends message in a sync way
func (ctx *ChannelContext) SendSync(module string, message model.Message, timeout time.Duration) (model.Message, error) {
	// avoid exception because of channel closing
	// TODO: need reconstruction
	defer func() {
		if exception := recover(); exception != nil {
			klog.Warningf("Recover when sendsync message, exception: %+v", exception)
		}
	}()

	if timeout <= 0 {
		timeout = MessageTimeoutDefault
	}
	deadline := time.Now().Add(timeout)

	// make sure to set sync flag
	message.Header.Sync = true

	// check req/resp channel
	reqChannel := ctx.getChannel(module)
	if reqChannel == nil {
		return model.Message{}, fmt.Errorf("bad request module name(%s)", module)
	}

	sendTimer := time.NewTimer(timeout)
	select {
	case reqChannel <- message:
	case <-sendTimer.C:
		return model.Message{}, errors.New("timeout to send message")
	}
	sendTimer.Stop()

	// new anonymous channel for response
	anonChan := make(chan model.Message)
	anonName := getAnonChannelName(message.GetID())
	ctx.anonChsLock.Lock()
	ctx.anonChannels[anonName] = anonChan
	ctx.anonChsLock.Unlock()
	defer func() {
		ctx.anonChsLock.Lock()
		delete(ctx.anonChannels, anonName)
		close(anonChan)
		ctx.anonChsLock.Unlock()
	}()

	var resp model.Message
	respTimer := time.NewTimer(time.Until(deadline))
	select {
	case resp = <-anonChan:
	case <-respTimer.C:
		return model.Message{}, errors.New("timeout to get response")
	}
	respTimer.Stop()

	return resp, nil
}

// getChannel return chan
func (ctx *ChannelContext) getChannel(module string) chan model.Message {
	ctx.chsLock.RLock()
	defer ctx.chsLock.RUnlock()

	if _, exist := ctx.channels[module]; exist {
		return ctx.channels[module]
	}

	klog.Warningf("Failed to get channel, type:%s", module)
	return nil
}

func getAnonChannelName(msgID string) string {
	return msgID
}
```

### SendSync to a Group

1、从group的typeChannels字典中得到modules的列表；

2、创建用于发送消息的channel，channel的缓冲区大小设置成和group中的modules的个数一致，并且被插入anonChannels字典中作为值，键为messageID；

3、在所有modules的channels上发送消息；

4、在超时之间等待。如果anonChannel的长度和group中modules的长度都不一致，则会检查channel中的message是否patentID=messageID。如果不是则返回error否则返回nil error；

5、如果超时则返回timeout error

e.g. 向edged group发送sync类型的消息，超时间隔设置为60s

```go
err:=coreContext.SendToGroupSync("edged",message,60*time.Second)
```

```go
// SendToGroupSync : broadcast the message to echo module channel, the module send response back anon channel
// check timeout and the size of anon channel
func (ctx *ChannelContext) SendToGroupSync(moduleType string, message model.Message, timeout time.Duration) error {
	// avoid exception because of channel closing
	// TODO: need reconstruction
	defer func() {
		if exception := recover(); exception != nil {
			klog.Warningf("Recover when sendToGroupsync message, exception: %+v", exception)
		}
	}()

	if timeout <= 0 {
		timeout = MessageTimeoutDefault
	}
	deadline := time.Now().Add(timeout)

	channelList := ctx.getTypeChannel(moduleType)
	if channelList == nil {
		return fmt.Errorf("failed to get module type(%s) channel list", moduleType)
	}

	// echo module must sync a response,
	// let anonchan size be module number
	channelNumber := len(channelList)
	anonChan := make(chan model.Message, channelNumber)
	anonName := getAnonChannelName(message.GetID())
	ctx.anonChsLock.Lock()
	ctx.anonChannels[anonName] = anonChan
	ctx.anonChsLock.Unlock()

	cleanup := func() error {
		ctx.anonChsLock.Lock()
		delete(ctx.anonChannels, anonName)
		close(anonChan)
		ctx.anonChsLock.Unlock()

		var uninvitedGuests int
		// cleanup anonchan and check parentid for resp
		for resp := range anonChan {
			if resp.GetParentID() != message.GetID() {
				uninvitedGuests++
			}
		}
		if uninvitedGuests != 0 {
			klog.Errorf("Get some unexpected:%d resp when sendToGroupsync message", uninvitedGuests)
			return fmt.Errorf("got some unexpected(%d) resp", uninvitedGuests)
		}
		return nil
	}

	// make sure to set sync flag before sending
	message.Header.Sync = true

	var timeoutCounter int32
	send := func(ch chan model.Message) {
		sendTimer := time.NewTimer(time.Until(deadline))
		select {
		case ch <- message:
			sendTimer.Stop()
		case <-sendTimer.C:
			atomic.AddInt32(&timeoutCounter, 1)
		}
	}
	for _, channel := range channelList {
		go send(channel)
	}

	sendTimer := time.NewTimer(time.Until(deadline))
	ticker := time.NewTicker(TickerTimeoutDefault)
	for {
		// annonChan is full
		if len(anonChan) == channelNumber {
			break
		}
		select {
		case <-ticker.C:
		case <-sendTimer.C:
			cleanup()
			if timeoutCounter != 0 {
				errInfo := fmt.Sprintf("timeout to send message, several %d timeout when send", timeoutCounter)
				return fmt.Errorf(errInfo)
			}
			klog.Error("Timeout to sendToGroupsync message")
			return fmt.Errorf("Timeout to send message")
		}
	}

	return cleanup()
}

// getTypeChannel return chan
func (ctx *ChannelContext) getTypeChannel(moduleType string) map[string]chan model.Message {
	ctx.typeChsLock.RLock()
	defer ctx.typeChsLock.RUnlock()

	if _, exist := ctx.typeChannels[moduleType]; exist {
		return ctx.typeChannels[moduleType]
	}

	klog.Warningf("Failed to get type channel, type:%s", moduleType)
	return nil
}

func getAnonChannelName(msgID string) string {
	return msgID
}
```

### SendResp to a sync message

1、SendResp被用于向sync类型的消息发送response；

2、response的parentID必须设置的和sync消息的messageID相同；

3、当SendResp被调用时，首先会检查对于response消息的parentID，是否有anonChannels类型的通道存在；

4、如果有channel，response消息被发送到对应的channel上。

5、否则其他类型的error被记录下。

```go
coreContext.SendResp(respMessage)
```

```go
// SendResp send resp for this message when using sync mode
func (ctx *ChannelContext) SendResp(message model.Message) {
	anonName := getAnonChannelName(message.GetParentID())

	ctx.anonChsLock.RLock()
	defer ctx.anonChsLock.RUnlock()
	if channel, exist := ctx.anonChannels[anonName]; exist {
		channel <- message
		return
	}

	klog.V(4).Infof("Get bad anonName:%s when sendresp message, do nothing", anonName)
}

func getAnonChannelName(msgID string) string {
	return msgID
}
```

