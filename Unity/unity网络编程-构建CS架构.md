## 📋 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                  客户端 (Unity)                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │   UI层     │→ │  网络层    │→ │  协议层    │        │
│  │ LoginUI.cs │  │NetworkMgr  │  │  Packets   │        │
│  └────────────┘  └────────────┘  └────────────┘        │
└─────────────────────────────────────────────────────────┘
                         ↕ TCP Socket
┌─────────────────────────────────────────────────────────┐
│                  服务器 (C# Console)                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │  网络层    │→ │  业务层    │→ │  数据层    │        │
│  │   Session  │  │  Service   │  │ Repository │        │
│  └────────────┘  └────────────┘  └────────────┘        │
└─────────────────────────────────────────────────────────┘
```

---

## 🏗️ 开发流程

### 阶段一：设计协议（共享层）

#### 1. 定义协议类型

```csharp
public enum PacketType : byte
{
    Login = 1,           // 登录
    Register = 2,        // 注册
    Heartbeat = 3,       // 心跳
    AddFriend = 4,       // 添加好友
    SendMessage = 5      // 发送消息
}

public enum ResponseCode : byte
{
    Success = 0,         // 成功
    Failed = 1,          // 失败
    InvalidData = 2,     // 无效数据
    UserExists = 3,      // 用户已存在
    UserNotFound = 4,    // 用户不存在
    WrongPassword = 5    // 密码错误
}
```

#### 2. 设计二进制协议结构

```
完整数据包：
┌────────────┬──────────┬───────────────┐
│ 长度(4字节) │ 类型(1字节)│   包体数据    │
└────────────┴──────────┴───────────────┘

登录请求包体：
┌────────────┬────────┬────────────┬────────┐
│用户名长度  │ 用户名 │ 密码长度   │  密码  │
│   (2字节)  │ (变长) │  (2字节)   │ (变长) │
└────────────┴────────┴────────────┴────────┘
```

#### 3. 实现序列化工具

```csharp
public static class BinarySerializer
{
    // 写入字符串（带长度前缀）
    public static void WriteString(BinaryWriter writer, string value)
    {
        if (string.IsNullOrEmpty(value))
        {
            writer.Write((ushort)0);
            return;
        }
        byte[] bytes = Encoding.UTF8.GetBytes(value);
        writer.Write((ushort)bytes.Length);  // 先写长度
        writer.Write(bytes);                 // 再写内容
    }

    // 读取字符串
    public static string ReadString(BinaryReader reader)
    {
        ushort length = reader.ReadUInt16();
        if (length == 0) return string.Empty;
        byte[] bytes = reader.ReadBytes(length);
        return Encoding.UTF8.GetString(bytes);
    }
}
```

#### 4. 实现包基类

```csharp
public abstract class Packet
{
    public PacketType Type { get; set; }

    // 序列化为字节数组
    public byte[] ToBytes()
    {
        using (var ms = new MemoryStream())
        using (var writer = new BinaryWriter(ms))
        {
            writer.Write((byte)Type);
            WriteBody(writer);
            byte[] body = ms.ToArray();

            // 添加长度头
            using (var finalMs = new MemoryStream())
            using (var finalWriter = new BinaryWriter(finalMs))
            {
                finalWriter.Write(body.Length);
                finalWriter.Write(body);
                return finalMs.ToArray();
            }
        }
    }

    // 子类实现具体序列化
    protected abstract void WriteBody(BinaryWriter writer);
    protected abstract void ReadBody(BinaryReader reader);
}
```

#### 5. 实现具体数据包

```csharp
// 登录包
public class LoginPacket : Packet
{
    public string Username;
    public string Password;

    public LoginPacket() { Type = PacketType.Login; }

    protected override void WriteBody(BinaryWriter writer)
    {
        BinarySerializer.WriteString(writer, Username);
        BinarySerializer.WriteString(writer, Password);
    }

    protected override void ReadBody(BinaryReader reader)
    {
        Username = BinarySerializer.ReadString(reader);
        Password = BinarySerializer.ReadString(reader);
    }
}

// 响应包
public class ResponsePacket : Packet
{
    public ResponseCode Code;
    public string Message;
    public int UserId;

    protected override void WriteBody(BinaryWriter writer)
    {
        writer.Write((byte)Code);
        BinarySerializer.WriteString(writer, Message);
        writer.Write(UserId);
    }

    protected override void ReadBody(BinaryReader reader)
    {
        Code = (ResponseCode)reader.ReadByte();
        Message = BinarySerializer.ReadString(reader);
        UserId = reader.ReadInt32();
    }
}
```

---

### 阶段二：服务器端开发

#### 1. 数据模型（Models/User.cs）

```csharp
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string PasswordHash { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? LastLoginAt { get; set; }
}
```

#### 2. 数据访问层（Repository）

```csharp
public interface IUserRepository
{
    void Add(User user);
    void Update(User user);
    User GetById(int id);
    User GetByUsername(string username);
}

public class UserRepository : IUserRepository
{
    private Dictionary<int, User> _users = new Dictionary<int, User>();
    private Dictionary<string, int> _usernameIndex = new Dictionary<string, int>();
    private int _nextId = 1;
    private object _lock = new object();

    public void Add(User user)
    {
        lock (_lock)
        {
            user.Id = _nextId++;
            _users[user.Id] = user;
            _usernameIndex[user.Username.ToLower()] = user.Id;
        }
    }

    public User GetByUsername(string username)
    {
        lock (_lock)
        {
            var key = username.ToLower();
            if (_usernameIndex.ContainsKey(key))
            {
                return _users[_usernameIndex[key]];
            }
            return null;
        }
    }
    
    // ... 其他方法
}
```

#### 3. 业务逻辑层（Service）

```csharp
public interface IUserService
{
    ResponsePacket Register(string username, string password);
    ResponsePacket Login(string username, string password);
}

public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;

    public UserService(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }

    public ResponsePacket Register(string username, string password)
    {
        // 1. 验证输入
        if (string.IsNullOrWhiteSpace(username) || string.IsNullOrWhiteSpace(password))
        {
            return new ResponsePacket
            {
                Code = ResponseCode.InvalidData,
                Message = "用户名和密码不能为空"
            };
        }

        // 2. 检查用户是否存在
        if (_userRepository.GetByUsername(username) != null)
        {
            return new ResponsePacket
            {
                Code = ResponseCode.UserExists,
                Message = "用户名已存在"
            };
        }

        // 3. 创建用户
        var user = new User
        {
            Username = username,
            PasswordHash = HashPassword(password),
            CreatedAt = DateTime.Now
        };

        _userRepository.Add(user);

        return new ResponsePacket
        {
            Code = ResponseCode.Success,
            Message = "注册成功"
        };
    }

    public ResponsePacket Login(string username, string password)
    {
        var user = _userRepository.GetByUsername(username);
        if (user == null)
        {
            return new ResponsePacket
            {
                Code = ResponseCode.UserNotFound,
                Message = "用户不存在"
            };
        }

        if (!VerifyPassword(password, user.PasswordHash))
        {
            return new ResponsePacket
            {
                Code = ResponseCode.WrongPassword,
                Message = "密码错误"
            };
        }

        user.LastLoginAt = DateTime.Now;
        _userRepository.Update(user);

        return new ResponsePacket
        {
            Code = ResponseCode.Success,
            Message = "登录成功",
            UserId = user.Id
        };
    }

    private string HashPassword(string password)
    {
        using (var sha256 = SHA256.Create())
        {
            var bytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(password));
            return Convert.ToBase64String(bytes);
        }
    }

    private bool VerifyPassword(string password, string hash)
    {
        return HashPassword(password) == hash;
    }
}
```

#### 4. 客户端会话（ClientSession）

```csharp
public class ClientSession
{
    private readonly TcpClient _client;
    private readonly NetworkStream _stream;
    private readonly IUserService _userService;
    private bool _isRunning;

