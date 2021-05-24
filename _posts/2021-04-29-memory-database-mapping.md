---
layout:     post
title:      "服务器框架（一） 游戏内存与数据库的映射"
subtitle:   "" 
date:   2021-04-29
author: "qijun"
tags:
    - game server
---


### 映射关系
对象 <-> 内存数组 <-> mysql

例如： Helper.GM_Cmd <-> Cache <-> knight_game0.gm_cmd 

### 映射的字段类型
包含'int'、~~'char'~~、'text'字符串的字段类型

如：**INT**(10)、BIG**INT**(20)、**TEXT**、MEDIUM**TEXT**

TEXT、MEDIUMTEXT字段名必须以'str_'开头

TableDef.cs中定义的枚举，如未在CreateAllDBTable.cs声明与数据库的映射关系，为内存字段

内存表以'_'开头,如 _npc

### 核心类

**Cache**： 对应数据库，m_vecDataCache[]缓存所有表的集合，默认支持1024张表，修改TABLE_RESERVE_NUM生效

**MyDataTable**： 对应表，Dictionary<ID, int[]> m_dicDataGrid缓存整张表数据

**CGrid**： 提供字段的查询与修改操作

**CIDxMgr**： 字段在内存数据中的索引管理器，配合CGrid一起工作

### 以gm_cmd表举例

字段名称定义:
```csharp
// TableDef.cs

public enum gm_cmd
{
    id = 1,
    str_cmd,            // 命令
    time,           // 时间
    state,          // 状态
    str_result,     // 给平台的返回值
    str_run_time,       // 待执行时间
    the_enum_end,
}
```
id作为主键，枚举值为1；枚举以the_enum_end结尾。这两点不可变

映射关系声明：
```csharp
// CreateAllDBTable.cs

tinfo = MysqlHelper.ReadInstance().InitBaseDbInfo(TABLE.gm_cmd.ToString(), true, "GM命令",true);
tinfo.listcln.Add(new DBTableColumnsInfo(gm_cmd.str_cmd.ToString(), "TEXT", "''", false, "命令"));
tinfo.listcln.Add(new DBTableColumnsInfo(gm_cmd.time.ToString(), "BIGINT(20)", "0", false, "时间"));
tinfo.listcln.Add(new DBTableColumnsInfo(gm_cmd.state.ToString(), "INT(10) UNSIGNED", "0", true, "状态"));
tinfo.listcln.Add(new DBTableColumnsInfo(gm_cmd.str_result.ToString(), "TEXT", "''", false, "给平台的返回值"));
tinfo.listcln.Add(new DBTableColumnsInfo(gm_cmd.str_run_time.ToString(), "TEXT", "''", false, "GM执行时间"));
```
mysql中数据：
```mysql
mysql> select * from gm_cmd where id = 1;
+----+-------------------------------------------------+------+-------+------------+--------------+
| id | str_cmd                                         | time | state | str_result | str_run_time |
+----+-------------------------------------------------+------+-------+------------+--------------+
|  1 | addmail,201900000002,201900000002,390,0,39501,1 |    0 |     1 | NULL       | NULL         |
+----+-------------------------------------------------+------+-------+------------+--------------+
1 row in set (0.00 sec)
```

代码查询字段值：
```csharp
//查询整型字段，相当于 'select state from gm_cmd where id = 1;'
m_mod.GetData((int)TABLE.gm_cmd, 1, (int)gm_cmd.state);

//查询字符型字段，相当于 'select str_cmd from gm_cmd where id = 1;'
m_mod.GetStr((int)TABLE.gm_cmd, 1, (int)gm_cmd.str_cmd);
```

查询过程图解：
![cache](/img/game-server/cache.bmp)

gm_cmd表在Cache中的下标由TABLE枚举定义

int[]数组存储整型字段的值和字符型数据的键，第0位和最后一位不使用。

- int类型数据，可以直接存入数据
- long类型数据，拆分成高四字节和低四字节，高四字节存储在数组前部分，第四字节存储在数组后部分
- string类型数据，数值存储在str2id字典中，key拆分成高四字节和第四字节，分别存储在数组前部和后部

state字段：通过CIDxMgr查表得知不存在高位下标，低位下标[4],值为1

