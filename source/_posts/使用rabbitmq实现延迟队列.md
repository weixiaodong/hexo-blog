---
title: 使用rabbitmq实现延迟队列
date: 2023-03-21 20:09:23
tags:
---

# rabbitmq使用

首先要引入一个概念： 死信队列，当我们发送的消息被接收端ack或reject，消息在队列的存活时间超过设定的ttl，消息数量超过队列最大长度，这样的消息会被认为是死信("dead letter")通过配置的死信交换机这样的死信可以被投递到对应的死信队列中。

<!--more-->

![](images/rabbitmq死信队列.png)

1. 我们将需要延迟的消息设定需要延迟的时间，也就是这个消息的最大存活时间（ttl），然后发送到普通队列中
2. 然后因为普通队列中没有消费者，所以只有静静的等待消息超时
3. 消息超时后，经过死信交换机，发送到对应的死信队列中
4. 我们只需要消费死信队列中的消息即可认为当前消息对应的队列已经到了应该执行的时间了



# golang 生产端实现

发送者的实现就很简单了，就和普通的发送实现几乎一致，因为反正就是投递到对应的队列中就可以了，只需要将发送消息的部分，在消息的header中加入x-delay字段表示当前消息的ttl就可以了，也就是设定延迟时间，注意单位为毫秒

```golang
package producer

import (
	"encoding/json"
	"log"
	"time"

	"github.com/streadway/amqp"
)

type Producer struct {
	Addr       string
	Exchange   string
	RoutingKey string
	Queue      string

	conn       *amqp.Connection
	Channel    *amqp.Channel
	done       chan bool
	connErr    chan error
	channelErr chan *amqp.Error
}

func NewProducer(addr string, exchange string, queue string, key string) *Producer {
	return &Producer{
		Addr:       addr,
		Exchange:   exchange,
		RoutingKey: key,
		Queue:      queue,
		done:       make(chan bool),
		connErr:    make(chan error),
		channelErr: make(chan *amqp.Error),
	}
}

// connect 连接到mq服务器
func (c *Producer) Connect() error {
	var err error
	if c.conn, err = amqp.Dial(c.Addr); err != nil {
		return err
	}

	if c.Channel, err = c.conn.Channel(); err != nil {
		_ = c.Close()
		return err
	}

	// 声明一个主要使用的 exchange
	err = c.Channel.ExchangeDeclare(
		c.Exchange, "x-delayed-message", true, false, false, false, amqp.Table{
			"x-delayed-type": "fanout",
		})
	if err != nil {
		return err
	}
	// 声明一个延时队列, 延时消息就是要发送到这里
	q, err := c.Channel.QueueDeclare(c.Queue, false, false, false, false, nil)
	if err != nil {
		return err
	}

	err = c.Channel.QueueBind(q.Name, "", c.Exchange, false, nil)
	if err != nil {
		return err
	}

	return nil
}

func (c *Producer) Close() error {
	close(c.done)

	if !c.conn.IsClosed() {
		if err := c.conn.Close(); err != nil {
			log.Print("rabbitmq producer - connection close failed: ", err)
			return err
		}
	}
	return nil
}

// publish 发送消息至mq
func (c *Producer) Publish(body []byte, delay int64) error {
	publising := amqp.Publishing{
		ContentType: "text/plain",
		Body:        body,
	}

	if delay >= 0 {
		publising.Headers = amqp.Table{
			"x-delay": delay,
		}
	}

	err := c.Channel.Publish(c.Exchange, c.RoutingKey, false, false, publising)
	if err != nil {
		switch v := err.(type) {
		case *amqp.Error:
			c.channelErr <- v
		default:
			c.connErr <- v
		}
	}
	return err
}

// PublishJSON 将对象JSON格式化后发送消息
func (c *Producer) PublishJSON(body interface{}, delay int64) error {
	data, err := json.Marshal(body)
	if err != nil {
		return err
	}

	return c.Publish(data, delay)
}

// watchconn 监控mq的连接状态
func (c *Producer) WatchConnect() {
	ticker := time.NewTicker(30 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case err := <-c.connErr:
			log.Printf("rabbitmq producer - connection notify close: %s", err.Error())
			c.ReConnect()
		case err := <-c.channelErr:
			log.Printf("rabbitmq producer - channel notify close: %s", err.Error())
			c.ReConnect()
		case <-ticker.C:
			c.ReConnect()

		case <-c.done:
			log.Print("auto detect connection is done")
			return

		}
	}

}

// ReConnect 根据当前链接状态判断是否需要重新连接，如果连接异常则尝试重新连接
func (c *Producer) ReConnect() {
	if c.conn == nil || (c.conn != nil && c.conn.IsClosed()) {
		log.Printf("rabbitmq connection is closed try to reconnect")
		if err := c.Connect(); err != nil {
			log.Printf("rabbitmq reconnect failed: %s", err.Error())
		} else {
			log.Printf("rabbitmq reconnect succeeded")
		}
	}
}
```

