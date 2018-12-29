# MQ PHP HTTP SDK  
Alyun MQ Documents: http://www.aliyun.com/product/ons

Aliyun MQ Console: https://ons.console.aliyun.com  

## Intall Composer

To install composer by following commands, or see [composer](https://docs.phpcomposer.com/00-intro.html)
```bash
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
```
## Install & Use

Add require to your `composer.json`
```json
{
  "require": {
     "aliyunmq/mq-http-sdk": ">=1.0.0"
  }
}
```
Use Composer to install requires
```bash
composer install
``` 

## Samples

You must fulfill the AccessId/AccessKey/Endpoint in the example before running.   

### Publish Message
```php
<?php

require "vendor/autoload.php";

use MQ\Model\TopicMessage;
use MQ\MQClient;

class ProducerTest
{
    private $client;
    private $producer;

    public function __construct()
    {
        $this->client = new MQClient(
            // 设置HTTP接入域名（此处以公共云生产环境为例）
            "${HTTP_ENDPOINT}",
            // AccessKey 阿里云身份验证，在阿里云服务器管理控制台创建
            "${ACCESS_KEY}",
            // SecretKey 阿里云身份验证，在阿里云服务器管理控制台创建
            "${SECRET_KEY}"
        );

        // 所属的 Topic
        $topic = "${TOPIC}";
        // Topic所属实例ID，默认实例为空NULL
        $instanceId = "${INSTANCE_ID}";

        $this->producer = $this->client->getProducer($instanceId, $topic);
    }

    public function run()
    {
        try
        {
            for ($i=1; $i<=100; $i++)
            {
                $publishMessage = new TopicMessage(
                    "xxxxxxxx"// 消息内容
                );
                $result = $this->producer->publishMessage($publishMessage);

                print "Send mq message success. msgId is:" . $result->getMessageId() . ", bodyMD5 is:" . $result->getMessageBodyMD5() . "\n";
            }
        } catch (\Exception $e) {
            print_r($e->getMessage() . "\n");
        }
    }
}


$instance = new ProducerTest();
$instance->run();

?>

```

### Consume Message
```php
<?php

require "vendor/autoload.php";

use MQ\Model\TopicMessage;
use MQ\MQClient;

class ConsumerTest
{
    private $client;
    private $consumer;

    public function __construct()
    {
        $this->client = new MQClient(
            // 设置HTTP接入域名（此处以公共云生产环境为例）
            "${HTTP_ENDPOINT}",
            // AccessKey 阿里云身份验证，在阿里云服务器管理控制台创建
            "${ACCESS_KEY}",
            // SecretKey 阿里云身份验证，在阿里云服务器管理控制台创建
            "${SECRET_KEY}"
        );

        // 所属的 Topic
        $topic = "${TOPIC}";
        // 您在控制台创建的 Consumer ID(Group ID)
        $groupId = "${GROUP_ID}";
        // Topic所属实例ID，默认实例为空NULL
        $instanceId = "${INSTANCE_ID}";

        $this->consumer = $this->client->getConsumer($instanceId, $topic, $groupId);
    }

    public function run()
    {
        // 在当前线程循环消费消息，建议是多开个几个线程并发消费消息
        while (True) {
            try {
                // 长轮询消费消息
                // 长轮询表示如果topic没有消息则请求会在服务端挂住3s，3s内如果有消息可以消费则立即返回
                $messages = $this->consumer->consumeMessage(
                    3, // 一次最多消费3条(最多可设置为16条)
                    3 // 长轮询时间3秒（最多可设置为30秒）
                );
            } catch (\Exception $e) {
                if ($e instanceof MQ\Exception\MessageNotExistException) {
                    // 没有消息可以消费，接着轮询
                    printf("No message, contine long polling!RequestId:%s\n", $e->getRequestId());
                    continue;
                }

                print_r($e->getMessage() . "\n");

                sleep(3);
                continue;
            }

            print "consume finish, messages:\n";

            // 处理业务逻辑
            $receiptHandles = array();
            foreach ($messages as $message) {
                $receiptHandles[] = $message->getReceiptHandle();
                printf("MessageID:%s TAG:%s BODY:%s \nPublishTime:%d, FirstConsumeTime:%d, \nConsumedTimes:%d, NextConsumeTime:%d\n",
                    $message->getMessageId(), $message->getMessageTag(), $message->getMessageBody(),
                    $message->getPublishTime(), $message->getFirstConsumeTime(), $message->getConsumedTimes(), $message->getNextConsumeTime());
            }

            // 5分钟之内若不确认消息消费成功，则消息会重复消费
            // 消息句柄有时间戳，同一条消息每次消费拿到的都不一样
            print_r($receiptHandles);
            try {
                $this->consumer->ackMessage($receiptHandles);
            } catch (\Exception $e) {
                if ($e instanceof MQ\Exception\AckMessageException) {
                    // 某些消息的句柄可能超时了会导致确认不成功
                    printf("Ack Error, RequestId:%s\n", $e->getRequestId());
                    foreach ($e->getAckMessageErrorItems() as $errorItem) {
                        printf("\tReceiptHandle:%s, ErrorCode:%s, ErrorMsg:%s\n", $errorItem->getReceiptHandle(), $errorItem->getErrorCode(), $errorItem->getErrorCode());
                    }
                }
            }
            print "ack finish\n";


        }

    }
}


$instance = new ConsumerTest();
$instance->run();

?>
```
