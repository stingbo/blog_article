* 工厂模式
	> 工厂设计模式提供获取某个对象的新实例的一个接口，同时使调用代码避免确定实际实例化基类的步骤。

    简述：假设，我们现在在获取微信端消息的通知，消息有许多类型，比如文本、图片、语音等，这些类型都包含在微信推送的内容里，需求是要把这些内容都保存在本地的数据库，我们来模拟一下，方法大概有两种：
    > 微信端消息的通知：用户向某服务号发送信息或触发了一些事件(扫码，上报地理位置等)，如果此服务号开启了开发者的一些设置，那么微信会把相关信息或事件推送给服务号设置的接收信息地址。


1. 获取到微信推送的内容后，判断出信息类型，然后实例化不同的对象来保存对应的信息。

    ```
    // 模拟获取信息
    $message = file_get_contents("php://input");
    switch ($message->type) {
        case 'link':
            $object = new Link();
            $object->save($message->content);
            break;
        case 'text':
            $object = new Text();
            $object->save($message->content);
            break;
        case 'image':
            $object = new Image();
            $object->save($message->content);
            break;
        case 'voice':
            $object = new Voice();
            $object->save($message->content);
            break;
        case 'video':
            $object = new Video();
            $object->save($message->content);
            break;
        case 'location':
            $object = new Location();
            $object->save($message->content);
            break;
        default:
            # code...
        break;
    }
    ```

2. 使用工厂模式，可以有助于减少主代码流中基于条件的复杂性。

    ```
    // 模拟获取信息
    $message = file_get_contents("php://input");
    class MessageFactory {
        public static function create($type) {
            $class = ucfirst(strtolower($type));

            return new $class;
        }
    }

    $objcet = MessageFactory::create($message->type);
    $object->save($message->content);
    ```

    可以看到，第二种方式明显比第一种更简洁，不需要做过多的判断。另外，如果微信端增加了新的信息类型，我们在接收时也不需要再增加判断语句，只要增加一个处理对应类型的类即可。