# golang 消费端实现

```golang
package consumer

import (
	"log"
	"os"
	"time"

	"github.com/streadway/amqp"
)

type Consumer struct {
	conn          *amqp.Connection
	channel       *amqp.Channel
	connNotify    chan *amqp.Error
	channelNotify chan *amqp.Error
	done          chan struct{}
	addr          string
	exchange      string
	queue         string
	routingKey    string
	consumerTag   string
	autoDelete    bool
	handler       func([]byte) error
	delivery      <-chan amqp.Delivery
}

// NewConsumer 创建消费者
func NewConsumer(addr, exchange, queue, routingKey string, handler func([]byte) error) *Consumer {
	hostname, _ := os.Hostname()
	return &Consumer{
		addr:        addr,
		exchange:    exchange,
		queue:       queue,
		routingKey:  routingKey,
		consumerTag: hostname,
		autoDelete:  false,
		handler:     handler,
		done:        make(chan struct{}),
	}
}

func (c *Consumer) Start() error {
	if err := c.Run(); err != nil {
		return err
	}
	go c.ReConnect()
	return nil
}

func (c *Consumer) Stop() {
	close(c.done)

	if !c.conn.IsClosed() {
		// 关闭 SubMsg message delivery
		if err := c.channel.Cancel(c.consumerTag, true); err != nil {
			log.Println("rabbitmq consumer - channel cancel failed: ", err)
		}

		if err := c.conn.Close(); err != nil {
			log.Println("rabbitmq consumer - connection close failed: ", err)
		}
	}
}

func (c *Consumer) Run() (err error) {
	if c.conn, err = amqp.Dial(c.addr); err != nil {
		return err
	}

	if c.channel, err = c.conn.Channel(); err != nil {
		c.conn.Close()
		return err
	}

	defer func() {
		if err != nil {
			c.channel.Close()
			c.conn.Close()
		}
	}()

	// 声明一个主要使用的 exchange
	err = c.channel.ExchangeDeclare(
		c.exchange,
		"x-delayed-message",
		true,
		c.autoDelete,
		false,
		false,
		amqp.Table{
			"x-delayed-type": "fanout",
		})
	if err != nil {
		return err
	}

	// 声明一个延时队列, 延时消息就是要发送到这里
	q, err := c.channel.QueueDeclare(c.queue, false, c.autoDelete, false, false, nil)
	if err != nil {
		return err
	}

	err = c.channel.QueueBind(q.Name, "", c.exchange, false, nil)
	if err != nil {
		return err
	}

	c.delivery, err = c.channel.Consume(
		q.Name, c.consumerTag, false, false, false, false, nil)
	if err != nil {
		return err
	}

	go c.Handle()

	c.connNotify = c.conn.NotifyClose(make(chan *amqp.Error))
	c.channelNotify = c.channel.NotifyClose(make(chan *amqp.Error))
	return
}

func (c *Consumer) ReConnect() {
	for {
		select {
		case err := <-c.connNotify:
			if err != nil {
				log.Println("rabbitmq consumer - connection NotifyClose: ", err)
			}
		case err := <-c.channelNotify:
			if err != nil {
				log.Println("rabbitmq consumer - channel NotifyClose: ", err)
			}
		case <-c.done:
			return
		}

		// backstop
		if !c.conn.IsClosed() {
			// close message delivery
			if err := c.channel.Cancel(c.consumerTag, true); err != nil {
				log.Println("rabbitmq consumer - channel cancel failed: ", err)
			}

			if err := c.conn.Close(); err != nil {
				log.Println("rabbitmq consumer - channel cancel failed: ", err)
			}
		}

		// IMPORTANT: 必须清空 Notify，否则死连接不会释放
		for err := range c.channelNotify {
			log.Println(err)
		}
		for err := range c.connNotify {
			log.Println(err)
		}

	quit:
		for {
			select {
			case <-c.done:
				return
			default:
				log.Println("rabbitmq consumer - reconnect")

				if err := c.Run(); err != nil {
					log.Println("rabbitmq consumer - failCheck: ", err)
					// sleep 15s reconnect
					time.Sleep(time.Second * 15)
					continue
				}
				break quit
			}
		}
	}
}

func (c *Consumer) Handle() {
	for d := range c.delivery {
		go func(delivery amqp.Delivery) {
			if err := c.handler(delivery.Body); err != nil {
				// 重新入队，否则未确认的消息会持续占用内存，这里的操作取决于你的实现，你可以当出错之后并直接丢弃也是可以的
				_ = delivery.Reject(true)
			} else {
				_ = delivery.Ack(false)
			}
		}(d)
	}
}
```