    public string SessionId { get; }
    public int? UserId { get; private set; }

    public ClientSession(TcpClient client, IUserService userService)
    {
        _client = client;
        _stream = client.GetStream();
        _userService = userService;
        SessionId = Guid.NewGuid().ToString();
        _isRunning = true;
    }

    public void Start()
    {
        try
        {
            while (_isRunning && _client.Connected)
            {
                // 1. 读取包头（4字节长度）
                byte[] lengthBytes = new byte[4];
                ReadExactly(_stream, lengthBytes, 4);
                int length = BitConverter.ToInt32(lengthBytes, 0);

                // 2. 读取包体
                byte[] packetData = new byte[length];
                ReadExactly(_stream, packetData, length);

                // 3. 处理数据包
                HandlePacket(packetData);
            }
        }
        catch (Exception ex)
        {
            Logger.Error($"会话错误: {ex.Message}");
        }
        finally
        {
            Close();
        }
    }

    private void HandlePacket(byte[] data)
    {
        using (var ms = new MemoryStream(data))
        using (var reader = new BinaryReader(ms))
        {
            // 读取包类型
            PacketType type = (PacketType)reader.ReadByte();
            
            ResponsePacket response;

            // 根据类型分发处理
            switch (type)
            {
                case PacketType.Login:
                    var loginPacket = ParsePacket<LoginPacket>(data);
                    response = _userService.Login(
                        loginPacket.Username, 
                        loginPacket.Password
                    );
                    if (response.Code == ResponseCode.Success)
                    {
                        UserId = response.UserId;
                    }
                    break;

                case PacketType.Register:
                    var registerPacket = ParsePacket<RegisterPacket>(data);
                    response = _userService.Register(
                        registerPacket.Username, 
                        registerPacket.Password
                    );
                    break;

                default:
                    response = new ResponsePacket
                    {
                        Code = ResponseCode.Failed,
                        Message = "未知请求类型"
                    };
                    break;
            }

            response.Type = type;
            SendPacket(response);
        }
    }

