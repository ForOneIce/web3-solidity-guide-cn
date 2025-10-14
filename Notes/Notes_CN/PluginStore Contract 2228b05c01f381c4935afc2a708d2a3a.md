# 插件商店合约

本教程介绍了使用 `call`、`delegatecall` 和 `staticcall` 进行合约间通信的模块化智能合约设计。构建了一个 `PluginStore`(核心配置文件合约)和两个插件(`AchievementsPlugin`、`WeaponStorePlugin`)来演示动态功能集成。

## 关键概念

### 核心合约: PluginStore
```solidity
struct PlayerProfile {
    string name;
    string avatar;
}

mapping(address => PlayerProfile) public profiles;
mapping(string => address) public plugins;
```

### 设置配置文件
```solidity
function setProfile(string memory _name, string memory _avatar) external {
    profiles[msg.sender] = PlayerProfile({
        name: _name,
        avatar: _avatar
    });
}

function getProfile(address user) external view returns (string memory, string memory) {
    PlayerProfile memory profile = profiles[user];
    return (profile.name, profile.avatar);
}
```

### 注册插件
```solidity
function registerPlugin(string memory key, address pluginAddress) external {
    plugins[key] = pluginAddress;
}

function getPlugin(string memory key) external view returns (address) {
    return plugins[key];
}
```

### 运行插件(使用 call)
```solidity
function runPlugin(
    string memory key,
    string memory functionSignature,
    address user,
    string memory argument
) external {
    address plugin = plugins[key];
    require(plugin != address(0), "Plugin not found");
    
    bytes memory data = abi.encodeWithSignature(functionSignature, user, argument);
    (bool success, ) = plugin.call(data);
    require(success, "Plugin call failed");
}
```

### 查询插件(使用 staticcall)
```solidity
function runPluginView(
    string memory key,
    string memory functionSignature,
    address user
) external view returns (string memory) {
    address plugin = plugins[key];
    require(plugin != address(0), "Plugin not found");
    
    bytes memory data = abi.encodeWithSignature(functionSignature, user);
    (bool success, bytes memory result) = plugin.staticcall(data);
    require(success, "Plugin call failed");
    
    return abi.decode(result, (string));
}
```

### 成就插件
```solidity
contract AchievementsPlugin {
    mapping(address => string) public latestAchievement;
    
    function setAchievement(address user, string memory achievement) public {
        latestAchievement[user] = achievement;
    }
    
    function getAchievement(address user) public view returns (string memory) {
        return latestAchievement[user];
    }
}
```

### 武器商店插件
```solidity
contract WeaponStorePlugin {
    mapping(address => string) public equippedWeapon;
    
    function setWeapon(address user, string memory weapon) public {
        equippedWeapon[user] = weapon;
    }
    
    function getWeapon(address user) public view returns (string memory) {
        return equippedWeapon[user];
    }
}
```

## 核心知识点

### 低级调用类型

1. **call**: 
   - 调用其他合约的函数
   - 可以修改被调用合约的状态
   - 语法: `address.call(data)`

2. **staticcall**:
   - 只读调用,不能修改状态
   - 用于 `view` 函数
   - 语法: `address.staticcall(data)`

3. **delegatecall**:
   - 在调用者的上下文中执行代码
   - 修改调用者的存储
   - 用于代理模式和升级

### ABI编码

- `abi.encodeWithSignature("functionName(type1,type2)", arg1, arg2)`
- 创建调用数据
- `abi.decode(data, (returnType))` 解析返回数据

### 模块化设计

- 核心合约管理基础数据
- 插件提供扩展功能
- 动态注册和调用
- 关注点分离

### 使用场景

- 游戏系统(物品、成就、技能)
- DeFi协议(可升级模块)
- DAO治理(插件式投票系统)
- 市场平台(第三方功能)