id字段： 通过CIDxMgr查表得知高位下标[1]和低位下标[7]，值为 (0 << 32) \| (1) = 1

str_cmd字段: 通过CIDxMgr查表得知高位下标[2]和低位下标[8],计算出 键 = 313645，查询str2id字典得值 'addmail,201900000002,201900000002,390,0,39501,1'


### 数组大小的计算

MyDataTable.m_datasize定义一张表在内存中数组的大小

1、 通过gm_cmd.the_enum_end，将数组初始大小赋值给m_datasize

```csharp
// DBInitor.cs
Cache.Instance().InitDatasize((int)TABLE.gm_cmd, (int)gm_cmd.the_enum_end);

// Cache.cs
public void InitDatasize(int table, int datasize)
{
    m_vecDataCache[table].m_datasize = datasize;
    m_vecDataCache[table].m_datamaxfiled = datasize;
}
```
2、 对id、TEXT、MEDIUMTEXT、BIGINT字段自增m_datasize大小

```csharp
// MysqlHelper.cs

public bool AlterTable(DBTableInfo din)
{
    try
    {
        string strdb = ConfigFile.Instance().GetStrValue("db_database");
        string strsql = string.Format("use {0}", strdb);
        new MySqlCommand(strsql, _mySqlConnection).ExecuteNonQuery();

        int table = CIDxMgr.s_Instance().GetTableIdx(din.name);
        CIDxMgr.s_Instance().RegistInt64(table, 1); // id
        foreach (DBTableColumnsInfo dci in din.listcln)
        {
            int fldidx = CIDxMgr.s_Instance().GetFieldIdx(table, dci.name);
            CIDxMgr.s_Instance().RegisterFieldIsDb(table,fldidx , true);
            if (dci.type.ToLower().Contains("char") || dci.type.ToLower().Contains("text"))
            {
                CIDxMgr.s_Instance().RegisterFieldIsString(table, fldidx, true);
                if (CIDxMgr.s_Instance().HighIndexOfInt64(table, fldidx) == 0)
                {
                    CIDxMgr.s_Instance().RegistInt64(table, fldidx);
                }
            }
            else if(dci.type.ToLower().Contains("bigint"))
            {
                CIDxMgr.s_Instance().RegistInt64(table, fldidx);
            }
            
        }
        
    ...
```

```csharp
// CIDxMgr.cs
    
public void  RegistInt64(int table,int index)
{
    MyDataTable mdt = Cache.Instance().m_vecDataCache[table];
    m_vecInt64[table,index]=(int)mdt.m_datasize;
    mdt.m_datasize++;
}
```
3、 对内存字段自增m_datasize大小

```csharp
// DBInitor.cs InitInt64Size()

if (!CIDxMgr.s_Instance().IsDbField(dicNameIdx[type.Name], nIdx))
{
    if (0 == CIDxMgr.s_Instance().HighIndexOfInt64(dicNameIdx[type.Name], nIdx))
    {
        CIDxMgr.s_Instance().RegistInt64(dicNameIdx[type.Name], nIdx);
    }
}
```

### 数据的加载

```csharp
// MModDB\Module.cs

public static bool CreateModule()
{
    // show tables -> show FIELDS 存储数据库中表的字段信息
    Cache.Instance().QueryClmInfo();
    /* 初始化表的集合大小
       
       
    */
    DBInitor.InitPos();
    // 对应上节第一步,定义数组得基础大小
    DBInitor.InitDataSize();

    // 为id、TEXT、MEDIUMTEXT、BIGINT字段申请额外数组空间;对比数据库中表的字段信息，对表结构进行增删改。
    if (!CreateAllDBTable.CreateAllDBTbale())
    {
        return false;
    }
    // 对应上节第三步,为内存字段申请额外数据空间
    DBInitor.InitInt64Size();
    // 注册私有表
    RegistPrivateTable();
    // 将数据库数据加载进Cache
    bool b = CreateAllDBTable.LoadDBData();

    if (!b)
    {
        return false;
    }


    ...
}
```