    private void SendPacket(Packet packet)
    {
        byte[] data = packet.ToBytes();
        _stream.Write(data, 0, data.Length);
    }

    private void ReadExactly(NetworkStream stream, byte[] buffer, int count)
    {
        int totalRead = 0;
        while (totalRead < count)
        {
            int read = stream.Read(buffer, totalRead, count - totalRead);
            if (read == 0) throw new Exception("连接断开");
            totalRead += read;
        }
    }

    private T ParsePacket<T>(byte[] data) where T : Packet, new()
    {
        using (var ms = new MemoryStream(data))
        using (var reader = new BinaryReader(ms))
        {
            var packet = new T();
            packet.Type = (PacketType)reader.ReadByte();
            packet.ReadBody(reader);
            return packet;
        }
    }

    public void Close()
    {
        _isRunning = false;
        _stream?.Close();
        _client?.Close();
    }
}
```

#### 5. 服务器主类（GameServer）

```csharp
public class GameServer
{
    private readonly ServerConfig _config;
    private readonly IUserService _userService;
    private TcpListener _listener;
    private List<ClientSession> _sessions;
    private bool _isRunning;

    public GameServer(ServerConfig config)
    {
        _config = config;
        _sessions = new List<ClientSession>();
        
        // 依赖注入
        var userRepository = new UserRepository();
        _userService = new UserService(userRepository);
    }

    public void Start()
    {
        _listener = new TcpListener(IPAddress.Any, _config.Port);
        _listener.Start();
        _isRunning = true;

        Logger.Info($"服务器启动在端口 {_config.Port}");

        // 开启接受连接的线程
        Thread acceptThread = new Thread(AcceptClients);
        acceptThread.Start();
    }

    private void AcceptClients()
    {
        while (_isRunning)
        {
            try
            {
                // 阻塞等待客户端连接
                TcpClient client = _listener.AcceptTcpClient();

                // 检查连接数
                if (_sessions.Count >= _config.MaxConnections)
                {
                    Logger.Warning("达到最大连接数");
                    client.Close();
                    continue;
                }

                // 创建会话
                var session = new ClientSession(client, _userService);
                _sessions.Add(session);

                // 为每个会话创建线程
                Thread sessionThread = new Thread(session.Start);
                sessionThread.Start();

                Logger.Info($"新连接，当前: {_sessions.Count}");
            }
            catch (Exception ex)
            {
                if (_isRunning)
                {
                    Logger.Error($"接受连接失败: {ex.Message}");
                }
            }
        }
    }

    public void Stop()
    {
        _isRunning = false;
        
        foreach (var session in _sessions)
        {
            session.Close();
        }
        _sessions.Clear();
        
        _listener?.Stop();
        Logger.Info("服务器已停止");
    }
}
```

#### 6. 程序入口（Program.cs）

```csharp
class Program
{
    static void Main(string[] args)
    {
        var config = new ServerConfig
        {
            Port = 7777,
            MaxConnections = 100
        };

        var server = new GameServer(config);
        server.Start();

        Console.WriteLine("按任意键停止服务器...");
        Console.ReadKey();

        server.Stop();
    }
}

public class ServerConfig
{
    public int Port { get; set; } = 7777;
    public int MaxConnections { get; set; } = 100;
}
```

---

### 阶段三：Unity 客户端开发

#### 1. 网络管理器（NetworkManager.cs）

```csharp
public class NetworkManager : MonoBehaviour
{
    private static NetworkManager instance;
    public static NetworkManager Instance => instance;

    private TcpClient client;
    private NetworkStream stream;

    public string serverIP = "127.0.0.1";
    public int serverPort = 7777;

