# 活动追踪器合约

本教程通过构建一个健身追踪器来演示Solidity事件的强大功能。涵盖了结构体、映射、事件声明,以及触发事件以实现实时前端更新和里程碑追踪。

## 关键概念

### 结构体定义
```solidity
struct UserProfile {
    string name;
    uint256 weight;
    bool isRegistered;
}

struct WorkoutActivity {
    string activityType;
    uint256 duration;
    uint256 distance;
    uint256 timestamp;
}
```

### 状态变量
```solidity
mapping(address => UserProfile) public userProfiles;
mapping(address => WorkoutActivity[]) private workoutHistory;
mapping(address => uint256) public totalWorkouts;
mapping(address => uint256) public totalDistance;
```

### 事件
```solidity
event UserRegistered(address indexed userAddress, string name, uint256 timestamp);
event ProfileUpdated(address indexed userAddress, uint256 newWeight, uint256 timestamp);
event WorkoutLogged(address indexed userAddress, string activityType, uint256 duration, uint256 distance, uint256 timestamp);
event MilestoneAchieved(address indexed userAddress, string milestone, uint256 timestamp);
```

### 修饰符
```solidity
modifier onlyRegistered() {
    require(userProfiles[msg.sender].isRegistered, "User not registered");
    _;
}
```

### 用户注册
```solidity
function registerUser(string memory _name, uint256 _weight) public {
    require(!userProfiles[msg.sender].isRegistered, "Already registered");
    
    userProfiles[msg.sender] = UserProfile({
        name: _name,
        weight: _weight,
        isRegistered: true
    });
    
    emit UserRegistered(msg.sender, _name, block.timestamp);
}
```

### 更新资料
```solidity
function updateWeight(uint256 _newWeight) public onlyRegistered {
    UserProfile storage profile = userProfiles[msg.sender];
    profile.weight = _newWeight;
    emit ProfileUpdated(msg.sender, _newWeight, block.timestamp);
}
```

### 记录锻炼
```solidity
function logWorkout(string memory _activityType, uint256 _duration, uint256 _distance) public onlyRegistered {
    WorkoutActivity memory newWorkout = WorkoutActivity({
        activityType: _activityType,
        duration: _duration,
        distance: _distance,
        timestamp: block.timestamp
    });
    
    workoutHistory[msg.sender].push(newWorkout);
    totalWorkouts[msg.sender]++;
    totalDistance[msg.sender] += _distance;
    
    emit WorkoutLogged(msg.sender, _activityType, _duration, _distance, block.timestamp);
    
    // 检查里程碑
    if (totalWorkouts[msg.sender] == 10) {
        emit MilestoneAchieved(msg.sender, "10 workouts completed!", block.timestamp);
    }
    
    if (totalDistance[msg.sender] >= 100) {
        emit MilestoneAchieved(msg.sender, "100km distance achieved!", block.timestamp);
    }
}
```

### 查询函数
```solidity
function getUserWorkoutCount() public view onlyRegistered returns (uint256) {
    return totalWorkouts[msg.sender];
}
```

## 核心知识点

- **结构体**: 组织相关数据的自定义类型
- **数组映射**: `mapping(address => WorkoutActivity[])` 存储每个用户的多条记录
- **事件**: 用于通知前端和记录链上活动
- `indexed` 参数可以被过滤
- `storage` vs `memory`: storage引用状态变量,memory是临时的
- `block.timestamp`: 当前区块时间戳
- 事件是连接智能合约和前端应用的桥梁

