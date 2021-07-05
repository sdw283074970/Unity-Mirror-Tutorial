### 通过Steam进行联机的原理是什么？

通过Steam联机，相当于Steam作为一个协调服务器充当中转战，所有服务器主机与客户端的数据交换都会经过中转站转发。中转站可以理解为游戏房间或游戏大厅。

其实现的逻辑步骤可简化如下：

1. 主机向Steam发出建立房间或游戏大厅；
2. Steam受理请求并在Steam端尝试建立大厅，并回传结果到主机；
3. 如果建立成功，则Steam服务器中会储存大厅信息供以后调用并在主机端执行相关操作；
4. 其他玩家点击加入游戏，会向Steam发送加入游戏请求；
5. 如果成功加入游戏大厅，则客户端可获取所有大厅信息并在当前客户端执行相关操作，所有玩家和主机实现数据同步；
6. 开始游戏。

### 如何通过Steam进行联机？

使用Steam数据交换即可通过Steam联机。使用Steam数据交换需要用到Steam的联机套件且完成相关设置。分为以下几步：

1. 更新`Mirror`到最新的版本；
2. 在网上下载`FizzySteamworks`库并导入到项目中；
3. 在项目中的`NetworkManager`预制件中添加`Steam Manager`组件；
4. 在项目中的`NetworkManager`预制件中，将`Transport`组件替换成`Fizzy Steam Transport`组件，并在`NetworkManager`组件中替换数据交换引用；
5. 在相关脚本中（如游戏大厅脚本）添加Steam相关回调代码。

### 如何添加Steamworks代码到项目中？

找到目标脚本，一般是菜单脚本或大厅脚本。如在一个RTS示例中，将相关Steam联机代码添加到`MainMenu`脚本中。其原始代码如下：

```c#
public class MainMenu : MonoBehaviour
{
    [SerializeField]
    private GameObject mainMenucanvas = null;

    public void HostLobby()
    {
        if (useSteam)
        {
            SteamMatchmaking.CreateLobby(ELobbyType.k_ELobbyTypeFriendsOnly, 4);
            return;
        }

        NetworkManager.singleton.StartHost();
    }
}
```

对这代码进行改造后如下：

```c#
public class MainMenu : MonoBehaviour
{
    [SerializeField]
    private GameObject mainMenucanvas = null;

    // 添加是否使用steam进行联机的开关
    [SerializeField]
    private bool useSteam = false;

    // 三个核心回调
    protected Callback<LobbyCreated_t> lobbyCreated;  // 当游戏大厅建立好之后的回调
    protected Callback<GameLobbyJoinRequested_t> gameLobbyJoinRequested;  // 当加入游戏大厅请求发送后的回调
    protected Callback<LobbyEnter_t> lobbyEntered;  // 当成功进入游戏大厅的回调

    private void Start()
    {
        // 如果不是适用steam进行联机，不做任何执行
        if (!useSteam) { return; }
        
        // 当脚本载入后，以下代码会让steam服务器建立回调，并通过委托调用我们本地方法
        lobbyCreated = Callback<LobbyCreated_t>.Create(OnLobbyCreated);
        gameLobbyJoinRequested = Callback<GameLobbyJoinRequested_t>.Create(OnGameLobbyJoinRequested);
        lobbyEntered = Callback<LobbyEnter_t>.Create(OnLobbyEntered);
    }

    // 当在steam端游戏大厅建立后执行的方法
    private void OnLobbyCreated(LobbyCreated_t callback)
    {
        // 如果回调结果不OK，则说明建立大厅失败，执行失败后的操作
        if (callback.m_eResult != EResult.k_EResultOK)
        {
            // 回到主页面
            mainMenucanvas.SetActive(true);
            return;
        }
  
        // 如果回调结果OK，则启动主持主机
        NetworkManager.singleton.StartHost();

        // 由于使用了steam联机，当大厅建立成功后，需要设置大厅数据，数据一定要有包括主机玩地址，也可以有其他比如地图数据等等
        // 第一个参数是steam游戏大厅的id，在回调中有
        // 第二和第三个参数是一对key/value，用来广播有用数据（如大厅名字，地图信息等）给玩家，新加入的玩家会自动收到最新的更新
        // 通过steam加入的游戏，其主机地址为主机玩家的steamid
        SteamMatchmaking.SetLobbyData(new CSteamID(callback.m_ulSteamIDLobby), "HostAddress", SteamUser.GetSteamID().ToString());
    }

    // 当点击加入游戏后执行的方法
    private void OnGameLobbyJoinRequested(GameLobbyJoinRequested_t callback)
    {
        // 指明加入哪个游戏大厅，大厅信息在回调中，把大厅id作为参数直接使用JoinLobby()加入即可
        SteamMatchmaking.JoinLobby(callback.m_steamIDLobby);
    }

    // 当成功进入游戏大厅后的方法
    private void OnLobbyEntered(LobbyEnter_t callback)
    {
        // 如果加入大厅的是主机本身，不做操作
        if (NetworkServer.active) { return; }

        // 获取主机地址，该地址以键值对的形式储存在大厅数据中，直接取用
        var hostAddress = SteamMatchmaking.GetLobbyData(new CSteamID(callback.m_ulSteamIDLobby), "HostAddress");

        // 使用mirror中的网络管理器以客户端身份开始游戏
        NetworkManager.singleton.networkAddress = hostAddress;
        NetworkManager.singleton.StartClient();
        
        // 设置一些菜单状态
        mainMenucanvas.SetActive(false);
    }

    private void Awake()
    {
        SceneManager.sceneLoaded += OnSceneLoaded;
    }

    private void OnDestroy()
    {
        SceneManager.sceneLoaded -= OnSceneLoaded;
    }

    public void HostLobby()
    {
        // 如果使用了steam联机，向steam发出建立大厅的命令并等待其完成在steam服务的大厅建立
        if (useSteam)
        {
            // 第一个参数是游戏大厅类型的枚举（这里限定了好友加入），第二个参数是玩家人数
            // 当在steam服务器大厅建立完成后，会触发之前注册的回调
            SteamMatchmaking.CreateLobby(ELobbyType.k_ELobbyTypeFriendsOnly, 4);
            return;
        }

        NetworkManager.singleton.StartHost();
    }

    private void OnSceneLoaded(Scene scene, LoadSceneMode mode)
    {
        if (scene.name == "Scene_Menu")
            mainMenucanvas.SetActive(true);

        if (scene.name.StartsWith("Scene_Map"))
            mainMenucanvas.SetActive(false);
    }
}
```