    void Awake()
    {
        if (instance == null)
        {
            instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }

    public bool Connect()
    {
        try
        {
            client = new TcpClient();
            client.Connect(serverIP, serverPort);
            stream = client.GetStream();
            Debug.Log("连接成功");
            return true;
        }
        catch (Exception ex)
        {
            Debug.LogError($"连接失败: {ex.Message}");
            return false;
        }
    }

    public void SendLogin(string username, string password, Action<ResponsePacket> callback)
    {
        var packet = new LoginPacket
        {
            Username = username,
            Password = password
        };
        SendPacket(packet, callback);
    }

    private void SendPacket(Packet packet, Action<ResponsePacket> callback)
    {
        if (client == null || !client.Connected)
        {
            if (!Connect())
            {
                callback?.Invoke(new ResponsePacket
                {
                    Code = ResponseCode.Failed,
                    Message = "无法连接服务器"
                });
                return;
            }
        }

        new Thread(() =>
        {
            try
            {
                // 发送
                byte[] data = packet.ToBytes();
                stream.Write(data, 0, data.Length);

                // 接收
                byte[] lengthBytes = new byte[4];
                ReadExactly(stream, lengthBytes, 4);
                int length = BitConverter.ToInt32(lengthBytes, 0);

                byte[] body = new byte[length];
                ReadExactly(stream, body, length);

                var response = ParseResponse(body);

                // 回主线程
                MainThreadDispatcher.Instance.Enqueue(() =>
                {
                    callback?.Invoke(response);
                });
            }
            catch (Exception ex)
            {
                Debug.LogError($"通信错误: {ex.Message}");
            }
        }).Start();
    }

    private void ReadExactly(NetworkStream stream, byte[] buffer, int count)
    {
        int totalRead = 0;
        while (totalRead < count)
        {
            int read = stream.Read(buffer, totalRead, count - totalRead);
            if (read == 0) throw new Exception("连接断开");
            totalRead += read;
        }
    }

    private ResponsePacket ParseResponse(byte[] data)
    {
        using (var ms = new MemoryStream(data))
        using (var reader = new BinaryReader(ms))
        {
            var response = new ResponsePacket();
            response.Type = (PacketType)reader.ReadByte();
            response.ReadBody(reader);
            return response;
        }
    }
}
```

#### 2. UI 层（LoginUI.cs）

```csharp
public class LoginUI : MonoBehaviour
{
    public TMP_InputField usernameInput;
    public TMP_InputField passwordInput;
    public Button loginButton;
    public TextMeshProUGUI messageText;

    void Start()
    {
        loginButton.onClick.AddListener(OnLogin);
    }

    void OnLogin()
    {
        string username = usernameInput.text.Trim();
        string password = passwordInput.text.Trim();

        if (string.IsNullOrEmpty(username) || string.IsNullOrEmpty(password))
        {
            ShowMessage("用户名和密码不能为空", Color.red);
            return;
        }

        ShowMessage("登录中...", Color.yellow);

        NetworkManager.Instance.SendLogin(username, password, (response) =>
        {
            if (response.Code == ResponseCode.Success)
            {
                ShowMessage($"登录成功! ID:{response.UserId}", Color.green);
            }
            else
            {
                ShowMessage(response.Message, Color.red);
            }
        });
    }

    void ShowMessage(string msg, Color color)
    {
        messageText.text = msg;
        messageText.color = color;
    }
}
```

---

## 🔄 完整通信流程

```
1. 客户端：用户点击登录
   ↓
2. LoginUI：收集用户名密码
   ↓
3. NetworkManager：创建 LoginPacket
   ↓
4. LoginPacket.ToBytes()：序列化为二进制
   [4字节长度][1字节类型][用户名][密码]
   ↓
5. TCP发送到服务器
   ↓
6. 服务器：GameServer.AcceptClients() 接受连接
   ↓
7. ClientSession：为该连接创建会话
   ↓
8. ClientSession.Start()：读取数据包
   - 读4字节长度
   - 读包体数据
   ↓
9. ClientSession.HandlePacket()：
   - 读取包类型 → Login
   - 解析 LoginPacket
   ↓
10. UserService.Login()：
    - 验证输入
    - 查询用户 → UserRepository
    - 验证密码
    - 更新登录时间
    - 返回 ResponsePacket
   ↓
11. ClientSession.SendPacket()：
    - ResponsePacket.ToBytes() 序列化
    - TCP发送回客户端
   ↓
12. NetworkManager：接收响应
    - 读4字节长度
    - 读包体
    - 解析 ResponsePacket
   ↓
13. MainThreadDispatcher：切回主线程
   ↓
14. LoginUI：显示结果（成功/失败）
```

---

## ✅ 核心要点总结

### 1. 分层架构
```
表现层（UI）→ 网络层（Network）→ 业务层（Service）→ 数据层（Repository）
```

### 2. 线程模型
```
Main线程 → Accept线程 → Session线程1、2、3...N
```

### 3. 协议设计
- 使用二进制协议（性能 + 安全）
- 协议代码客户端服务器共享
- 每种功能一个 Packet 类

### 4. 依赖注入
```
GameServer → UserService → UserRepository
通过接口解耦，便于测试和替换
```

### 5. 线程安全
- Repository 使用 `lock` 保护共享数据
- Unity 使用 MainThreadDispatcher 回主线程

### 6. 扩展新功能
1. 添加新的 PacketType
2. 创建新的 Packet 类
3. 创建新的 Service
4. 在 Session 的 switch 中添加处理