```csharp
// CreateAllDBTable.LoadDBData()

while (true)
{
    string strcon = string.Format("id>{0} order by id asc limit 10000", startid);
    List<ID> lid = Module.Instance().LoadOneTableData(tbl, strcon);
    if (lid == null)
    {
        log.Error(Uti.PacketString("ERROR:{0}表 加载失败程序退出", tbl));
        return false;
    }
    if (lid.Count == 0)
        break;

    tableRowCnt += lid.Count;
    startid = lid.Last();
}
```
对每个表，程序每次加载10000条数据

```csharp
public List<long> LoadOneTable(string table, int tableindex, string condition, MysqlHelper mh = null)
{
    ...
    var ret = mh.ExecuteQuery_New(string.Format("select * from {0} where {1}", table, condition),
        reader =>

        {
            MyDataTable rmap = m_vecDataCache[tableindex];
            while (reader.Read())
            {
                // 初始化内存数组
                int[] grd = new int[rmap.m_datasize];
                for (int i = 0; i < reader.FieldCount; ++i)
                {
                    ...
                    // 解析字段值,写入MyDataTable
                    if (obj is String)
                    {
                        CGrid.SetStr(grd,tableindex, id,fidx, (string)obj, false);
                    }
                    else if (obj is DBNull)
                    {
                        CGrid.SetStr(grd,tableindex, id, fidx, "", false);
                    }
                    else
                    {
                        ...
                        CGrid.Set(grd,tableindex, id, fidx, v, false);
                    }
                }
                // 向表中声明一行新数据
                Int64 x = m_vecDataCache[tableindex].Insert(tableindex, id, false);
                if (x != 0)
                {
                    // 声明成功，保存此行数据
                    m_vecDataCache[tableindex].m_dicDataGrid[id] = grd;
                }
            }
        });
        ...
}
```

对于 DBInitor.InitPos(); 初始化表的集合大小。这里的集合，指的是表里的一个区间内的id从属于一个对象：
> 例如定义了pet表的集合大小为1000(IDRegionPet.OFFSET)，
> n(n<=1000)个宠物从属于一个玩家对象(user).
> 已知一个user.id,定义[user.id * 1000, user.id * 1000 + 999]是pet表的可用id区间

这是对查询的优化，也是限制。查询时，只能查询出全部或一个集合大小的主键集合
>对于m_mod.GetIdSetByAreaP0((int)TABLE.pet, 0, 12345)是不会返回正确结果的

### 数据的使用

Helper\Data\ 定义映射对象GM_Cmd

##### 查找和修改
```csharp
protected long Get(gm_cmd eField) { return m_mod.GetData((int)TABLE.gm_cmd, m_id, (int)eField); }
protected void Set(gm_cmd eField, long lValue, bool bNeedSync = false) { m_mod.SetData((int)TABLE.gm_cmd, m_id, (int)eField, lValue); }

protected string GetStr(gm_cmd eField) { return m_mod.GetStr((int)TABLE.gm_cmd, m_id, (int)eField); }
protected void SetStr(gm_cmd eField, string strValue, bool bNeedSync = false) { m_mod.SetStr((int)TABLE.gm_cmd, m_id, (int)eField, strValue); }
```

字段的get、set封装了对Cache的调用

##### 删除数据

```csharp
mod.Delete(TABLENAME.gm_cmd, id);
```

##### 添加数据

```csharp
m_mod.Insert(TABLENAME.gm_cmd, id);
```

### 数据的卸载

私有数据表定义为玩家离线后，不会被查询和修改的数据。定义玩家离线4小时为私有表的卸载时间。玩家登陆时需要先加载私有表才允许进入游戏。私有表在RegistPrivateTable()方法中定义

卸载:
```csharp
// MsgOnMinuteTimer.cs

UserDataManager.UnloadUserData(idplayer, m_mod);
```

加载：
```csharp
// MsgLoadUser.cs

if(UserDataManager.m_unloadedPlayer.Contains(idActor))
{
    if (!UserDataManager.m_lodingPlayer.Contains(idActor))
    {
        DatabaseEventUserDataGet dbevt = new DatabaseEventUserDataGet(idActor, m_mod,idLoad,nFCM,nGameTime,strIp,nPort,src,(uint)p.userredisver_unload);
        ThreadRedisClient.Instance().AddRedisEvent(dbevt);
        UserDataManager.m_lodingPlayer.Add(idActor);
    }
    return;
}
```

